# DECISION MODEL

## Purpose

This document defines how overlapping documentation should be interpreted across the repository.

Its purpose is to make the documentation-driven skill system predictable, scalable, and maintainable by clearly defining:

- which document type has authority in which context
- how conflicts should be resolved
- how stricter rules interact with broader rules
- how examples should be treated relative to policy and execution documents

Without an explicit decision model, large documentation systems tend to accumulate ambiguity, duplication, and inconsistent implementation behaviour over time.

---

## Core Principle

**Not all documents have equal authority.**

Some documents define repository-wide rules.
Some define skill-specific behaviour.
Some define provider-specific constraints.
Some define framework-specific implementation notes.
Examples demonstrate patterns but do not define policy.

Agents and maintainers must resolve ambiguity according to the hierarchy below.

---

## Authority Hierarchy

When two documents overlap or appear to conflict, use the following order of authority from highest to lowest:

1. `core/SECURITY_INVARIANTS.md`
2. `core/EXECUTION_RULES.md`
3. `core/STOP_CONDITIONS.md`
4. skill-specific `AGENT_EXECUTION_SPEC.md`
5. skill-specific governance documentation
6. skill-specific pattern documentation
7. provider-specific documentation
8. adapter/framework-specific documentation
9. examples, samples, or illustrative snippets

---

## Interpretation Rules

### 1. Core files define cross-skill truth

Core files govern rules that apply across multiple skills.

Examples:

- trust boundaries
- verification expectations
- stop conditions
- execution order
- rule centralisation
- how conflicting docs should be interpreted

If a rule is globally true across multiple domains, it belongs in core.

---

### 2. Skill execution specs define domain-specific execution behaviour

`AGENT_EXECUTION_SPEC.md` exists to define the execution model for a specific skill.

It may:

- extend the core rules
- add stricter requirements
- define domain-specific discovery and routing logic
- clarify how supporting docs should be used within that skill

It must not:

- weaken core rules
- silently override core stop conditions
- redefine examples as authoritative

---

### 3. Governance docs define domain-specific policy and modelling decisions

Governance documentation is authoritative for domain-specific policy choices within a skill.

Examples:

- account linking rules
- identity merge policy
- ownership and lifecycle policy
- reconciliation and data modelling expectations

Governance docs should not redefine repository-wide security or execution policy unless the domain requires stricter controls.

---

### 4. Pattern docs define reusable approaches, not global policy

Pattern docs describe reusable domain patterns.

Examples:

- OAuth flow pattern
- session handling pattern
- webhook processing pattern
- verification pattern
- idempotency pattern

Patterns are important, but they do not outrank core policy or skill execution specifications.

---

### 5. Provider docs define provider-specific constraints

Provider documents are authoritative for provider-specific requirements.

Examples:

- required scopes
- callback quirks
- event naming
- signature formats
- transaction references
- subunit rules
- identity mapping specifics

Provider docs must stay provider-specific.

They must not become a second source of global policy.

---

### 6. Adapter docs define framework/runtime implementation constraints

Adapter documentation is authoritative for framework-specific implementation details.

Examples:

- request parsing behaviour
- session middleware patterns
- route placement
- SSR/CSR differences
- framework-specific callback handling

Adapter docs must not override provider requirements or core security rules.

---

### 7. Examples are illustrative only

Examples exist to demonstrate how a compliant implementation may look.

Examples do not:

- define policy
- create new allowed behaviour
- override stop conditions
- replace provider requirements
- weaken security invariants

If an example conflicts with a rule-bearing document, the example must be treated as non-authoritative.

---

## Conflict Resolution Rules

### Rule A: Higher authority wins

If two documents conflict, the document higher in the hierarchy wins.

---

### Rule B: Stricter rule wins when compatible

If a lower-level document introduces a stricter rule that does not weaken a higher-level rule, the stricter rule may apply within its scope.

Examples:

- a provider requiring additional verification steps
- a skill requiring additional discovery checks
- a framework adapter forbidding a pattern due to runtime constraints

---

### Rule C: Weaker rule loses

No lower-level document may weaken a stronger higher-level rule.

This includes:

- examples implying shortcuts
- provider docs omitting required stop conditions
- adapter docs bypassing verification expectations
- skill docs softening global security invariants

---

### Rule D: Unsupported ambiguity must be surfaced

If conflict cannot be resolved cleanly, the agent should surface the ambiguity rather than choose silently.

This is especially important where:

- provider support is partial
- examples are outdated
- framework behaviour has changed
- routing instructions do not match folder structure

---

## Document Ownership Model

To preserve maintainability, each layer should own a different type of truth.

### Core owns:

- shared security policy
- shared execution behaviour
- shared stop conditions
- shared documentation interpretation rules

### Skill files own:

- skill orchestration
- domain-specific discovery requirements
- domain-specific routing instructions
- stricter domain constraints

### Governance owns:

- domain policy and modelling rules

### Patterns own:

- reusable domain implementation approaches

### Providers own:

- provider-specific requirements and caveats

### Adapters own:

- framework/runtime implementation specifics

### Examples own:

- demonstrations of compliant usage

---

## Scalability Rule

As the repository expands, new skills must fit into this decision model rather than creating parallel authority systems.

This avoids:

- duplicated policy
- contradictory guidance
- unclear override behaviour
- maintenance fragmentation

---

## Maintenance Rule

When documentation is added or refactored:

1. decide which layer owns the truth
2. place the content in that layer
3. reference canonical content instead of copying it
4. remove weaker or duplicated alternatives

This is a core maintainability requirement for the repository.
