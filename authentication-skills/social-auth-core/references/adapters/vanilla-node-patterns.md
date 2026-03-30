# Vanilla Node Adapter Patterns

## Purpose

Provides server-only Node.js integration patterns for social authentication.

## Relationship to Core Rules

Global security rules defined in core/.

## Route Placement

/auth/:provider
/auth/:provider/callback

## Login Start Pattern

- generate state, nonce, verifier
- derive PKCE challenge
- store server-side
- redirect

## Callback Pattern

- validate state and nonce
- exchange token
- validate
- create session
- redirect

## Minimal Example

import http from 'http';
import crypto from 'crypto';

const server = http.createServer(async (req, res) => {
if (req.url?.startsWith('/auth/google')) {
const state = crypto.randomUUID();
const nonce = crypto.randomUUID();
const verifier = crypto.randomBytes(32).toString('base64url');

    res.writeHead(302, { Location: "PROVIDER_URL" });
    res.end();

}
});
