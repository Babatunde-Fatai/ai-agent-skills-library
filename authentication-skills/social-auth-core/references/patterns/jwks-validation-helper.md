# JWKS Validation Helper

## Purpose

Provides a lightweight helper pattern for validating OIDC ID token signatures using a provider JWKS endpoint when a heavier library is not being used.

## Relationship to Core Rules

Global OIDC validation requirements are defined by:

- `oauth-flow-core.md`
- provider docs
- `../../../core/SECURITY_INVARIANTS.md`

This file is an implementation helper, not a substitute for full validation requirements.

## When to Use

Use this file when:

- OIDC applies
- the project does not already use a mature JWT / OIDC library
- a lightweight helper is acceptable for the runtime

If the project already uses a vetted library such as `jose`, prefer the existing library unless there is a strong reason not to.

## Minimal Node.js Example

```ts
import crypto from 'crypto';
import https from 'https';

type Json = Record<string, any>;

const jwksCache = {
  keys: null as null | Json,
  fetchedAt: 0,
  ttlMs: 60 * 60 * 1000,
};

function fetchJson(url: string): Promise<Json> {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      let data = '';
      res.on('data', (chunk) => (data += chunk));
      res.on('end', () => {
        try {
          resolve(JSON.parse(data));
        } catch (err) {
          reject(err);
        }
      });
    }).on('error', reject);
  });
}

async function getJwks(jwksUri: string): Promise<Json> {
  const now = Date.now();
  if (jwksCache.keys && now - jwksCache.fetchedAt < jwksCache.ttlMs) {
    return jwksCache.keys;
  }

  const jwks = await fetchJson(jwksUri);
  jwksCache.keys = jwks;
  jwksCache.fetchedAt = now;
  return jwks;
}

function decodeBase64Url(input: string): Buffer {
  const normalized = input.replace(/-/g, '+').replace(/_/g, '/');
  const padded = normalized.padEnd(Math.ceil(normalized.length / 4) * 4, '=');
  return Buffer.from(padded, 'base64');
}

function parseJwt(token: string) {
  const [header, payload, signature] = token.split('.');
  return {
    header: JSON.parse(decodeBase64Url(header).toString('utf8')),
    payload: JSON.parse(decodeBase64Url(payload).toString('utf8')),
    signature,
    signingInput: token.split('.').slice(0, 2).join('.'),
  };
}
```

## Required Validation Beyond Signature

Do not stop at signature validation. Also validate:

- issuer
- audience
- expiry
- nonce
- authorized party where relevant
- provider-specific claim expectations

## Caching Guidance

- cache JWKS for a bounded TTL
- allow key rotation by refreshing on cache expiry
- handle unknown `kid` safely by re-fetching once before failing

## When Not to Use This Pattern

Do not use this helper as-is when:

- the runtime already has a proven OIDC library
- the provider has unusual signing requirements not covered here
- the team cannot safely maintain custom crypto-sensitive code

## Maintenance Rule

- lightweight helper examples belong here
- provider-specific discovery endpoints and claims belong in provider docs
