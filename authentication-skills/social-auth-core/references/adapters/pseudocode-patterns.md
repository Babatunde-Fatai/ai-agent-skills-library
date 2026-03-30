# Pseudocode Adapter Patterns

## Purpose

Provides framework-agnostic adapter guidance for social authentication when no dedicated framework adapter is available.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- ../../../core/SECURITY_INVARIANTS.md
- ../../../core/EXECUTION_RULES.md
- ../../../core/STOP_CONDITIONS.md
- ../../../core/DECISION_MODEL.md

This file defines framework-agnostic adapter patterns only.

## When to Use

Use this file only when no framework-specific adapter applies or when creating a new adapter for an unsupported runtime.

---

## Login Start Pattern

handle GET /auth/<provider>:
state = secureRandom(32 bytes)
nonce = secureRandom(32 bytes)
verifier = secureRandom(32 bytes)

challenge = base64url(sha256(verifier))

storePreAuthState(
provider: provider,
state: state,
nonce: nonce,
verifier: verifier,
expiresIn: 10 minutes,
oneTimeUse: true
)

authUrl = buildAuthUrl(
provider: provider,
clientId: ENV.CLIENT_ID,
redirectUri: ENV.REDIRECT_URI,
state: state,
nonce: nonce,
codeChallenge: challenge,
codeChallengeMethod: "S256",
scope: "openid email profile",
responseType: "code"
)

redirectTo(authUrl)

---

## Callback Pattern

handle GET /auth/<provider>/callback:

if providerError exists:
fail safely (do not create session)

if code missing:
reject request

preAuth = loadPreAuthState(provider)

if preAuth not found:
reject request

if provider mismatch:
reject request

if state mismatch:
reject request

if preAuth expired:
reject request

if preAuth already used:
reject request

tokens = exchangeCodeForTokens(
code: code,
codeVerifier: preAuth.verifier
)

if token exchange fails:
fail safely (do not create session)

if OIDC applies:
validateIdToken(
idToken: tokens.id_token,
expectedNonce: preAuth.nonce
)

profile = loadOrDeriveProfile(tokens)

if profile is invalid or incomplete:
handle according to account-linking policy

user = linkOrCreateUser(profile, provider)

if linking fails:
fail safely (no session created)

rotateSessionId()

persistSessionForUser(user)

markPreAuthStateAsUsed(preAuth)

redirectToApprovedDestination()

---

## Session Defaults

- prefer a server-side session store or encrypted backend-controlled pre-auth state
- use HTTPOnly, Secure, SameSite cookies for session identifiers
- rotate session after successful login
- avoid exposing token material to browser storage

---

## Language-Specific Notes

- Python: use secrets; prefer server-side sessions
- PHP: use random_bytes(); keep secret values server-side
- Ruby: use SecureRandom; prefer encrypted or server-side session handling
- Java: use SecureRandom; ensure framework session controls match auth requirements
- C#: use RandomNumberGenerator; protect session and cryptographic material

---

## When Creating a New Adapter

A new framework adapter should document:

- route placement
- session/pre-auth state storage approach
- callback handler placement
- cookie defaults
- framework-specific runtime caveats
- how secure redirects are returned

---

## Validation Checklist

- Pre-auth state is backend-controlled
- state is validated
- nonce is validated when OIDC applies
- PKCE uses S256
- Tokens remain server-side
- Session rotation occurs after successful login
- Pre-auth state is invalidated after use
- Redirect destinations are controlled and safe

---

## Maintenance Rule

- framework-agnostic implementation guidance belongs here
- once a framework gains enough usage, create a dedicated adapter
- generic policy belongs in ../../../core/
