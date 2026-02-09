# Session Handling

**When to read:** When creating or rotating sessions or pre-auth cookies.
**What problem it solves:** Secure session and cookie defaults.
**When to skip:** If session handling is fully managed and unchanged.
**Prerequisites:** Read `references/patterns/oauth-flow-core.md`.

# Pre-Auth "Transaction" Cookie

When a user clicks "Login with Google", you cannot attach state to their main session (it doesn't exist yet). You must use an ephemeral cookie.
Pre-auth cookies are a valid form of server-side storage when the payload is encrypted/signed and bound to a short TTL.

## Recommended Flow (adapt to current task)
1. **Generate** `state`, `code_verifier`, and `nonce`.
2. **Serialize** this data (JSON).
3. **Encrypt/Sign** the data (Required to prevent tampering).
4. **Set Cookie:**
   - Name: `oauth_state` (or similar prefix)
   - Value: `encrypted_json_blob`
   - Path: `/api/auth` (Restrict scope if possible)
   - MaxAge: 10 minutes (Short TTL)
   - HttpOnly: true
   - Secure: true
   - SameSite: Lax

## Verification (Callback)
1. Read `oauth_state` cookie.
2. Decrypt and parse.
3. Compare `cookie.state` vs `query.state`.
4. If valid, use `cookie.code_verifier` for token exchange.
5. **CRITICAL:** Delete `oauth_state` cookie immediately after use.

## Session Security Defaults

- Rotate session id after successful login.
- Use HTTPOnly, Secure cookies for session identifiers.
- Use `SameSite=Lax` unless explicit cross-site flows require `None`.
- Set short session TTLs and explicit idle timeouts.

## CSRF Protection

- Enforce CSRF protections for any state-changing endpoints.
- Bind pre-auth state to a server-side session to prevent login CSRF.

## Session Storage

- Prefer server-side session stores (Redis, database, or framework session store).
- Avoid storing user identity or tokens in client-readable cookies.

## Logout

- Clear server session and rotate identifiers.
- Revoke refresh tokens if supported.


---
