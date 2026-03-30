## `references/adapters/flask-patterns.md`

````md
# Flask Adapter Patterns

## Purpose

Provides Flask-specific integration patterns for social authentication.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- `../../../core/SECURITY_INVARIANTS.md`
- `../../../core/EXECUTION_RULES.md`
- `../../../core/STOP_CONDITIONS.md`
- `../../../core/DECISION_MODEL.md`

This file defines Flask-specific routing, session, and callback implementation guidance only.

## When to Use

Use this file when social authentication is being implemented in a Flask application.

## Route Placement

Define login start and callback routes using Flask decorators.

Example:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/auth/<provider>")
def oauth_start(provider):
    ...

@app.route("/auth/<provider>/callback")
def oauth_callback(provider):
    ...
```
````
