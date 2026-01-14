<ralph_loop_pattern>

# Ralph Loop Pattern

The Ralph loop is an autonomous execution pattern for AI coding agents, popularized by Geoffrey Huntley. GSD's implementation adapts this pattern to work with its context engineering and phase-based workflow.

## Core Concept

Run an AI agent in a loop until completion criteria are met. Each iteration:
1. Gets fresh context (no accumulated bloat)
2. Picks next incomplete task
3. Executes task fully
4. Persists learnings
5. Checks if done
6. Loops or exits

## How It Works in GSD

### The Loop Structure

```
ITERATION 1:
  - Find next incomplete PLAN.md (no matching SUMMARY.md)
  - Spawn fresh execute-plan subagent
  - Subagent loads: STATE.md, ralph-progress.txt, plan file
  - Subagent executes all tasks in plan
  - Subagent creates SUMMARY.md
  - Extract learnings → append to ralph-progress.txt
  - Update STATE.md and ROADMAP.md
  - Check completion: all plans have SUMMARYs?
    - YES: Output <promise>COMPLETE</promise>, exit
    - NO: Continue to iteration 2

ITERATION 2:
  - Find next incomplete PLAN.md
  - Spawn NEW fresh subagent (previous context discarded)
  - Subagent loads: STATE.md, ralph-progress.txt, plan file
  - Subagent sees learnings from iteration 1 in ralph-progress.txt
  - ... (same process)

ITERATION N:
  - Max iterations reached OR all plans complete
  - Output final status
  - Exit loop
```

### Memory Persistence

Memory persists between iterations through:

**1. Git history**
- Each task commits immediately
- Previous iterations' work visible in git log
- Subagents can `git show <hash>` to see past changes

**2. ralph-progress.txt**
- Append-only learnings file
- Each iteration adds: patterns, gotchas, decisions
- Accumulates project knowledge across iterations
- Subagents read full file for context

**3. STATE.md**
- Project memory and decisions
- Updated after each iteration
- Tracks: position, blockers, alignment status

**4. ROADMAP.md**
- Shows which plans have SUMMARY.md (complete)
- Loop uses this to find next plan
- Updated after each iteration

**5. Planning artifacts**
- PLAN.md files remain unchanged
- SUMMARY.md created when plan completes
- Presence of SUMMARY = plan done

### Fresh Context Per Iteration

**Key insight:** Each subagent spawns with clean context window.

**Old approach (context fills up):**
```
Agent loads 20k tokens
Does work → 40k tokens
Does more work → 60k tokens
Quality degrades → 80k tokens
"Being more concise now" → 100k tokens (poor quality)
```

**Ralph loop (always fresh):**
```
Iteration 1: Agent spawns, 20k tokens → work → done, context discarded
Iteration 2: NEW agent spawns, 20k tokens → work → done, context discarded
Iteration 3: NEW agent spawns, 20k tokens → work → done, context discarded
...
Quality stays consistent across all iterations.
```

### Loop vs Execute-Phase

**execute-phase:**
- Analyzes dependencies
- Spawns parallel subagents for independent plans
- All run simultaneously
- Fast for phases with 3+ independent plans

**ralph-loop:**
- Sequential execution
- One plan at a time
- Learns from previous iterations
- Good for phases with dependencies or when you want accumulated learnings

**When to use each:**

| Scenario | Use |
|----------|-----|
| Plans are independent, no cross-dependencies | execute-phase (parallel) |
| Plans build on each other, patterns emerge | ralph-loop (sequential, learning) |
| Want maximum speed | execute-phase |
| Want accumulated context and learnings | ralph-loop |
| Testing/debugging large codebase | ralph-loop (learns gotchas) |
| Greenfield build with clear plan | execute-phase |

## Configuration

### ralph-config.json

```json
{
  "ralph": {
    "max_iterations": 10,
    "auto_commit": true,
    "verification_level": "standard",
    "scope": "milestone",
    "stop_on_checkpoint": false
  }
}
```

**max_iterations:**
- How many times loop can run
- Prevents infinite loops
- Set based on scope:
  - Single phase (3-5 plans): 5-10
  - Multiple phases (10-15 plans): 20-30
  - Full milestone (20+ plans): 40-50

**auto_commit:**
- true: Commit after each task (default)
- false: Defer commits until plan complete

**verification_level:**
- basic: File creation + git commit
- standard: + build + smoke tests (default)
- comprehensive: + full test suite + linter

**scope:**
- Phase number (e.g., 1, 2.1): Loop over that phase only
- "milestone": Loop over all incomplete phases

**stop_on_checkpoint:**
- false: Orchestrator handles checkpoint, continues loop (default)
- true: Loop pauses, returns to user for input

## ralph-progress.txt Format

```
# Ralph Loop Progress
# Learnings, patterns, and context accumulated across iterations

Started: 2024-01-14 10:30:00 UTC
Scope: Phase 1 - Foundation
Max Iterations: 10

---

## Iteration 1 / 10
Time: 2024-01-14 10:30:15 UTC
Executing: .planning/phases/01-foundation/01-01-PLAN.md

✓ Plan completed: 01-01-PLAN.md

### Plan 01-01 Completed
Files: src/config.ts, src/types.ts
Commits: abc123f, def456g

Patterns:
- Config files live in src/config/
- Types use TypeScript interfaces, not type aliases
- All configs export default object

Gotchas:
- Environment variables must be validated at startup
- Config changes require server restart in dev mode

Decisions:
- Used Zod for config validation over joi (smaller bundle, better TS)
- Implemented singleton pattern for config access

---

## Iteration 2 / 10
...
```

Each iteration appends:
- Which plan executed
- Files modified
- Commit hashes
- Patterns discovered
- Gotchas encountered
- Decisions made

This creates growing knowledge base that informs future iterations.

## Stop Conditions

Loop exits when:

**1. All plans complete**
```
<promise>COMPLETE</promise>
All plans in Phase 1 executed successfully.
```

**2. Max iterations reached**
```
Max iterations (10) reached.
Completed: 7 / 10 plans
Remaining: [list plans]
```

**3. Consecutive failures (3)**
```
ERROR: 3 consecutive plan failures.
Last failure: [plan-path]
Check ralph-progress.txt for details.
```

**4. User cancellation**
User can stop loop anytime. Progress saved in ralph-progress.txt.

## Error Recovery

**Plan fails:**
- Log to ralph-progress.txt
- Increment failure counter
- Continue to next plan (don't retry immediately)
- If 3 consecutive failures: stop loop

**Build/test fails:**
- Subagent tries to fix within plan execution
- If can't fix: returns failure
- Orchestrator logs and moves to next plan

**Critical error:**
- Stop loop immediately
- Report to user
- Preserve state in ralph-progress.txt

## Best Practices

**Start small:**
```
/gsd:ralph-loop 1  # Single phase first
```
Learn the pattern before running full milestone.

**Set realistic max_iterations:**
- 1 plan = 1-2 iterations (usually 1)
- Allow buffer for potential retries
- Phase with 5 plans: set max_iterations: 7-10

**Review ralph-progress.txt:**
After loop completes, review learnings:
- Patterns → Document in codebase
- Gotchas → Add to AGENTS.md or comments
- Decisions → Update PROJECT.md constraints

**Use appropriate verification:**
- Development: "basic" (fast feedback)
- Pre-merge: "standard" (confidence without slowness)
- Release: "comprehensive" (full validation)

**Handle checkpoints:**
- stop_on_checkpoint: false (default) - For overnight runs
- stop_on_checkpoint: true - When architectural input needed

**Monitor progress:**
```bash
# While loop runs
tail -f .planning/ralph-progress.txt

# Check status
/gsd:progress
```

**Adjust scope incrementally:**
```
# Start narrow
/gsd:ralph-loop 1

# Once confident
/gsd:ralph-loop 2

# Eventually
/gsd:ralph-loop  # Full milestone
```

## Comparison to Traditional Ralph

**Traditional Ralph (snarktank/ralph, Amp-based):**
- Bash script spawning Amp CLI
- PRD as JSON with user stories
- Progress in prd.json + progress.txt
- Amp handles one story per iteration

**GSD Ralph Loop:**
- Native Claude Code slash command
- Plans as structured markdown
- Progress in ROADMAP.md + ralph-progress.txt
- Subagent handles full plan per iteration

**Shared principles:**
- Fresh context each iteration
- Persistent learnings file
- Git history as memory
- Loop until complete
- Autonomous "walk away" execution

**GSD advantages:**
- Integrated with GSD's phase/plan system
- Works with existing GSD commands
- No external dependencies
- Checkpoint support built-in

## Example Usage

### Execute single phase

```bash
/gsd:ralph-loop 1
```

Ralph will:
1. Ask for configuration (max iterations, etc.)
2. Find all plans in phase 1
3. Execute each sequentially with fresh context
4. Accumulate learnings in ralph-progress.txt
5. Stop when all phase 1 plans have SUMMARY.md

### Execute entire milestone

```bash
/gsd:ralph-loop
```

Ralph will:
1. Ask for configuration
2. Find all incomplete plans across all phases
3. Execute in order (phase 1 → phase 2 → ...)
4. Learn patterns that apply to later phases
5. Stop when all plans complete or max iterations

### Resume after interruption

```bash
/gsd:ralph-loop
```

Ralph detects existing ralph-progress.txt and:
- Appends new session marker
- Continues from where it left off
- Preserves previous learnings
- Finds next incomplete plan

## Limitations

**Not suitable for:**
- Plans requiring human architectural decisions mid-execution
- Plans with external dependencies (API keys, access)
- Plans where context is critical across all plans (use execute-phase)

**Better suited for:**
- Greenfield builds with clear requirements
- Refactoring/migration tasks
- Test coverage expansion
- Documentation generation
- Repetitive implementation patterns

**Debugging:**
If loop gets stuck or produces poor results:
1. Check ralph-progress.txt for patterns in failures
2. Review plan quality (too vague? too large?)
3. Lower max_iterations, run smaller scopes
4. Use execute-plan manually for problematic plans
5. Adjust verification_level if builds/tests flaky

</ralph_loop_pattern>
