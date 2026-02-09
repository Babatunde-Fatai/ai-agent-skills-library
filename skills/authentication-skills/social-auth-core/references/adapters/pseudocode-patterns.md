# Pseudocode Patterns (Framework-Agnostic)

**When to read:** When your framework is not covered by a specific adapter.
**What problem it solves:** Provides a language-agnostic OAuth/OIDC flow you can adapt.
**When to skip:** If a framework-specific adapter applies.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and `references/patterns/oauth-flow-core.md`.

## Login Start

```text
handle GET /auth/<provider>:
  // Step 1: Generate cryptographic random values
  state = secureRandom(32 bytes)
  nonce = secureRandom(32 bytes)
  verifier = secureRandom(32 bytes)

  // Step 2: Derive PKCE challenge (S256)
  challenge = base64url(sha256(verifier))

  // Step 3: Store pre-auth state server-side
  storePreAuthState(provider, state, nonce, verifier, expiresIn: 10 minutes)

  // Step 4: Build authorization URL
  authUrl = buildAuthUrl(
    provider: provider,
    clientId: ENV.CLIENT_ID,
    redirectUri: ENV.REDIRECT_URI,
    state: state,
    nonce: nonce,
    codeChallenge: challenge,
    codeChallengeMethod: "S256",
    scope: "openid email profile",
    responseType: "code"
  )

  // Step 5: Redirect
  redirectTo(authUrl)
```

## Callback

```text
handle GET /auth/<provider>/callback:
  if error in query: fail with safe error
  if code missing: reject

  preAuth = loadPreAuthState(provider)
  if preAuth.state mismatch: reject
  if preAuth expired or already used: reject

  tokens = exchangeCodeForTokens(code, preAuth.verifier)
  validate ID token claims and signature

  link or create user account
  rotate session id
  persist tokens server-side only
  redirect to app
```

## Session Defaults

- Use a server-side session store or encrypted pre-auth cookies.
- Use HTTPOnly, Secure, SameSite cookies.
- Rotate session on login.

## Language-Specific Notes

- Python: Use `secrets` for randomness. Django sessions via `django.contrib.sessions`. Flask sessions via `flask_session` or server store.
- PHP: Use `random_bytes()`. Laravel session encryption is built-in.
- Ruby: Use `SecureRandom`. Rails encrypted cookies via `cookies.encrypted`.
- Java: Use `SecureRandom`. Spring sessions via `HttpSession` or Redis.
- C#: Use `RandomNumberGenerator`. ASP.NET data protection via `IDataProtector`.
