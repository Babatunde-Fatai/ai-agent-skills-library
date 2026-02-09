# Vue OAuth Patterns (Frontend Only)

**When to read:** When implementing the client-side login entry point in Vue.
**What problem it solves:** Vue-specific guidance for login buttons and redirects.
**When to skip:** If you are not building a Vue frontend.
**Prerequisites:** Read `references/AGENT_EXECUTION_SPEC.md` and select a backend adapter.

## Login Button
Use a full-page redirect to the backend login start route.

Example:
```vue
<template>
  <a :href="loginUrl">Sign in with Google</a>
</template>

<script setup>
const loginUrl = '/auth/google';
</script>
```

## Redirect Handling
- Let the backend handle provider redirects and session creation.
- Use frontend routes only for post-login navigation or error display.
- Do not parse or store tokens in the browser.

## Checklist
- [ ] Login button points to backend start route (for example, `/auth/google`).
- [ ] No tokens stored in localStorage or cookies.
- [ ] Frontend relies on backend redirects for auth completion.
- [ ] Error UI reads sanitized query params only (no raw tokens).
