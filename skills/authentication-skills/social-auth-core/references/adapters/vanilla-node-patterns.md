# Vanilla Node Patterns

**When to read:** When implementing social auth in vanilla Node.
**What problem it solves:** Server-only flow without a framework.
**When to skip:** If a framework adapter applies.
**Prerequisites:** Read `references/patterns/oauth-flow-core.md`.

## Minimal Flow Outline

```ts
import http from 'http';
import crypto from 'crypto';

const server = http.createServer(async (req, res) => {
  if (req.url?.startsWith('/auth/google')) {
    const state = crypto.randomUUID();
    const nonce = crypto.randomUUID();
    const verifier = crypto.randomBytes(32).toString('base64url');
    const challenge = crypto.createHash('sha256').update(verifier).digest('base64url');

    await savePreAuthState({ provider: 'google', state, nonce, verifier });

    const url = buildAuthorizeUrl({
      provider: 'google',
      state,
      nonce,
      codeChallenge: challenge,
    });

    res.writeHead(302, { Location: url });
    res.end();
    return;
  }

  if (req.url?.startsWith('/auth/google/callback')) {
    const { code, state } = parseQuery(req.url);
    const preAuth = await loadPreAuthState('google');
    assertStateMatches(preAuth, state);

    const tokens = await exchangeCodeForTokens({
      provider: 'google',
      code,
      codeVerifier: preAuth.verifier,
    });

    const profile = await validateAndFetchProfile(tokens);
    await rotateAndCreateSession(profile, tokens, res);

    res.writeHead(302, { Location: '/' });
    res.end();
    return;
  }

  res.writeHead(404);
  res.end();
});
```

## Session Handling

- Use a server-side session store.
- Send only a session identifier in cookies.
- Set HTTPOnly, Secure, SameSite cookies.
