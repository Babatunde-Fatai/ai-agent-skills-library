# Account Linking Governance

## Purpose

Defines how social authentication identities are linked to user accounts safely and consistently.

## Relationship to Core Rules

Global security and trust-boundary rules are defined in:

- `../../../core/SECURITY_INVARIANTS.md`

This file defines only identity-linking policy for social authentication.

## Identity Model

- each provider identity must be stored separately
- each identity is uniquely identified by:
  - provider
  - provider account ID / subject identifier
- multiple provider identities may map to one user
- one provider identity must never map to multiple users

## Linking Policy

### Auto-Link Conditions

Auto-linking is allowed only if all of the following are true:

1. emails match exactly after normalization
2. provider email is verified
3. existing account email is verified
4. no conflicting identity record already exists

### Manual-Link Conditions

Require explicit user action or manual confirmation if:

- email is unverified
- provider identity is new but email matches an existing user
- multiple existing identities create ambiguity
- local account exists but linking confidence is insufficient

### Forbidden Link Conditions

Do not link if:

- email is unverified
- provider identity already belongs to another user
- identity mismatch cannot be resolved safely
- provider data is incomplete in a way that violates policy

## Email-Based Decisions

- verified email is required for trust-based auto-linking
- unverified email must not be used for automatic linking
- missing email requires alternate identity handling or restricted account flow
- email match alone is not sufficient unless policy conditions are satisfied

## Provider Subject Identifier Rules

- provider `sub` / provider account ID is the canonical external identity key
- it must always be stored
- it must not be used as a display name
- it must remain stable once linked unless provider migration rules are explicitly documented

## Collision Handling

- duplicate provider identity → reject linking
- same email across different providers → follow explicit linking policy
- ambiguous identity resolution → require manual confirmation
- already-linked external identity → do not silently reassign

## Multi-Provider Expectations

- users may have multiple linked providers
- adding a provider must not override existing identities
- linking must be idempotent and safe
- unlinking one provider must not corrupt the user’s remaining auth methods

## Audit Requirements

- linking decisions must be traceable
- failed linking attempts should be logged safely
- logs must not contain tokens, secrets, or sensitive provider payloads
- audit records should identify why a link was allowed, rejected, or escalated

## Maintenance Rule

- account-linking policy belongs here
- execution logic belongs in `AGENT_EXECUTION_SPEC.md`
- schema details belong in patterns or data-model docs where appropriate
