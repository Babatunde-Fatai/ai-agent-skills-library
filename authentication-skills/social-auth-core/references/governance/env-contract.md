# Environment Contract

## Purpose

Defines the environment-variable ownership model and configuration rules for social authentication.

## Relationship to Core Rules

Global security and environment-handling rules are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`

This file defines only social-auth-specific configuration governance.

## Configuration Ownership Model

### Backend (Server-Only)

These must never be exposed to the frontend:

- `SOCIAL_AUTH_<PROVIDER>_CLIENT_SECRET`
- `SOCIAL_AUTH_SESSION_SECRET`
- any signing keys or private credentials

### Backend-Controlled Public Values

These are controlled by the backend but may influence frontend integration:

- backend public URL
- callback routes
- redirect targets returned by the backend

### Frontend (Public)

Only allowed if required by the approved architecture:

- API base URL
- frontend callback success/error route
- other explicitly approved public integration values

## Naming Rules

Use consistent provider-scoped naming:

- `SOCIAL_AUTH_<PROVIDER>_CLIENT_ID`
- `SOCIAL_AUTH_<PROVIDER>_CLIENT_SECRET`
- `SOCIAL_AUTH_<PROVIDER>_REDIRECT_URI`
- `SOCIAL_AUTH_<PROVIDER>_SCOPES`
- `SOCIAL_AUTH_SESSION_SECRET`

Do not reuse the same variable names across different providers.

## Multi-Provider Scaling Rules

- each provider must have isolated environment variables
- no shared credentials across providers
- adding a provider must not affect existing provider configuration
- provider-specific callback and scope values must remain isolated

## Redirect URI Governance

- redirect URIs must match provider allowlists exactly
- redirect URIs must be backend-owned unless architecture explicitly documents otherwise
- redirect URIs must not be dynamically constructed from untrusted input
- local, staging, and production redirect URIs must be kept distinct and explicit

## Invalid Configurations

The following are forbidden:

- exposing client secrets to frontend runtime
- mixing frontend and backend ownership of auth credentials
- ambiguous callback URL ownership
- shared credentials between providers
- storing provider secrets in public build configuration

## Maintenance Rule

- social-auth-specific configuration governance belongs here
- global environment and security rules remain in `core/`
- provider-specific exceptions should be documented in provider docs, not here
