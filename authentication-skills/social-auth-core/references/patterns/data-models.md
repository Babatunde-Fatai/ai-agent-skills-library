# Data Model Patterns

## Purpose

Defines reusable data-model patterns for multi-provider social authentication.

## Relationship to Core Rules

Global identity-linking policy is governed by:

- `../governance/account-linking.md`

This file defines model patterns only. It does not replace linking governance.

## When to Use

Use this file when creating or validating schema changes for social authentication.

## Core Model Principle

Separate:

- the human/application user
- the external provider identity

A single user may have multiple external identities.

## Relational Pattern

### User Table

Represents the local application user.

Typical fields:

- `id`
- `email`
- `emailVerified`
- `name`
- `image`
- audit timestamps

### Identity / Account Table

Represents the provider login method.

Typical fields:

- `id`
- `userId`
- `provider`
- `providerAccountId`
- `type` (`oauth` or `oidc`)
- token metadata if stored
- granted scopes if useful
- audit timestamps

### Constraints

- unique `(provider, providerAccountId)`
- foreign key from identity to user
- indexes supporting provider lookup and user linkage

## Token Storage Consideration

If tokens are stored:

- keep them in the identity/account table or a related token table
- encrypt sensitive values at rest
- separate long-lived secret material from broad user-profile concerns where possible

## Document Pattern (NoSQL)

Where a document model is used, preserve the same conceptual separation:

- stable local user document
- nested or related provider identities
- unique provider identity constraint enforced by the application or datastore

## Multi-Provider Expectations

A correct model should support:

- multiple providers per user
- disconnecting one provider without deleting the user
- linking policy enforcement
- provider lookup by `(provider, providerAccountId)`

## Anti-Patterns

Avoid:

- using email as the sole external identity key
- storing provider subject/account ID only on the user row when multiple providers may exist
- mixing provider-specific token state into unrelated profile fields

## Maintenance Rule

- reusable model patterns belong here
- concrete ORM or migration examples should be adapted to the target project
