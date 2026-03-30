# Agent Execution Spec (Social Auth Core)

## Purpose

This document defines the skill-specific execution model for social authentication.

It must be read before any design or code changes for this skill. It extends, but does not weaken, the global repository rules defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file is the operational contract for applying the social-auth skill after the global core rules have been read.

---

## When to Read

Read this document:

- before any social auth design decision
- before any code change
- before choosing implementation patterns
- before selecting provider or adapter docs

Do not skip it for social-auth tasks.

---

## What Problem It Solves

This document defines:

- how to execute the social-auth skill safely
- how to select implementation scope
- how to route to the correct supporting docs
- which social-auth-specific constraints are mandatory
- how to structure outputs deterministically

It exists so that social authentication work is performed consistently across:

- different providers
- different frameworks
- different deployment topologies
- greenfield and incremental integrations

---

## Quick Navigation

- **Core OAuth/OIDC flow**: `patterns/oauth-flow-core.md`
- **Provider docs**: `providers/`
- **Framework adapters**: `adapters/`
- **Governance**: `governance/env-contract.md`, `governance/error-edge-cases.md`, `governance/account-linking.md`, `governance/testing-validation.md`

---

## Relationship to Core Rules

This document extends the core rule layer.

### Core layer remains authoritative for:

- repository-wide security invariants
- execution order
- global stop conditions
- document conflict resolution

### This document is authoritative for:

- social-auth-specific execution order refinements
- role selection for frontend/backend/full-stack implementations
- social-auth-specific discovery expectations
- social-auth-specific output structure
- social-auth-specific non-interference and compatibility rules

If this document introduces a stricter rule than the core layer, the stricter rule applies within this skill.

---

## Skill-Specific Execution Order

After reading the core layer and `SKILL.md`, use this skill-specific sequence:

1. Determine implementation role:
   - frontend-only
   - backend-only
   - full-stack

2. Produce the Discovery Report.

3. Read all governance references in `governance/`.

4. Confirm provider and load:
   - `providers/{provider}/CREDENTIALS.md`
   - `providers/{provider}.md`

5. Read required patterns based on discovered architecture.

6. Read the required framework adapter.

7. Confirm:
   - callback ownership
   - session ownership
   - account-linking policy
   - redirect URI architecture
   - environment variable ownership

8. Plan file changes and environment variables.

9. Implement with security checklist in view.

10. Provide testing checklist and a Decisions Log documenting key architectural choices.

If any step is blocked by missing information, stop and surface the missing requirement.

---

## Discovery Requirements (Skill-Level)

The Discovery Report must be completed before implementation.

It must include:

- framework and runtime
- implementation role (frontend, backend, full-stack)
- existing auth system and session model
- user model and account-linking rules
- existing OAuth providers and routes
- deployment topology and redirect URI constraints
- environment variable system and secrets manager
- CSRF, rate limiting, and logging protections
- auth middleware and route guards
- whether flow is OAuth or OIDC
- whether multi-provider support is required

If any required information is missing, stop and ask.

### Credential Translation (Required)

After confirming the provider:

1. request credential format from user
2. load `providers/{provider}/CREDENTIALS.md`
3. map credentials to environment variables
4. confirm backend URL and callback URL
5. provide a concrete `.env` template
6. validate redirect URI matches provider configuration

No implementation should begin without validated credentials.

---

## Role Selection and Multi-Part Implementation

OAuth and OIDC social authentication require clear role ownership.

Before implementation, determine which role applies.

---

## Full-Stack Role (Recommended)

Use this role when the full codebase is available.

### Responsibilities

- implement backend routes first
- implement callback handling and validation
- create or integrate server-side session logic
- integrate frontend login initiation and auth-state handling second
- validate end-to-end behaviour across redirect boundaries

### Expected sequence

1. backend login start route
2. backend callback route
3. session creation and rotation
4. account creation or linking
5. frontend login trigger
6. frontend auth-state hydration
7. end-to-end testing

---

## Frontend Role (Client-Side Scope)

Use this role only when the backend already exists or is intentionally out of scope.

### Frontend MUST implement

- login button or link that redirects to a backend route
- optional loading and error UI
- auth-state hydration from the backend if required by the architecture

### Frontend MUST NOT implement

- provider token exchange
- state generation
- nonce generation
- PKCE verifier generation unless explicitly required
- client-side token validation
- localStorage token persistence
- direct session creation

### If backend uses cookie-based sessions

- configure API requests with credentials included
- derive auth state from the server
- do not infer auth state from URL params or localStorage

### Required backend-facing output

Document:

- login start route
- user hydration route
- logout route
- success/error redirect expectations
- credential requirements

---

## Backend Role (Server-Side Authority)

Use this role when implementing routes, handlers, sessions, storage, linking, or token validation.

### Backend MUST implement

- login start route
- callback route
- secure state generation and validation
- secure nonce handling for OIDC
- PKCE handling
- token exchange
- token validation
- server-side session creation
- account creation or linking
- session rotation

### Backend MUST NOT implement

- client-secret exposure to frontend
- trust in frontend-provided auth state
- skipping state validation
- skipping nonce validation
- token storage in client-accessible storage

### Required frontend-facing output

Provide:

- route definitions
- redirect behavior
- auth-state hydration endpoint
- logout flow
- cookie/session delivery expectations
- frontend env vars

---

## Non-Interference Rule (Critical)

When implementing social authentication:

- do NOT modify existing authentication systems unless explicitly instructed
- do NOT refactor unrelated auth middleware, guards, or session logic
- do NOT replace existing login flows unless required

If existing logic appears insecure:

- document the issue
- propose improvements
- do NOT implement changes without instruction

---

## Implementation Context Transfer

When implementation is split across roles, provide structured handoff.

### Backend-to-frontend

```json
{
  "loginRoute": "GET /auth/{provider}",
  "callbackBehavior": "Redirects to FRONTEND_URL/auth/callback?success=true|false",
  "userHydrationRoute": "GET /auth/me",
  "logoutRoute": "POST /auth/logout",
  "cookieConfig": "HTTP-only, Secure, SameSite=<value>",
  "fetchConfig": "credentials: 'include'",
  "frontendMustNot": [
    "extract tokens",
    "store tokens",
    "validate tokens client-side"
  ],
  "frontendEnvVars": ["API base URL"],
  "providers": ["{provider}"]
}
```
