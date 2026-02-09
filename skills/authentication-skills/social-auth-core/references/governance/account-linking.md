# Account Linking and Multi-Provider Coexistence

**When to read:** When linking accounts or merging by email.
**What problem it solves:** Prevents account takeover during linking.
**When to skip:** If account linking is out of scope.
**Prerequisites:** Read `references/patterns/data-models.md`.

## Linking Policy

- **Explicit Consent:** Prefer linking only when the user is already logged in or explicitly consents.
- **Trusted Email Rule:** You may **only** auto-link an OAuth identity to an existing account if:
    1. The emails match exactly (normalize to lowercase).
    2. The *new* provider asserts the email is **verified** (`email_verified: true`).
    3. The *existing* account has a verified email.
- **Unverified Emails:** Never auto-merge on unverified emails.

## Data Model Requirements

- **Separation of Concerns:** Store provider identity records in a separate table (e.g., `Account` or `Identity`) keyed by `provider` and `providerAccountId`.
- **Uniqueness:** Enforce a compound unique index on `[provider, providerAccountId]`.
- **One-to-Many:** Allow multiple provider identities to map to a single `User` entity.

## Logic & Security Flow (Implementation Guide)

### 1. The "Trusted Email" Check
When a user logs in via a Provider:
1. Search for an existing user by email.
2. **IF** user exists:
   - Check `Provider.email_verified`.
   - **IF TRUE:** Link the new identity to the existing user. (Safe auto-link).
   - **IF FALSE:** **STOP.** Return error: "Please verify your email with the provider first."
3. **IF** user does not exist:
   - Create new User + Identity.
   - Set User email verified status based on `Provider.email_verified`.

### 2. Pre-Account Takeover Protection
*Scenario:* An unverified account exists (e.g., someone signed up with `target@gmail.com` but never clicked the magic link). A real user now logs in with Google as `target@gmail.com`.

**Required Behavior:**
- If the existing local account is **unverified**, and the incoming OAuth login is **verified**, the OAuth login MUST supersede/claim the account.
- Update the user's email verified status to `true`.
- Link the provider identity.
- *Reasoning:* A verified provider is a stronger proof of ownership than an unverified local registration.

## Conflict Handling

- **ID Collision:** If `provider` + `providerAccountId` is already linked to a *different* user ID, stop and surface a "Account already connected to another user" error.
- **Duplicate Prevention:** Ensure your database schema constraints prevent creating duplicate rows for the same provider identity.
