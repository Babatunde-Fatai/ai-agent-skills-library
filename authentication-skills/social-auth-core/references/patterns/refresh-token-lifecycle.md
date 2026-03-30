# Refresh Token Lifecycle

## Purpose

Defines the reusable lifecycle for refresh-token storage, rotation, reuse detection, and revocation.

## Relationship to Core Rules

Global token-safety requirements remain in:

- `../../../core/SECURITY_INVARIANTS.md`

This file refines refresh-token-specific handling only.

## When to Use

Use this file when:

- the provider issues refresh tokens
- long-lived provider access is required
- the application refreshes provider access tokens after login

## Lifecycle Stages

### 1. Issue / Capture
- store the refresh token server-side only
- bind it to the provider identity and user
- record issued-at and expiry metadata where available

### 2. Use
- use the current valid refresh token only when access renewal is needed
- avoid refreshing on every request
- refresh within a controlled expiry window

### 3. Rotate
When the provider returns a new refresh token:

- store the new token atomically
- invalidate the old token immediately
- update related expiry metadata

### 4. Detect Reuse
If an old refresh token is seen after rotation:

- treat this as suspicious or compromised state
- revoke affected token chains if possible
- force re-authentication where appropriate

### 5. Revoke / Disconnect
When the user disconnects or logs out:

- revoke refresh capability if supported and required
- delete or invalidate local token records
- do not assume provider revocation succeeded unless confirmed

## Failure Handling

- refresh failure should not keep stale tokens active
- repeated refresh failures should converge toward re-authentication
- do not fall back to an old rotated token

## Minimal Data Requirements

Track enough information to support:

- active vs invalid refresh token status
- rotation history if needed
- provider and user linkage
- expiry and revocation state

## Maintenance Rule

- refresh-token lifecycle rules belong here
- provider-specific refresh quirks belong in provider docs
