# Data Model Patterns

**When to read:** When changing or validating auth-related schemas.
**What problem it solves:** Multi-provider user/account modeling.
**When to skip:** If schema is already compliant and unchanged.
**Prerequisites:** Read `references/governance/account-linking.md`.

**ADAPTABILITY RULE:** Do not copy-paste these schemas. Adapt them to the project's specific ORM (Prisma, TypeORM, Mongoose, Sequelize) and naming conventions (camelCase vs snake_case).

## The Core Concept: 1 User, N Identities
You must separate the "Human" (User) from the "Login Method" (Account/Identity).

## Normalized Schema Pattern (Relational / SQL)

### 1. User Table
*The profile entity.*
- `id`: Primary Key
- `email`: Unique, Indexed
- `emailVerified`: Timestamp (Nullable)
- `image`: URL
- `name`: String

### 2. Account (or Identity) Table
*The authentication link.*
- `id`: Primary Key
- `userId`: Foreign Key -> User.id
- `provider`: String (e.g., 'google', 'github')
- `providerAccountId`: String (External ID, e.g., Google's 'sub')
- `type`: String ('oauth', 'oidc')
- **Tokens (Encrypted at rest if possible):**
  - `access_token`: Text
  - `refresh_token`: Text
  - `expires_at`: Int
  - `token_type`: String
  - `scope`: String
  - `id_token`: Text

### 3. Session Table (If not using JWTs)
- `sessionToken`: Unique, Indexed
- `userId`: Foreign Key -> User.id
- `expires`: Timestamp

**Required Constraints:**
- Compound Unique Index on Account: `(provider, providerAccountId)`

---

## Document Schema Pattern (NoSQL / MongoDB)

Embed accounts *or* reference them, depending on access patterns.

```json
// User Collection
{
  "_id": "ObjectId",
  "email": "user@example.com",
  "emailVerified": ISODate("..."),
  "name": "User Name",
  "accounts": [
    {
      "provider": "google",
      "providerAccountId": "123456789",
      "accessToken": "...",
      "refreshToken": "..."
    }
  ]
}

Constraint: Ensure accounts.provider + accounts.providerAccountId is unique across the collection.

---
