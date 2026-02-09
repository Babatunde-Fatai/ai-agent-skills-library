# Task Rules & Standard Patterns

## When You Get a "Build Feature X" Request

0. **Leave important code comments** don't overdo it, but leave enough to help me understand what it does
1. **Check MEMORY.md** for related prior decisions
2. **State assumptions** explicitly before coding (list 3-5)
3. **Ask 1-2 clarifying questions** if the request is ambiguous
4. **Create sub-agent code review** (mandatory for: refactors, security changes, or new code or integrations)
5. **Update MEMORY.md** with the decision you made + rationale

## When You Get a "Fix Bug X" Request

1. **Write a test that reproduces it** (or confirm existing test fails)
2. **Implement fix** only to make that test pass
3. **Run full test suite** to ensure no regressions
4. **Create doc in /docs** ONLY if:
   - The fix is non-obvious (others need to understand it)
   - OR it's a production-critical bug (document for team)
5. **Update MEMORY.md** with issue resolution status

## When You Get a "Optimize/Refactor X" Request

1. **Measure baseline** (if possible) - what metric are we improving?
2. **State the tradeoff** - what are we sacrificing?
3. **Apply surgical changes only** (follow karpathy-guidelines)
4. **Test thoroughly** - regression suite + load tests if applicable
5. **Create sub-agent code review** (mandatory for large refactors)

## When You Create a Doc File

**EVERY new doc must have a header like this:**
```markdown
---
Purpose: [1 sentence]
Audience: [Who should read this?]
Longevity: [Permanent / 2 weeks / Project-specific]
Last Updated: [Date]
Next Review: [Date]
---
```

**And a table of contents** if >100 lines.

**After creating, immediately add to MEMORY.md**.

## When You Complete a Session

**Output format for Babatunde:**
```
## Session Report [Date]

### Completed
- [TASK 1] - [Brief result]
- [TASK 2] - [Brief result]

### Decisions Made
1. [Decision + 1-line rationale]
2. [Decision + 1-line rationale]

### Open Items
- [Item] - Owner: [Babatunde/Next Agent] - Priority: [HIGH/NORMAL]

### Docs Changed
- Created: [doc names]
- Updated: [doc names]

### Sub-Agent Reviews
- [Review 1 result: PASS/ISSUES]
- [Review 2 result: PASS/ISSUES]
```

Keep it **under 30 lines total**.

## Red Flags (Ask Babatunde Before Proceeding)

- Changing authentication or security logic without full test coverage
- Modifying database schema without migration plan
- Adding new external dependencies without asking (new npm packages, APIs, etc.)
- Creating docs that overlap with existing docs (consolidate instead)