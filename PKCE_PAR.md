# PAR + PKCE Behavior Summary

This fork extends the Generic OAuth2 Capacitor plugin to support **Pushed Authorization Requests (PAR)** with **PKCE** on all platforms (Web, Android, iOS) while keeping the existing API backward compatible.

## New configuration

- `OAuth2AuthenticateBaseOptions.parEndpoint?: string`
  - URL of the provider's PAR endpoint (`pushed_authorization_request_endpoint`).
  - Can be set globally, or per platform using `web.parEndpoint`, `android.parEndpoint`, or `ios.parEndpoint`.

Behavior:

- If `parEndpoint` is **unset** for a platform:
  - Existing behavior is preserved (no PAR).
  - PKCE, authorization, and token flows work as before.
- If `parEndpoint` is **set** for a platform:
  - The plugin performs a PAR call before the authorization redirect and uses the returned `request_uri` when starting the authorization flow.

## Single PKCE verifier per auth attempt

Across all platforms the plugin now guarantees that:

- When `pkceEnabled` is `true`, a single **PKCE code verifier** is generated for the auth attempt.
- The **code challenge** used in the PAR request is always derived from this same verifier.
- The same verifier is then used at the **token exchange** step.

Platform details:

- **Web**
  - `WebUtils.buildWebOptions`:
    - Generates `pkceCodeVerifier` once (or reuses it from `sessionStorage`).
    - Computes `pkceCodeChallenge` and `pkceCodeChallengeMethod`.
  - `WebUtils.performPar`:
    - Sends `client_id`, `response_type`, `redirect_uri`, `scope`, `state`, and PKCE (`code_challenge`, `code_challenge_method`) plus `additionalParameters` to `parEndpoint` via `POST` (`application/x-www-form-urlencoded`).
    - Reads `request_uri` from the JSON response and stores it.
  - `WebUtils.getAuthorizationUrl`:
    - If `request_uri` is present, builds an authorization URL with `client_id` and `request_uri` instead of repeating all parameters.
  - `getTokenEndpointData`:
    - Uses the same `pkceCodeVerifier` as the `code_verifier` in the token request.

- **Android (AppAuth)**
  - `OAuth2Options`:
    - Holds `parEndpoint`, `parRequestUri`, `pkceEnabled`, and `pkceCodeVerifier` (generated once per auth attempt).
  - `ParRequestAsyncTask`:
    - Sends `client_id`, `response_type`, `redirect_uri`, `scope`, `state`, PKCE (`code_challenge` derived from `pkceCodeVerifier` with S256, or plain as fallback), and `additionalParameters` to `parEndpoint` via `POST` (`application/x-www-form-urlencoded`).
    - Stores the returned `request_uri` on `OAuth2Options`.
  - `GenericOAuth2Plugin.startAuthorization`:
    - Builds the usual `AuthorizationRequest.Builder` (state, scope, etc).
    - Always passes the same `pkceCodeVerifier` to `setCodeVerifier` when PKCE is enabled.
    - Includes `request_uri` in the additional parameters when PAR is used.

- **iOS (OAuthSwift)**
  - `GenericOAuth2Plugin`:
    - Generates `requestState` and, when `pkceEnabled`, generates a single `pkceCodeVerifier` and `pkceCodeChallenge` per auth attempt.
  - If `parEndpoint` is configured:
    - Builds a PAR request with `client_id`, `response_type`, `redirect_uri`, `scope`, `state`, PKCE (`code_challenge`, `code_challenge_method=S256` when enabled), plus `additionalParameters`.
    - Sends `POST` (`application/x-www-form-urlencoded`) to `parEndpoint` and extracts `request_uri` from the JSON response.
  - Calls `oauthSwift.authorize`:
    - Adds `request_uri` to the `parameters` dictionary when PAR is used.
    - Passes the same PKCE `codeChallenge`/`codeVerifier` pair to `authorize`, which is reused by OAuthSwift for the token request.

## Error handling

If PAR is enabled and fails, the plugin fails fast before opening a browser / external UI:

- **Web**
  - `WebUtils.performPar` rejects with an `Error` whose message starts with `PAR_FAILED:`:
    - `PAR_FAILED: missing request_uri in response`
    - `PAR_FAILED: invalid JSON response`
    - `PAR_FAILED: HTTP <status> [error - error_description]`
    - `PAR_FAILED: network error`

- **Android**
  - `ParRequestAsyncTask` rejects the call with:
    - Error code: `ERR_PAR_FAILED`
    - Message: `PAR_FAILED: ...` (HTTP status, JSON `error` / `error_description`, or network/format issues).

- **iOS**
  - PAR failures reject the CAP plugin call with:
    - Error code: `ERR_PAR_FAILED`
    - Message: `PAR_FAILED: ...` (HTTP status, JSON parsing issues, or network error).

Logging:

- The existing `logsEnabled` flag continues to control logging behavior.
- When `logsEnabled` is `true`, PAR requests and responses are logged (without printing secrets like client secret, which is not used in this flow).

## Backward compatibility

- When `parEndpoint` is **not configured**:
  - Web, Android, and iOS behave exactly as in the upstream plugin:
    - No PAR call is made.
    - PKCE (when enabled) works unchanged.
- When `parEndpoint` **is configured**:
  - The only behavioral difference is:
    - The plugin performs a PAR call before redirecting to the authorization endpoint.
    - The authorization redirect uses the resulting `request_uri` alongside the existing PKCE and state handling.

## Example configuration (Authelia + PAR + PKCE)

```ts
const options: OAuth2AuthenticateOptions = {
  appId: 'my-client-id',
  authorizationBaseUrl: 'https://auth.example.com/oauth2/authorize',
  accessTokenEndpoint: 'https://auth.example.com/oauth2/token',
  parEndpoint: 'https://auth.example.com/oauth2/par',
  redirectUrl: 'myapp://callback',
  responseType: 'code',
  scope: 'openid profile email offline_access',
  pkceEnabled: true,
  logsEnabled: true,

  // Optional per-platform overrides
  web: {
    parEndpoint: 'https://auth.example.com/oauth2/par',
  },
  android: {
    parEndpoint: 'https://auth.example.com/oauth2/par',
  },
  ios: {
    parEndpoint: 'https://auth.example.com/oauth2/par',
  },

  additionalParameters: {
    // any provider-specific params, e.g. audience
  },
};
```

This setup will use PAR + PKCE correctly with providers like Authelia that require both PAR and PKCE, while still allowing non-PAR providers to work by simply omitting `parEndpoint`.

