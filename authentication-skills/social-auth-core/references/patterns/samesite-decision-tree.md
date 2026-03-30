# SameSite Decision Tree

## Purpose

Helps choose an appropriate SameSite policy for cookies used by social authentication flows.

## Relationship to Core Rules

Global security requirements remain in:

- `../../../core/SECURITY_INVARIANTS.md`

This file provides only a reusable decision aid for cookie policy selection.

## When to Use

Use this file when authentication relies on cookies and the deployment spans one or more origins.

## Decision Tree

### 1. Backend and frontend share the same site
Example:
- `app.example.com`
- `api.example.com` with cookie scoped appropriately

Recommended default:
- `SameSite=Lax`

### 2. Backend and frontend are on different subdomains of the same parent domain
Recommended default:
- `SameSite=Lax` if the cookie domain and navigation pattern support it

Validate:
- cookie domain
- redirect path
- whether browser requests needing credentials are same-site or cross-site

### 3. Backend and frontend are on completely different sites
Example:
- `myapp.com`
- `api.otherdomain.dev`

Recommended when unavoidable:
- `SameSite=None`
- `Secure=true`

Also required:
- credentialed requests configured correctly
- clear CSRF posture for any cookie-authenticated endpoints beyond the OAuth callback flow

### 4. Provider redirect callback
OAuth provider redirects are top-level navigations.

Implication:
- `SameSite=Lax` often works for the backend callback itself
- follow-up frontend/API requests may still behave differently depending on site topology

## Selection Guidance

Prefer:
1. `Lax` when possible
2. `None` only when architecture requires cross-site cookie delivery
3. `Strict` only when the auth flow and surrounding app behaviour can support it

## Operational Checks

Before finalizing SameSite:

- confirm backend callback domain
- confirm frontend-to-backend request pattern after login
- confirm whether cookies need to be sent on cross-site XHR/fetch
- confirm `Secure` can be enforced in deployed environments

## Maintenance Rule

- SameSite selection logic belongs here
- exact cookie configuration per framework belongs in adapter docs
