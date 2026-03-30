# SECURITY INVARIANTS

## Purpose

This document defines the mandatory security rules that apply across all skills in this repository. These rules are the global baseline for safe implementation. Individual skills may introduce stricter controls, but they must not weaken the requirements defined here.

The purpose of this file is to provide a single source of truth for cross-cutting security expectations so that security guidance is not duplicated inconsistently across skills, providers, adapters, patterns, or examples.

---

## Scope

These invariants apply to all documentation-driven agent skills in this repository, including but not limited to:

- authentication and identity flows
- payment flows
- webhook-driven systems
- session and token handling
- third-party provider integrations
- callback and redirect-based workflows
- any flow involving secrets, trust boundaries, or user/account state transitions

---

## Core Security Principles

All skills in this repository must operate according to the following principles:

1. **Do not trust unverified external input**
   - Any data received from browsers, clients, callbacks, or webhooks must be treated as untrusted until verified.

2. **Critical state changes require backend authority**
   - Security-sensitive or financially meaningful state changes must be determined by trusted backend verification, not by client-side assertions.

3. **Identity, payment, and event workflows must be auditable**
   - Implementations must preserve enough structure, references, and state transitions to support debugging, reconciliation, and incident response.

4. **Security rules must be centralised and consistent**
   - Skills may reference these rules, but they must not redefine weaker alternatives elsewhere in the repository.

---

## Global Security Invariants

### 1. Backend verification is mandatory for critical outcomes

Client-side success indicators must not be treated as authoritative for:

- payment completion
- account linking completion
- login/session establishment where backend validation is required
- subscription activation
- webhook-driven fulfilment
- irreversible workflow transitions

If a provider exposes a client callback indicating success, that callback may be used for user experience only. Final state must be confirmed by trusted backend verification where applicable.

---

### 2. Webhook authenticity must be verified before processing

Any webhook or event callback must be verified before it is parsed into trusted business logic.

Required expectations:

- verify the request using the provider-approved verification mechanism
- prefer cryptographic verification where available
- reject or quarantine unverifiable events
- do not mutate application state before verification is complete

If a provider offers multiple verification options, the stronger method must be preferred unless the skill explicitly documents why a fallback method is acceptable.

---

### 3. Idempotency protections are required for externally triggered events

Implementations handling retries, duplicate callbacks, or replayable external events must include idempotency protections.

This applies especially to:

- webhooks
- payment verification callbacks
- subscription renewal events
- provider retry flows
- externally initiated account linking or provisioning callbacks

A duplicate event must not produce duplicate business effects.

---

### 4. Secrets must never be hardcoded or exposed in unsafe locations

Secrets must not be:

- hardcoded in source files
- committed to version control
- embedded in client-side bundles unless explicitly intended as public configuration
- logged in plaintext
- exposed in sample code without clear placeholder treatment

Secret handling expectations:

- use environment variables or approved secrets management systems
- keep test and live secrets separate
- minimise secret exposure in documentation examples
- avoid ambiguous naming that causes test/live confusion

---

### 5. Redirect and callback URIs must use exact-match configuration

Redirect, callback, and return URLs must not rely on permissive wildcard assumptions unless the provider explicitly supports and requires them and the skill documents the risks.

Required expectations:

- use exact-match configuration where possible
- make environment-specific URLs explicit
- do not accept arbitrary caller-controlled redirect values without validation
- document callback ownership clearly

This applies especially to OAuth, hosted checkout, billing portals, and any provider-managed redirect flow.

---

### 6. Token and session handling must follow the approved trust model

Tokens, session identifiers, or credentials must only be stored and transmitted according to the approved trust model for the chosen architecture.

Examples of unacceptable patterns unless explicitly justified by a skill:

- storing high-value tokens in insecure client storage
- exposing server-only secrets to browsers
- mixing session ownership models without clarity
- treating temporary client state as durable authentication state

Each implementation must clearly define:

- where session authority lives
- what token types exist
- where each token may be stored
- how renewal, revocation, and invalidation are handled

---

### 7. Test and live environments must remain strictly separated

Implementations must explicitly separate:

- credentials
- endpoints
- callback URLs
- data handling expectations
- webhook secrets
- verification workflows

Test assumptions must not silently leak into live workflows.

Skills should require environment clarity before implementation proceeds.

---

### 8. Sensitive operations require explicit ownership and boundary clarity

Any integration touching user identity, payments, subscriptions, or externally triggered state updates must clearly define:

- which component owns the state transition
- which system is authoritative
- when trust is transferred between systems
- what verification occurs before state is accepted

This rule exists to prevent hidden ambiguity between frontend, backend, provider, and callback layers.

---

### 9. Security-relevant failures must fail safely

If verification fails, a secret is missing, a provider response is malformed, or trust cannot be established:

- do not continue silently
- do not mark the flow as complete
- do not create irreversible side effects
- surface the condition clearly for investigation or retry

---

### 10. Examples must not weaken security rules

Examples are illustrative only.

Examples must not:

- bypass verification for convenience without clearly labeling that as non-production
- imply insecure defaults
- replace policy documents as the source of truth
- encourage copy-paste behaviour that skips discovery or trust-boundary analysis

---

## Override Policy

Skills may define stricter rules than this document.

Examples:

- a specific provider may require stronger signature handling
- a specific authentication flow may forbid a storage pattern that is otherwise conditionally allowed
- a specific domain may require additional verification checkpoints

However:

- no skill may weaken these invariants
- no provider or adapter doc may silently contradict these invariants
- if a stricter rule exists, it must be stated explicitly

---

## Repository Usage Rule

All skills must reference this file as the canonical source of cross-cutting security requirements.

Security guidance that is globally true across multiple skills should live here rather than being restated in multiple places.

---

## Maintenance Rule

When a new cross-skill security rule is introduced:

1. add or update it here first
2. update affected skills to reference it
3. remove duplicated or outdated variants elsewhere

This ensures long-term maintainability and prevents rule drift as the repository grows.
