# SameSite Decision Tree

**When to read:** Session or token cookies are being set and the deployment involves more than one domain.
**What problem it solves:** Selects safe SameSite policy based on domain topology and OAuth redirects.
**When to skip:** No cookies are used.
**Prerequisites:** Read `references/patterns/session-handling.md`.

## Decision Tree

1. Frontend and backend on the same domain (e.g., app.example.com)
   - Use `SameSite=Lax` (or `Strict` if no cross-site needs).

2. Frontend and backend on different subdomains of the same root (e.g., app.example.com and api.example.com)
   - `SameSite=Lax` works if cookies are set on the root domain (e.g., `.example.com`).

3. Frontend and backend on completely different domains (e.g., myapp.com and api.myapp.dev)
   - Use `SameSite=None` + `Secure`.
   - Document CORS implications and ensure `credentials: include` is enabled.

4. OAuth callback redirects (provider -> backend -> frontend)
   - Callback is a top-level cross-site navigation.
   - `SameSite=Lax` cookies are sent on top-level navigations, but NOT on subresource requests.

## Security Tradeoffs

- `SameSite=None` increases CSRF surface.
- If `SameSite=None` is required, ensure CSRF protections are in place.

## Testing Note

Safari and some mobile browsers have quirks with `SameSite=None`. Test across browsers.
