# OpenAI Codex OAuth

Detailed walkthrough of the OpenAI Codex (ChatGPT Plus/Pro subscription) OAuth flow, implemented mainly in `packages/ai/src/utils/oauth/openai-codex.ts`.

## Where it plugs in

The provider declares OAuth lazily so the Node-only flow code isn't bundled into browser builds:

```ts
// providers/openai-codex.ts
createProvider({
  id: "openai-codex",
  baseUrl: "https://chatgpt.com/backend-api",
  auth: { oauth: lazyOAuth({ name: "OpenAI (ChatGPT Plus/Pro)", load: loadOpenAICodexOAuth }) },
  api: openAICodexResponsesApi(),
});
```

`lazyOAuth` (in `auth/helpers.ts`) defers `import()` of the implementation until the first `login`/`refresh`/`toAuth` call. Two shapes of the implementation are exported:
- `openaiCodexOAuth` (the `OAuthAuth` interface: `login`/`refresh`/`toAuth`) - the current path used by `Models`.
- `openaiCodexOAuthProvider` (the older `OAuthProviderInterface`: `login`/`refreshToken`/`getApiKey`).

## Constants

```
CLIENT_ID   = "app_EMoamEEZ73f0CkXaXp7hrann"
AUTH_BASE   = https://auth.openai.com
AUTHORIZE   = /oauth/authorize
TOKEN       = /oauth/token
REDIRECT    = http://localhost:1455/auth/callback
SCOPE       = "openid profile email offline_access"
JWT claim   = "https://api.openai.com/auth"
```

It's a public OAuth client using **PKCE** (no client secret). There are two login methods the user picks between: browser and device-code.

## Method 1: Browser login (`loginOpenAICodex`)

1. **PKCE + state**: `generatePKCE()` (in `pkce.ts`) makes a random 32-byte `verifier`, base64url-encodes it, and computes `challenge = base64url(SHA-256(verifier))` via Web Crypto. `createState()` makes a random 16-byte hex `state` (CSRF guard).

2. **Authorize URL** (`createAuthorizationFlow`): builds the `/oauth/authorize` URL with `response_type=code`, `client_id`, `redirect_uri`, `scope`, `code_challenge` + `code_challenge_method=S256`, `state`, plus Codex-specific params `id_token_add_organizations=true`, `codex_cli_simplified_flow=true`, and `originator=pi`.

3. **Local callback server** (`startLocalOAuthServer`): starts an HTTP server on `127.0.0.1:1455` (host overridable via `PI_OAUTH_CALLBACK_HOST`). It only accepts `/auth/callback`, **verifies `state` matches** (400 on mismatch), extracts `code`, and serves a success/error HTML page. If port 1455 is already bound, it degrades gracefully to a null server so the manual-paste path can still complete.

4. **Getting the code** - three racing/fallback sources:
   - The browser redirect hitting the local server (`waitForCode()`).
   - A **manual paste** prompt (`onManualCodeInput`) that races the server; whichever resolves first wins. `parseAuthorizationInput` accepts a bare code, a `code#state` string, a query string, or a full redirect URL, and re-checks `state`.
   - A final `onPrompt` fallback if neither produced a code.

5. **Token exchange** (`exchangeAuthorizationCode`): POSTs `grant_type=authorization_code` with `client_id`, `code`, `code_verifier` (the PKCE proof), and `redirect_uri` to `/oauth/token`.

## Method 2: Device-code login (`loginOpenAICodexDeviceCode`, headless)

1. `startOpenAICodexDeviceAuth`: POSTs `{ client_id }` to `/api/accounts/deviceauth/usercode`, receiving `device_auth_id`, `user_code`, and a poll `interval`. A 404 means device login isn't enabled server-side.
2. The `user_code` and verification URI (`https://auth.openai.com/codex/device`) are surfaced via `onDeviceCode`.
3. `pollOpenAICodexDeviceAuth` uses the shared `pollOAuthDeviceCodeFlow` (in `device-code.ts`), which implements RFC 8628 polling: honors `interval`, handles `authorization_pending`/403/404 as "pending", `slow_down` by increasing the interval (server-provided value preferred to survive WSL/VM clock drift), and a 15-minute deadline. On success the server returns an `authorization_code` **and** its `code_verifier`.
4. Those are exchanged the same way, but with `redirect_uri = https://auth.openai.com/deviceauth/callback`.

## Token response and credentials

`readTokenResponse` validates `access_token`, `refresh_token`, and numeric `expires_in`, then produces:

```ts
{ access, refresh, expires: Date.now() + expires_in * 1000 }
```

`credentialsFromToken` then **decodes the JWT access token** (`decodeJwt`, base64 of the middle segment) and pulls `chatgpt_account_id` from the `https://api.openai.com/auth` claim. The stored credential is:

```ts
{ type: "oauth", access, refresh, expires, accountId }
```

If no `accountId` can be extracted, login fails hard. Credentials are persisted by the app-owned `CredentialStore` (keyed by provider id, one per provider).

## Refresh and expiry (the locked path)

Refresh is a `refresh_token` grant to `/oauth/token` with `client_id` (`refreshAccessToken` -> `refreshOpenAICodexToken`). It is *not* called directly at request time. Instead, `Models.getAuth()` -> `resolveProviderAuth` -> `resolveStoredOAuth` (in `auth/resolve.ts`) owns expiry handling with **double-checked locking**:

```ts
if (Date.now() >= credential.expires) {
  post = await credentials.modify(providerId, async (current) => {
    if (current?.type !== "oauth") return undefined;      // logged out meanwhile
    if (Date.now() < current.expires) return undefined;    // someone else already refreshed
    return await oauth.refresh(current);                   // single global refresh
  });
  ...
}
return { auth: await oauth.toAuth(credential), source: "OAuth" };
```

- Valid tokens cost zero locks.
- `CredentialStore.modify` is a serialized read-modify-write (with a cross-process file lock in coding-agent's `AuthStorage`), so concurrent requests can't double-refresh a rotated token.
- A failed refresh throws `ModelsError("oauth", ...)` and **preserves** the stored credential (re-login fixes it); there is no silent env fallback.

`toAuth` is deliberately trivial and side-effect-free: `{ apiKey: credential.access }`. So the resolved "API key" for a Codex request is simply the current access token.

## Applying auth at request time

`Models.applyAuth` merges the resolved `apiKey` into the stream options. Then in the Codex API implementation (`api/openai-codex-responses.ts`), for each request:

- `extractAccountId(apiKey)` re-decodes the JWT to get `chatgpt_account_id` (it re-derives it from the token rather than trusting stored state).
- Headers set (`buildBaseCodexHeaders`):
  - `Authorization: Bearer <access token>`
  - `chatgpt-account-id: <accountId>`
  - `originator: pi`
  - a `pi (<platform> <release>; <arch>)` User-Agent
- SSE requests add `OpenAI-Beta: responses=experimental` and `accept: text/event-stream`; the WebSocket transport builds its own variant.

## End-to-end summary

```
/login (openai-codex)
  -> choose browser | device_code
  browser:  PKCE(verifier,challenge) + state
            open /oauth/authorize (S256, originator=pi, codex flow flags)
            localhost:1455 callback  (state-checked)  OR  manual paste (raced)
            POST /oauth/token  grant=authorization_code + code_verifier
  device:   POST /deviceauth/usercode -> user_code
            poll /deviceauth/token (RFC 8628) -> authorization_code + verifier
            POST /oauth/token  grant=authorization_code (deviceauth redirect)
  -> {access, refresh, expires}; decode JWT -> accountId
  -> store {type:"oauth", access, refresh, expires, accountId}

per request:
  Models.getAuth -> if expired: locked single refresh (refresh_token grant)
                 -> toAuth => apiKey = access token
  codex api: headers Authorization: Bearer, chatgpt-account-id (from JWT), originator: pi
```

The design cleanly separates the interactive login/refresh flow (Node-only, lazily loaded) from the stateless per-request auth derivation, and centralizes token-rotation safety in the credential store's locked `modify`.
