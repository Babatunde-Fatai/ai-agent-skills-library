---
name: supagent-core
description: Core superagent orchestration. Always active. Detects project context, manages .superagent/ structure, routes to specialized skills. Automatically invoked at conversation start.
auto_load: true
What problem it solves: Defines non-negotiable safety invariants and role separation.
Prerequisites: None — this is the primary prerequisite for implementation.
priority: 1
---

## Auto-Initialization Protocol

**This skill is always active. On every new conversation:**

## Execute this sequence automatically:

1. **Check for `.supagent/` directory**
   - if Found → Read `.supagent/AGENT_INIT.md` and proceed to step 3
   - Not found → Trigger initialization sequence below (step 2)

2. **Initialization Sequence** (when .superagent/ missing):
```
   a. Scan project files (package.json, requirements.txt, etc.)
   b. Detect project type and frameworks
   c. Create .supagent/ structure:
      - AGENT_INIT.md (context-aware routing - template at `references/AGENT_INIT_TEMPLATE.md`)
      - config.json (recommended skills - template at `references/CONFIG_TEMPLATE.json`)
      - MEMORY.md (empty, ready for use - template at `references/MEMORY_TEMPLATE.md`)
      - TASK_RULES.md (rules guiding tasks, ready for use - copy from `references/TASK_RULES_TEMPLATE.md`)
      - sessions/SESSION.md (empty, ready for use - template at `references/SESSION_TEMPLATE.md`)
   d. Inform user: "I've initialized Supagent for this project, important memories a"
   e. Show recommended skills based on detected type
```

3. **Standard Operation** (when .supagent/ exists):
```
   a. Load AGENT_INIT.md, if not found, trigger initialization sequence (step 2)
   b. Follow routing logic defined there
   c. Load additional skills only when needed
```

## User Never Needs to Ask

- Initialization happens silently on first interaction
- No explicit "setup" command needed
- Users just start working, system adapts