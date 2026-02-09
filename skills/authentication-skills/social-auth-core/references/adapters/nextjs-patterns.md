# Next.js Adapter Patterns

**When to read:** When implementing social auth in Next.js.
**What problem it solves:** Next.js route handler patterns for OAuth.
**When to skip:** If you are not using Next.js.
**Prerequisites:** Read `references/patterns/oauth-flow-core.md`.

## App Router (Route Handlers)

Routes:
- `app/api/auth/[provider]/route.ts`
- `app/api/auth/[provider]/callback/route.ts`

Minimal example:

```ts
import crypto from 'crypto';
import { NextResponse } from 'next/server';

export async function GET(req: Request, { params }: { params: { provider: string } }) {
  const state = crypto.randomUUID();
  const nonce = crypto.randomUUID();
  const verifier = crypto.randomBytes(32).toString('base64url');
  const challenge = crypto.createHash('sha256').update(verifier).digest('base64url');

  await savePreAuthState({ provider: params.provider, state, nonce, verifier });

  const url = buildAuthorizeUrl({
    provider: params.provider,
    state,
    nonce,
    codeChallenge: challenge,
  });

  return NextResponse.redirect(url);
}
```

Callback handler:

```ts
export async function GET(req: Request, { params }: { params: { provider: string } }) {
  const { searchParams } = new URL(req.url);
  const code = searchParams.get('code');
  const state = searchParams.get('state');

  const preAuth = await loadPreAuthState(params.provider);
  assertStateMatches(preAuth, state);

  const tokens = await exchangeCodeForTokens({
    provider: params.provider,
    code,
    codeVerifier: preAuth.verifier,
  });

  const profile = await validateAndFetchProfile(tokens);
  const session = await rotateAndCreateSession(profile, tokens);

  const res = NextResponse.redirect(new URL('/', req.url));
  setSessionCookie(res, session);
  return res;
}
```

## Pages Router (API Routes)

Routes:
- `pages/api/auth/[provider].ts`
- `pages/api/auth/[provider]/callback.ts`

Follow the same logic as above using `req` and `res` and a server-side session store.

## Cookie Defaults

- HTTPOnly and Secure cookies only.
- `SameSite=Lax` unless explicitly required.
- Rotate session identifier after login.
