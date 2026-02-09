# JWKS Validation Helper

**When to read:** Validating OIDC ID tokens from providers that publish JWKS endpoints (Google, Microsoft, etc.).
**What problem it solves:** Provides a minimal JWKS fetch/cache helper without adding heavy dependencies.
**When to skip:** If the project already uses `jose`, `jsonwebtoken`, `passport`, or similar.
**Prerequisites:** Read `references/providers/{provider}.md` for discovery and JWKS endpoints.

## Minimal Helper (Node.js)

This helper fetches JWKS from the provider discovery document, caches keys for 1 hour, and verifies
an ID token signature using Node.js `crypto`.

```ts
import crypto from 'crypto';
import https from 'https';

const jwksCache = {
  keys: null as null | any,
  fetchedAt: 0,
  ttlMs: 60 * 60 * 1000,
};

function fetchJson(url: string): Promise<any> {
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

async function getJwks(jwksUri: string) {
  const now = Date.now();
  if (jwksCache.keys && now - jwksCache.fetchedAt < jwksCache.ttlMs) {
    return jwksCache.keys;
  }
  const jwks = await fetchJson(jwksUri);
  jwksCache.keys = jwks;
  jwksCache.fetchedAt = now;
  return jwks;
}

function importRsaKey(jwk: any) {
  return crypto.createPublicKey({
    key: jwk,
    format: 'jwk',
  });
}

function verifyJwtSignature(idToken: string, key: crypto.KeyObject) {
  const [headerB64, payloadB64, signatureB64] = idToken.split('.');
  const data = `${headerB64}.${payloadB64}`;
  const signature = Buffer.from(signatureB64, 'base64url');
  const verifier = crypto.createVerify('RSA-SHA256');
  verifier.update(data);
  verifier.end();
  return verifier.verify(key, signature);
}

export async function validateIdTokenSignature(idToken: string, discoveryUrl: string) {
  const { jwks_uri: jwksUri } = await fetchJson(discoveryUrl);
  const { keys } = await getJwks(jwksUri);
  const header = JSON.parse(Buffer.from(idToken.split('.')[0], 'base64url').toString('utf8'));
  const jwk = keys.find((k: any) => k.kid === header.kid);
  if (!jwk) return false;

  const key = importRsaKey(jwk);
  if (verifyJwtSignature(idToken, key)) return true;

  // Key rotation fallback: refetch JWKS once and retry
  jwksCache.keys = null;
  const { keys: freshKeys } = await getJwks(jwksUri);
  const freshJwk = freshKeys.find((k: any) => k.kid === header.kid);
  if (!freshJwk) return false;
  const freshKey = importRsaKey(freshJwk);
  return verifyJwtSignature(idToken, freshKey);
}
```

## Cache Invalidation

- Cache JWKS for 1 hour max.
- If signature validation fails, refetch JWKS once (key rotation scenario).
