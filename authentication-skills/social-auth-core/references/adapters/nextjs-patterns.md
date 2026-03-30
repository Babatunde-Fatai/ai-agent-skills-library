## `references/adapters/nextjs-patterns.md`

````md
# Next.js Adapter Patterns

## Purpose

Provides Next.js-specific integration patterns for social authentication.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file defines Next.js-specific route handler, API route, cookie, and callback implementation guidance only.

## When to Use

Use this file when social authentication is being implemented in a Next.js application.

## Supported Placement Models

This adapter covers:

- App Router route handlers
- Pages Router API routes

Choose the one that matches the existing project structure.

## App Router Pattern

Typical routes:

- `app/api/auth/[provider]/route.ts`
- `app/api/auth/[provider]/callback/route.ts`

### Login Start Pattern

In the route handler:

- generate `state`, `nonce`, and `code_verifier` server-side
- derive a PKCE challenge using `S256`
- store pre-auth state in a backend-controlled store
- redirect to the provider authorization URL

### Callback Pattern

In the callback route handler:

- read `code`, `state`, and provider errors from the request URL
- load pre-auth state for the provider
- validate:
  - provider match
  - `state`
  - TTL / expiration
  - one-time-use status
  - `nonce` where OIDC applies
- exchange authorization code for tokens
- validate tokens and profile data
- link or create account
- rotate or create session
- set secure session cookie
- redirect safely

### Minimal App Router Skeleton

```ts
import crypto from "crypto";
import { NextResponse } from "next/server";

export async function GET(
  req: Request,
  { params }: { params: { provider: string } },
) {
  const state = crypto.randomUUID();
  const nonce = crypto.randomUUID();
  const verifier = crypto.randomBytes(32).toString("base64url");
  const challenge = crypto
    .createHash("sha256")
    .update(verifier)
    .digest("base64url");

  await savePreAuthState({
    provider: params.provider,
    state,
    nonce,
    verifier,
  });

  const url = buildAuthorizeUrl({
    provider: params.provider,
    state,
    nonce,
    codeChallenge: challenge,
  });

  return NextResponse.redirect(url);
}
```
````
