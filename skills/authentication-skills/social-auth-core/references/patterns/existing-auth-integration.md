# Existing Auth Integration

**When to read:** The Discovery Report detects an existing authentication system (JWT, session-based, or other) already in place.
**What problem it solves:** Prevents breaking existing auth flows while adding social login.
**When to skip:** Greenfield auth with no existing login system.
**Prerequisites:** Read `references/governance/account-linking.md` and `references/patterns/token-management.md` if tokens are used.

## Compatibility Mode Decision Tree

1. Does the existing system use JWTs or server sessions?
   - JWTs -> Social login must mint the same access/refresh tokens.
   - Server sessions -> Social login must create the same session/cookie format.

2. How are tokens delivered today?
   - Authorization header only -> Preserve header flow and add cookie support only if approved.
   - HTTP-only cookies -> Preserve cookies; do not introduce URL tokens.
   - Both -> Preserve both to avoid breaking clients.

3. Is there an existing user model with email-based accounts?
   - Yes -> Link social identities to existing users when emails match and provider email is verified.
   - No -> Define a linking policy before implementation.

4. What is the session store?
   - Database/Redis -> Reuse existing store.
   - In-memory -> Flag risk for scaling and confirm acceptable.

## Dual Token Extraction Pattern

Use this pattern when you must accept both `Authorization: Bearer <token>` and a cookie-based access token.

Pseudocode:
```text
function extractAccessToken(req):
  if Authorization header exists and starts with "Bearer ":
    return header token
  if req.cookies.accessToken exists:
    return cookie token
  return null
```

NestJS example:
```ts
import { ExtractJwt } from 'passport-jwt';

const fromAuthHeader = ExtractJwt.fromAuthHeaderAsBearerToken();

const fromCookie = (req: any) => {
  if (req?.cookies?.accessToken) return req.cookies.accessToken;
  return null;
};

const extractToken = (req: any) => {
  return fromAuthHeader(req) || fromCookie(req);
};
```

Express example:
```ts
function extractToken(req) {
  const auth = req.headers.authorization || '';
  if (auth.startsWith('Bearer ')) return auth.slice(7);
  if (req.cookies?.accessToken) return req.cookies.accessToken;
  return null;
}
```

## Non-Breaking Integration Rules

- Preserve existing env var names; use internal mapping only.
- Do not change existing session/token format unless explicitly approved.
- Social login should produce the same session/token type as existing auth.
- Account linking: if social login email matches an existing account, link only if provider email is verified. Do not create a duplicate.

## Decisions to Capture (Required)

Add a "Compatibility Decisions" section to the output and document:
- What was preserved from the existing auth system.
- What changed and why.
- How token delivery was handled (header, cookies, or both).
- Account linking policy and verified email requirement.
