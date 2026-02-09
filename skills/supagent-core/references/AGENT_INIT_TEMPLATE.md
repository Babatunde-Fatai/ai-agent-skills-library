# Agent Initialization - MANDATORY CHECKLIST

**This file controls agent behavior. Follow checkpoints in order.**

---

## CHECKPOINT 1: Session Setup (DO FIRST)

**DO NOT START ANY WORK until you complete these steps:**

1. **Create/Open Session File**
   - Check: `.supagent/sessions/YYYY-MM-DD-SESSION.md` (today's date)
   - If EXISTS: Read `## ACTIVE` section only
   - If MISSING: Create from template `.supagent/sessions/session-template.md`
   - Move any active tasks from previous day's session

2. **Read Standing Context**
   - `.supagent/MEMORY.md` - Check for conflicts with current task
   - `.supagent/TASK_RULES.md` - Task-specific instructions

3. **Load Required Skills**
   - Based on request and context, User mentions name and keywords from `.supagent/config.json` triggers:
   → Load corresponding skill if not in active-skills.json

   - User says "continue" or references previous work:
   → Check `.supagent/MEMORY.md` & latest SESSION.md for context, reactivate skills mentioned there or necessary

   - User starts new feature/task with no clear skill match:
   → Proceed normally, offer skill suggestions if applicable

   - Based on request and context, check `.supagent/config.json` for all available skills, name and keywords, and load any skill that is relevant to your current task. 

**YOU MUST acknowledge in your first response:**
```
Session: [created/opened] .claude/sessions/YYYY-MM-DD-SESSION.md
Memory: [read/no conflicts] or [read/found: <conflict summary>]
Skills: [e.g loaded karpathy-guidelines, ...]
```

---

## CHECKPOINT 2: During Work

**While working on tasks:**

- [ ] Update session file before starting each task
- [ ] Update `.supagent/MEMORY.md` immediately if before and after making key decisions
- [ ] Use TodoWrite tool or similar to track multi-step tasks

**If making a decision that future agents need to know:**
1. Add to `.supagent/MEMORY.md` under appropriate project section
2. Use format: `- [Decision] - [Date] - [Rationale] - Status: FINAL`

---

## CHECKPOINT 3: After Task Complete

**Before ending your response on a completed task:**

1. **Create Review Sub-Agent**
   - YOU MUST spawn a sub-agent to review your work
   - Sub-agent provides you actionable feedback and reports findings in structured format to me
   - if subagent not available, do a self and critical review yourself. Notes all issues found and how they were resolved and report findings. Dont just tell me PASS or FAIL.

2. **Update Session File**
   - Change `- [ ] Task` → `- [x] Task`
   - Change `Status: NOT STARTED` → `Status: ACTIVE` → `Status: DONE`
   - Move completed tasks to `## COMPLETED` section
   - Sessions must include ALL TASKS & ALL subtasks to be covered under each task, not just a vague description

3. **Update MEMORY.md**
   - Add any new decisions made, label appropriately e.g [architectural decision]... etc 
   - Update documentation scores if applicable

4. **Report to User**
   - Format: `[DONE]` / `[IN PROGRESS]` / `[BLOCKED]`
   - Max 50 lines per update
   - Decisions: state + 1-2 line rationale

---

## CHECKPOINT 4: Session Close

**Close session when:**
- All ACTIVE tasks are DONE or BLOCKED
- User says "that's it for today"
- Starting new session >24 hours later

**To close:**
1. Rename closing session file to: `YYYY-MM-DD-SESSION.md` → `YYYY-MM-DD-SESSION-[CLOSED].md`
2. Move remaining ACTIVE tasks to new session file

---

## CHECKPOINT 5: Reference - File Management Rules

**Docs Governance:**
- Only create `.md` in `.supagent/docs` if persistent (>10 days value knowledge about codebase or product)
- Max 5 active docs; archive oldest when creating #6
- Archive location: `.supagent/docs/archive/YYYY-MM/`
- Every new doc must be referenced in `MEMORY.md`

**Create docs for:**
- Architecture decisions
- API contracts
- Testing procedures
- Deployment steps
- important longterm info that cannot stay in Memory

**Delete (don't create docs for):**
- Debug logs
- "X vs Y" analysis (keep decision only)
- Session-specific breakdowns
- Exploratory notes

---

## CHECKPOINT 6: Edge Cases

**If MEMORY.md >500 lines:**
- Alert: "MEMORY.md growing large, consider splitting"
- Suggest moving older memories to new folder: `.supagent/memory/YYYY-MM-DD-SESSION.md`

**If unsure whether to create doc:**
- "Will the user AI need this in 2 weeks?" → Yes = create
- "Would another AI agent need this context in a few weeks?" → Yes = create with header
