# Existing Auth Integration

## Purpose

Defines how social authentication should be integrated into an application that already has an existing auth system.

## Relationship to Core Rules

Global execution and non-interference expectations are defined in:

- `../../../core/EXECUTION_RULES.md`
- `../AGENT_EXECUTION_SPEC.md`

This file defines only the reusable compatibility pattern for integrating with an existing auth system.

## When to Use

Use this file when the discovery report identifies an existing authentication model such as:

- server sessions
- JWT-based auth
- custom local login
- passwordless flows
- previously integrated providers

## Compatibility Questions

Before implementation, determine:

1. How does the application currently represent authenticated users?
2. How are sessions or tokens delivered today?
3. What user model already exists?
4. What logout and session invalidation behaviour already exists?
5. How should social login fit into that model without breaking it?

## Compatibility Pattern

### Existing Server Session Model
Social auth should:
- create the same kind of server-side session the app already uses
- preserve existing cookie format and session hydration expectations
- avoid introducing parallel token-based auth unless explicitly required

### Existing JWT Model
Social auth should:
- mint the same application JWT structure only after backend verification succeeds
- preserve existing issuer, expiry, and refresh patterns unless explicitly changing them

### Existing Mixed Model
Social auth should:
- preserve the current client contract
- avoid adding a second competing auth-delivery mechanism without approval

## User and Identity Mapping

- link social identities to the existing user model according to `../governance/account-linking.md`
- do not duplicate users unnecessarily
- do not assume email equality is sufficient unless the linking policy allows it

## Non-Interference Implications

- preserve existing login, logout, and session semantics unless explicitly changing them
- avoid refactoring unrelated auth middleware or guards
- document compatibility decisions clearly

## Output Implications

A correct implementation should be able to explain:

- what existing auth behaviour was preserved
- what new routes or handlers were added
- how session or token issuance stays compatible
- how account linking interacts with the current user model

## Maintenance Rule

- reusable integration guidance belongs here
- project-specific compatibility choices should be documented in the implementation output
