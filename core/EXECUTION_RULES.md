# EXECUTION RULES

## Purpose

This document defines the mandatory execution order and reasoning behaviour for agents using this repository. Its purpose is to standardise how skills are interpreted and applied so that implementations remain consistent, scalable, and maintainable as the repository grows.

This file exists to prevent agents from jumping directly to examples, skipping discovery, or making unsafe implementation assumptions.

---

## Scope

These execution rules apply to all skills in this repository, regardless of domain.

This includes:

- authentication skills
- payment integration skills
- provider-specific skills
- framework-specific adapters
- pattern libraries
- supporting governance documentation

---

## Core Execution Principle

**Discovery and rule interpretation must happen before implementation.**

Agents must understand:

- the problem context
- the provider context
- the framework/runtime context
- the trust model
- the security requirements
- the relevant stop conditions

before writing or suggesting implementation details.

---

## Mandatory Reading Order

Unless a skill defines a stricter order, the default execution order is:

1. Read the skill entry point (`SKILL.md`)
2. Read `core/SECURITY_INVARIANTS.md`
3. Read `core/EXECUTION_RULES.md`
4. Read `core/STOP_CONDITIONS.md`
5. Read the skill-specific `AGENT_EXECUTION_SPEC.md`
6. Read governance documentation relevant to the skill
7. Read pattern documentation relevant to the skill
8. Read provider-specific documentation
9. Read adapter/framework-specific documentation
10. Read examples last

This order is mandatory because rules and trust boundaries must be understood before implementation examples are considered.

---

## Required Discovery Sequence

Before implementation begins, an agent must identify all relevant contextual inputs for the skill.

At minimum, discovery must determine:

- the target provider
- the target framework/runtime
- the trust boundary between client, server, and provider
- the callback/redirect ownership model
- the secret/configuration strategy
- the verification strategy for critical events
- whether an existing auth/payment/session model already exists
- the intended environment or deployment context
- whether any missing information triggers a stop condition

If any required discovery input remains unresolved, the agent must not proceed as if assumptions are confirmed.

---

## Mandatory Behaviour Rules

### 1. Do not start with code

Agents must not start implementation by writing code first.

Code generation or implementation guidance must follow:

- discovery
- rule interpretation
- trust-boundary clarification
- stop-condition checking

---

### 2. Do not treat examples as authority

Examples are useful for illustration, but they are not the source of truth.

If an example conflicts with:

- a core rule
- a stop condition
- a skill execution spec
- a provider requirement

the example loses.

---

### 3. Prefer invariants over convenience

When a choice exists between:

- a shorter implementation path
- and a safer or more correct implementation path

the safer or more correct path must be preferred.

---

### 4. Surface assumptions explicitly

If the agent must proceed with assumptions for explanatory purposes, those assumptions must be stated clearly and must not be presented as confirmed facts.

This is especially important for:

- missing framework details
- unknown callback ownership
- unclear token/session models
- incomplete webhook strategies
- ambiguous provider support

---

### 5. Stop when required context is missing

If a required context element is missing and the absence materially affects correctness, security, or implementation shape, the agent must stop and surface the missing requirement rather than silently improvising.

This rule is enforced through `core/STOP_CONDITIONS.md`.

---

### 6. Apply the narrowest relevant documentation set

Agents should read all required rule layers, but they should only use the specific provider, adapter, governance, and pattern docs that are relevant to the discovered context.

This reduces noise and improves maintainability.

---

### 7. Respect existing system boundaries

If the target project already has:

- authentication
- sessions
- payment state
- subscription state
- data models
- callback endpoints

the agent must integrate with those boundaries rather than assuming a greenfield implementation.

---

## Execution Output Expectations

After discovery and interpretation, the agent should be able to produce one or more of the following:

- a discovery report
- a recommended implementation path
- identified risks and missing information
- a mapping of relevant docs to read next
- an implementation plan
- code or pseudocode consistent with the discovered context

The exact output may vary by skill, but it must be grounded in the rules above.

---

## Conflict Handling

If documentation overlaps or conflicts, agents must follow the repository decision hierarchy defined in:

- `core/DECISION_MODEL.md`

In general:

- core files define shared rules
- skill files define skill-specific execution
- provider files define provider-specific constraints
- adapter files define framework/runtime constraints
- examples demonstrate patterns only

---

## Scalability Rule

As the repository grows, new skills must align with this execution model rather than inventing their own incompatible flow.

This makes the system easier to:

- scale
- review
- extend
- audit
- maintain over time

---

## Maintenance Rule

When execution behaviour changes across the repository:

1. update this file first
2. update skill execution specs only where skill-specific differences are required
3. remove outdated duplicated execution guidance elsewhere

This preserves a single source of truth for cross-skill execution behaviour.
