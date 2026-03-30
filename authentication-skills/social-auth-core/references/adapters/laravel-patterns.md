## `references/adapters/laravel-patterns.md`

````md
# Laravel Adapter Patterns

## Purpose

Provides Laravel-specific integration patterns for social authentication.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file defines Laravel-specific routing, session, and callback implementation guidance only.

## When to Use

Use this file when social authentication is being implemented in a Laravel application.

## Route Placement

Define login start and callback routes in `routes/web.php` so they run with session-aware web middleware.

Example:

```php
use Illuminate\Support\Facades\Route;

Route::get('/auth/{provider}', [SocialAuthController::class, 'start']);
Route::get('/auth/{provider}/callback', [SocialAuthController::class, 'callback']);
```
````
