# Express Adapter Patterns

**When to read:** When implementing social auth in Express.
**What problem it solves:** Express routing patterns for OAuth.
**When to skip:** If you are not using Express.
**Prerequisites:** Read `references/patterns/oauth-flow-core.md`.

## Where to Insert the Flow

- Login start route: `/auth/<provider>`
- Callback route: `/auth/<provider>/callback`

## Minimal Safe Example

```ts
import crypto from 'crypto';
import { Router } from 'express';

const router = Router();

router.get('/auth/google', (req, res) => {
  const state = crypto.randomUUID();
  const nonce = crypto.randomUUID();
  const verifier = crypto.randomBytes(32).toString('base64url');
  const challenge = crypto.createHash('sha256').update(verifier).digest('base64url');

  req.session.oauth = { state, nonce, verifier, provider: 'google' };

  const url = buildAuthorizeUrl({
    provider: 'google',
    state,
    nonce,
    codeChallenge: challenge,
  });

  res.redirect(url);
});

router.get('/auth/google/callback', async (req, res) => {
  const { code, state } = req.query;
  assertStateMatches(req.session.oauth, state);

  const tokens = await exchangeCodeForTokens({
    provider: 'google',
    code,
    codeVerifier: req.session.oauth.verifier,
  });

  const profile = await validateAndFetchProfile(tokens);

  await rotateSession(req);
  await linkOrCreateAccount(profile, tokens);

  res.redirect('/');
});
```

## Session and Cookie Defaults

- Use HTTPOnly, Secure cookies.
- Set `SameSite=Lax` unless cross-site redirects require `None`.
- Rotate session identifiers after login.
