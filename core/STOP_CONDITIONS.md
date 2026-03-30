# STOP CONDITIONS

## Purpose

This document defines the conditions under which an agent must stop implementation and surface missing or unresolved requirements instead of continuing with assumptions.

Its purpose is to protect correctness, security, and maintainability by preventing undocumented guesswork in high-risk workflows.

This file is especially important in a documentation-driven skill system because silent assumptions scale badly and become harder to detect as the repository grows.

---

## Core Principle

**If a missing detail materially affects implementation shape, trust boundaries, security, or correctness, the agent must stop and surface the gap.**

Stopping does not mean failure. It means the system has correctly identified that implementation cannot safely proceed without additional clarity.

---

## Global Stop Conditions

An agent must stop implementation if any of the following are unresolved and relevant to the requested task.

---

### 1. Provider is unknown

Stop if the implementation depends on a third-party provider and the provider has not been identified clearly.

Examples:

- social login requested, but provider unknown
- payment integration requested, but gateway unknown
- webhook handling requested, but event source unknown

Why this matters:

- provider-specific rules, callback formats, verification methods, and constraints differ materially

---

### 2. Framework or runtime is unknown

Stop if the implementation depends on framework-specific integration details and the framework/runtime is not known.

Examples:

- OAuth integration requested, but framework unknown
- payment callback route requested, but backend stack unknown
- adapter-specific guidance required, but runtime unspecified

Why this matters:

- storage, routing, middleware, request parsing, and deployment constraints differ across frameworks

---

### 3. Trust boundary ownership is unclear

Stop if it is not clear which component owns the authoritative state transition.

Examples:

- unclear whether login session is owned by backend or frontend
- unclear whether payment status comes from webhook or verify endpoint
- unclear whether account linking is initiated client-side but confirmed server-side

Why this matters:

- ambiguity here leads directly to incorrect or insecure implementations

---

### 4. Verification strategy is undefined

Stop if the flow depends on verification but the required verification strategy is missing or unclear.

Examples:

- webhook present, but verification method unspecified
- hosted payment flow present, but backend verification path missing
- callback-based auth flow present, but token validation responsibility unclear

Why this matters:

- critical outcomes must not rely on unverified external signals

---

### 5. Required credentials or environment configuration are missing

Stop if implementation requires secrets, keys, callback URLs, or environment-specific configuration that have not been identified.

Examples:

- no client ID or client secret strategy
- no webhook secret
- no return/callback URL plan
- test/live environment unknown

Why this matters:

- configuration affects security, routing, verification, and deployment correctness

---

### 6. Existing system behaviour may conflict with the proposed flow

Stop if the target system already contains related functionality and the interaction model is unclear.

Examples:

- existing auth system already present, but social login integration pattern unclear
- existing payment state model present, but provider reference persistence unclear
- existing session handling present, but new token strategy may conflict

Why this matters:

- introducing a new integration without understanding existing boundaries causes fragile systems and maintenance debt

---

### 7. Redirect or callback ownership is unclear

Stop if the implementation requires redirect or callback handling and it is not clear:

- which component receives it
- which origin is registered
- which environment it belongs to
- how post-callback state is reconciled

Why this matters:

- callback ownership errors break security, routing, and environment configuration

---

### 8. Data/state mapping is unclear for critical events

Stop if the implementation requires mapping provider data into application state and the required persistence or reconciliation model is undefined.

Examples:

- no plan for storing transaction references
- no plan for linking provider identity to internal user record
- no plan for deduplicating webhook event IDs
- no plan for subscription lifecycle reconciliation

Why this matters:

- without explicit mapping, systems become unreliable and difficult to audit

---

### 9. Supported provider status is ambiguous

Stop if repository structure suggests a provider may be supported, but the actual implementation or routing support is incomplete or undocumented.

Examples:

- credentials folder exists but provider guide does not
- provider appears in folder structure but not in skill routing
- provider is partially scaffolded but not production-ready

Why this matters:

- false support assumptions lead to broken or incomplete implementations

---

### 10. A stronger governing rule is in conflict with an example

Stop if an example implies a path that appears to violate:

- a core security invariant
- a stop condition
- a provider requirement
- a skill execution constraint

Why this matters:

- examples are not authoritative and must not silently override rules

---

## Required Output When Stopping

When a stop condition is triggered, the agent should surface a concise structured discovery block containing:

1. what is missing
2. why it matters
3. what must be clarified before implementation continues
4. which docs become relevant once clarified

Example structure:

```md
## Cannot Proceed Yet

### Missing Information

- Provider not identified
- Framework/runtime not identified
- Verification strategy not defined

### Why This Blocks Implementation

These details determine routing, callback structure, verification logic, and security boundaries.

### Required Clarification

- Which provider is being used?
- Which backend/framework handles callbacks?
- How should verification be performed?

### Relevant Docs After Clarification

- provider docs
- adapter docs
- skill execution spec
```
