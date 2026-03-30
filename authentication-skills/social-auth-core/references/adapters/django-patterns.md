# Django Adapter Patterns

## Purpose

Provides Django-specific integration patterns for social authentication.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file defines Django-specific routing, view, session, and callback implementation guidance only.

## When to Use

Use this file when the target application uses Django and social authentication must be implemented within Django routing and request handling.

## Route Placement

Define login start and callback routes in `urls.py`.

Example:

```python
from django.urls import path
from . import views

urlpatterns = [
    path("auth/<provider>/", views.oauth_start, name="oauth_start"),
    path("auth/<provider>/callback/", views.oauth_callback, name="oauth_callback"),
]
```
