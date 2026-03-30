# Vue Adapter Patterns

## Purpose

Frontend integration patterns for Vue apps.

## Relationship to Core Rules

Global rules are defined in core/.

## Login Pattern

<template>
  <button @click="startLogin">Sign in</button>
</template>

<script setup>
const startLogin = () => {
  window.location.href = '/auth/google';
};
</script>

## Rules

- frontend must not handle tokens
- backend handles auth
- frontend only triggers login and displays result
