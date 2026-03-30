## `references/adapters/express-patterns.md`

````md
# Express Adapter Patterns

## Purpose

Provides Express-specific integration patterns for social authentication.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file defines Express-specific routing, session, and callback implementation guidance only.

## When to Use

Use this file when the backend is built with Express and social authentication must be implemented using Express routes and middleware.

## Route Placement

Typical routes:

- login start → `/auth/:provider`
- callback → `/auth/:provider/callback`

Keep these routes on the backend-controlled domain.

## Login Start Pattern

In the start route:

- generate `state`, `nonce`, and `code_verifier` server-side
- derive a PKCE challenge using `S256`
- persist pre-auth state in a server-controlled session or encrypted store
- bind pre-auth state to the selected provider
- redirect to the provider authorization URL

## Callback Pattern

In the callback route:

- read `code`, `state`, and provider error parameters from the query
- load pre-auth state from the session or server store
- validate:
  - provider match
  - `state`
  - expiration / TTL
  - one-time-use status
  - `nonce` where OIDC applies
- exchange authorization code for tokens
- validate tokens and profile data
- link or create account
- rotate session after successful login
- redirect to an approved destination

## Session and Cookie Guidance

- use HTTPOnly and Secure cookies
- default to `SameSite=Lax` unless cross-site requirements explicitly require `None`
- do not expose tokens to browser storage
- keep only a session identifier in cookies where possible
- session rotation after successful login is required

## Express-Specific Considerations

- confirm session middleware is available before auth routes
- ensure callback routes are not affected by body-parsing assumptions that do not apply to GET callbacks
- if behind a proxy, configure trust proxy correctly so secure cookies behave as expected
- confirm route ordering does not accidentally shadow provider routes

## Minimal Implementation Skeleton

```ts
import crypto from "crypto";
import { Router } from "express";

const router = Router();

router.get("/auth/:provider", async (req, res) => {
  const provider = req.params.provider;
  const state = crypto.randomUUID();
  const nonce = crypto.randomUUID();
  const verifier = crypto.randomBytes(32).toString("base64url");
  const challenge = crypto
    .createHash("sha256")
    .update(verifier)
    .digest("base64url");

  req.session.oauth = {
    provider,
    state,
    nonce,
    verifier,
  };

  const url = buildAuthorizeUrl({
    provider,
    state,
    nonce,
    codeChallenge: challenge,
  });

  res.redirect(url);
});

router.get("/auth/:provider/callback", async (req, res) => {
  const provider = req.params.provider;
  const { code, state } = req.query;

  const preAuth = req.session.oauth;
  assertValidPreAuth(preAuth, provider, state);

  const tokens = await exchangeCodeForTokens({
    provider,
    code,
    codeVerifier: preAuth.verifier,
  });

  const profile = await validateAndFetchProfile(tokens, preAuth.nonce);
  const user = await linkOrCreateAccount(profile, provider);

  await rotateSession(req, user);

  res.redirect("/");
});
```
````
