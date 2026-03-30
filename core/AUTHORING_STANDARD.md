# AUTHORING STANDARD

## Purpose

This document defines how documentation should be written, structured, and maintained across the repository.

Its goal is to ensure that the documentation-driven skill system remains:

- scalable
- maintainable
- consistent
- easy to review
- easy to extend
- predictable for agents and human contributors

This file is intended for maintainers and contributors who add, edit, or refactor skills and supporting documentation.

---

## Core Authoring Principle

**Write each truth once at the correct layer, then reference it elsewhere.**

The repository should avoid repeating the same guidance in multiple places unless there is a clear scoped reason to do so.

This reduces:

- rule drift
- contradiction
- review burden
- maintenance overhead
- confusion for agents

---

## Documentation Layers

The repository is organised into documentation layers. Each layer has a specific role.

---

### 1. Core layer

Location:

- `core/`

Purpose:

- repository-wide rules that apply across multiple skills

Examples of content that belongs here:

- security invariants
- execution order
- stop conditions
- decision hierarchy
- documentation authoring rules

What does **not** belong here:

- provider-specific rules
- framework-specific details
- domain-specific implementation quirks
- concrete example walkthroughs

---

### 2. Skill layer

Location:

- each skill's `SKILL.md`
- each skill's `README.md`
- each skill's `references/AGENT_EXECUTION_SPEC.md`

Purpose:

- define what the skill is for
- define how the skill should be applied
- define skill-specific discovery requirements
- define skill-specific routing into supporting docs
- define stricter domain-specific execution constraints where necessary

What does **not** belong here if already global:

- duplicated security baseline
- duplicated generic stop conditions
- duplicated generic execution rules

---

### 3. Governance layer

Location:

- `references/governance/`

Purpose:

- domain-specific policy, modelling, ownership, and lifecycle rules

Examples:

- account linking policy
- identity ownership rules
- reconciliation expectations
- lifecycle governance
- conflict resolution within the domain

---

### 4. Pattern layer

Location:

- `references/patterns/`

Purpose:

- reusable domain patterns that can be adapted across providers or frameworks

Examples:

- OAuth flow pattern
- session handling pattern
- verification pattern
- webhook processing pattern
- idempotency pattern

Patterns should explain reusable approaches without redefining global policy.

---

### 5. Provider layer

Location:

- `references/providers/`

Purpose:

- provider-specific requirements, caveats, and constraints

Examples:

- provider callback requirements
- provider scopes
- provider event schemas
- provider verification method
- provider-specific field mapping
- provider-specific payment rules

Provider docs must remain focused on what is unique to that provider.

---

### 6. Adapter layer

Location:

- `references/adapters/`

Purpose:

- framework/runtime-specific implementation details

Examples:

- route handling in Express
- session handling in Next.js
- callback parsing in Flask
- middleware differences in Rails

Adapter docs should not become generic security or policy docs.

---

### 7. Example layer

Location:

- wherever examples are stored within a skill

Purpose:

- demonstrate a compliant implementation pattern

Examples should be:

- clearly scoped
- easy to understand
- consistent with higher-level rules
- labeled when simplified or non-production

Examples are never the canonical source of truth.

---

## Canonical Ownership Rules

To preserve maintainability, every piece of content should have a clear owner.

### A rule belongs in `core/` if:

- it applies across multiple skills
- it is a cross-cutting security or execution rule
- it should be inherited by future skills

### A rule belongs in a skill if:

- it applies only to that skill/domain
- it defines skill-specific routing or discovery
- it is stricter than core within that skill

### A rule belongs in a provider doc if:

- it is unique to a provider
- changing provider would change the rule

### A rule belongs in an adapter doc if:

- it is unique to a framework/runtime
- changing framework would change the implementation detail

### A rule belongs in an example if:

- it is illustrating usage only
- removing the example would not remove canonical policy

---

## Duplication Rules

### Allowed duplication

Duplication is allowed only when:

- a short summary points to canonical content
- a skill adds a stricter scoped rule
- context requires a brief reminder before routing elsewhere

### Not allowed

The following should be avoided:

- copying full global rule blocks into multiple skills
- rewording the same invariant in many provider files
- placing the only important rule inside an example
- duplicating stop conditions across unrelated docs
- repeating repository-wide execution order in every provider doc

---

## Required Structure for `SKILL.md`

Every `SKILL.md` should include, at minimum:

1. skill title
2. purpose
3. when the skill should be used
4. mandatory reading order
5. skill-specific discovery requirements
6. routing guidance to supporting docs
7. any stricter skill-specific constraints
8. implementation output expectations, if useful

The file should be orchestration-focused, not overloaded with repeated policy.

---

## Required Structure for `AGENT_EXECUTION_SPEC.md`

Every `AGENT_EXECUTION_SPEC.md` should include, at minimum:

1. purpose
2. relationship to core rules
3. skill-specific execution model
4. skill-specific discovery expectations
5. skill-specific stop conditions, if stricter
6. routing logic to governance/pattern/provider/adapter docs
7. implementation expectations
8. any scoped constraints that extend core rules

This file should define how the skill operates, not restate the entire repository policy.

---

## Naming and Consistency Rules

Use consistent naming across the repository.

Recommended conventions:

- uppercase markdown filenames for core authority docs where already established
- clear and stable folder names
- no ambiguous placeholders that look fully supported when they are not
- use consistent terms for the same concept across files

Examples:

- do not alternate unpredictably between "execution spec" and "agent contract" unless intentionally differentiated
- do not use one provider name in folder form and another in doc titles without explanation
- do not use hidden shorthand that a new maintainer would not understand

---

## Cross-Reference Rules

When referencing canonical content:

- link or point to the owning document
- do not rewrite full content unless a scoped stricter rule is needed
- keep summaries shorter than the canonical source
- make it clear when the current document is extending rather than redefining a rule

Example:

- "This skill extends the global verification expectations in `core/SECURITY_INVARIANTS.md`."

This style improves maintainability and reduces contradiction risk.

---

## Support Status Rules

If a provider or feature is not fully supported, that status must be explicit.

Do not leave ambiguous structural signals such as:

- credentials folders with no provider guide
- provider names in folder trees without routing support
- partial stubs that appear production-ready

Supported status should be made clear using:

- provider docs
- status notes
- explicit exclusions in routing logic

---

## Example Quality Rules

Examples should:

- be consistent with current rules
- avoid insecure shortcuts unless clearly labeled as non-production
- be concise enough to support comprehension
- not replace discovery or rule interpretation
- be updated when higher-level rules change

When an example becomes outdated, it should be corrected or removed.

---

## Review Checklist for Maintainers

Before merging documentation changes, check:

- Does this content live in the correct layer?
- Is any global rule being duplicated unnecessarily?
- Is the canonical owner clear?
- Does any example imply behaviour that violates a rule?
- Does this change introduce ambiguity about provider support?
- Does this edit improve or reduce maintainability?
- Would a new contributor understand where to place related future content?

---

## Scalability Rule

Every new skill added to the repository should be written in a way that still makes sense when the repository becomes significantly larger.

That means:

- keep global rules centralised
- keep skill files focused
- keep provider docs provider-specific
- keep adapters framework-specific
- keep examples illustrative
- avoid structures that require editing many files for one policy change

---

## Maintenance Rule

When refactoring or adding content:

1. identify the correct owner layer
2. place the canonical rule there
3. update dependent docs to reference it
4. remove duplicated or weaker variants
5. keep the structure understandable for future contributors

This is the default standard for maintaining the repository over time.
