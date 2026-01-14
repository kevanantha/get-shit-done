<purpose>
Execute plans in autonomous Ralph loop until completion criteria met.

Each iteration spawns a fresh execute-plan subagent with clean context. Memory persists via git history, ralph-progress.txt, and planning artifacts. The loop continues until all plans in scope are complete or max iterations reached.
</purpose>

<required_reading>
@.planning/STATE.md
@.planning/ROADMAP.md
@.planning/ralph-config.json (if exists)
@.planning/ralph-progress.txt (if exists)
</required_reading>

<process>

<step name="load_configuration" priority="first">
**Load Ralph configuration:**

```bash
cat .planning/ralph-config.json 2>/dev/null
```

**If exists:** Parse configuration:
- max_iterations (default: 10)
- auto_commit (default: true)
- verification_level (default: "standard")
- scope (phase number or "milestone")

**If missing:** Use defaults and create config file.
</step>

<step name="initialize_progress_file">
**Initialize or load progress file:**

```bash
cat .planning/ralph-progress.txt 2>/dev/null
```

**If missing:** Create with header:

```
# Ralph Loop Progress
# Each iteration appends learnings, patterns, and context

Started: $(date -u +"%Y-%m-%d %H:%M:%S UTC")
Scope: [phase/milestone description]
Max Iterations: [N]

---
```

**If exists:** Append new session separator:

```
---
## New Session: $(date -u +"%Y-%m-%d %H:%M:%S UTC")
```
</step>

<step name="discover_incomplete_plans">
**Discover plans in scope:**

```bash
# Load ROADMAP to find in-progress phase(s)
cat .planning/ROADMAP.md

# List plans in scope
if [ -n "$PHASE_NUM" ]; then
  PHASE_DIR=".planning/phases/${PHASE_NUM}-*"
else
  # All phases in current milestone
  PHASE_DIR=".planning/phases/*"
fi

# Find all PLAN.md files
find $PHASE_DIR -name "*-PLAN.md" | sort

# Find existing SUMMARY.md files
find $PHASE_DIR -name "*-SUMMARY.md" | sort
```

**Build incomplete list:**
- Plans with PLAN.md but no matching SUMMARY.md
- Store in array for iteration

**If no incomplete plans:**
```
<promise>COMPLETE</promise>

All plans in scope already executed.
```

Exit loop.
</step>

<step name="execute_loop">
**Main Ralph loop:**

```bash
ITERATION=1
MAX_ITER=[from config]
CONSECUTIVE_FAILURES=0

while [ $ITERATION -le $MAX_ITER ]; do
  echo "## Iteration $ITERATION / $MAX_ITER" | tee -a .planning/ralph-progress.txt
  echo "Time: $(date -u +"%Y-%m-%d %H:%M:%S UTC")" | tee -a .planning/ralph-progress.txt
  
  # Find next incomplete plan
  NEXT_PLAN=[first plan without SUMMARY from list]
  
  if [ -z "$NEXT_PLAN" ]; then
    echo "All plans complete!" | tee -a .planning/ralph-progress.txt
    echo "<promise>COMPLETE</promise>"
    break
  fi
  
  echo "Executing: $NEXT_PLAN" | tee -a .planning/ralph-progress.txt
  
  # Execute plan via subagent
  [Spawn execute-plan subagent for $NEXT_PLAN]
  
  # Check result
  if [ SUMMARY created successfully ]; then
    echo "✓ Plan completed: $NEXT_PLAN" | tee -a .planning/ralph-progress.txt
    CONSECUTIVE_FAILURES=0
    
    # Extract learnings from SUMMARY and append to progress
    [Parse SUMMARY for key decisions, patterns, gotchas]
    echo "Learnings:" | tee -a .planning/ralph-progress.txt
    [Append extracted learnings]
    
  else
    echo "✗ Plan failed: $NEXT_PLAN" | tee -a .planning/ralph-progress.txt
    CONSECUTIVE_FAILURES=$((CONSECUTIVE_FAILURES + 1))
    
    if [ $CONSECUTIVE_FAILURES -ge 3 ]; then
      echo "ERROR: 3 consecutive failures. Stopping loop." | tee -a .planning/ralph-progress.txt
      break
    fi
  fi
  
  # Update STATE.md with iteration results
  [Update STATE.md with current position and learnings]
  
  ITERATION=$((ITERATION + 1))
done

if [ $ITERATION -gt $MAX_ITER ]; then
  echo "Max iterations reached. Some plans may remain incomplete." | tee -a .planning/ralph-progress.txt
fi
```
</step>

<step name="spawn_execute_plan_subagent">
**Spawn fresh execute-plan subagent:**

For each plan, create a fresh subagent with full context:

```
Task(
  prompt="""
You are executing plan: {plan_path}

@.planning/STATE.md
@.planning/ralph-progress.txt
@~/.claude/get-shit-done/workflows/execute-plan.md

Follow the execute-plan workflow to:
1. Parse and execute all tasks in the plan
2. Handle checkpoints (return to orchestrator if needed)
3. Create SUMMARY.md when complete
4. Commit work per task

After completing all tasks, include in your response:
- SUMMARY.md path
- Key learnings (patterns, gotchas, decisions)
- All commit hashes

Ralph loop will extract learnings and continue to next plan.
""",
  subagent_type="general-purpose"
)
```

**Key points:**
- Fresh context per iteration (no accumulated bloat)
- ralph-progress.txt provides continuity between iterations
- STATE.md gives project-wide context
- Subagent returns learnings for progress file
</step>

<step name="extract_and_persist_learnings">
**After each successful plan execution:**

Parse subagent response and SUMMARY.md for:

**Patterns discovered:**
- "This codebase uses X for Y"
- "Files are organized by Z pattern"
- "Always check W before modifying V"

**Gotchas found:**
- "Do not forget to update A when changing B"
- "Component C requires D to be initialized first"
- "Test E needs environment variable F"

**Decisions made:**
- "Chose library X over Y because Z"
- "Implemented pattern P for consistency with Q"
- "Deferred R to future phase due to S"

**Append to ralph-progress.txt:**

```
### Plan {phase}-{plan} Completed
Files: [list modified files]
Commits: [commit hashes]

Patterns:
- [pattern 1]
- [pattern 2]

Gotchas:
- [gotcha 1]
- [gotcha 2]

Decisions:
- [decision 1]
- [decision 2]

---
```

This creates growing context for future iterations.
</step>

<step name="update_state_and_roadmap">
**After each iteration:**

**Update STATE.md:**
- Current position (last completed plan)
- Accumulated learnings from ralph-progress.txt
- Any blockers discovered
- Iteration count

**Update ROADMAP.md:**
- Mark completed plans with checkmark
- Update phase status if phase completed
- Note any plan reorderings or additions

This keeps project state synchronized for next iteration.
</step>

<step name="report_final_status">
**When loop exits:**

Present summary to user:

```
## Ralph Loop Complete

**Scope:** [phase N / full milestone]
**Iterations:** [X of Y]
**Plans Completed:** [N]
**Status:** [COMPLETE / PARTIAL / STOPPED]

### Completed Plans:
- [phase]-[plan]: [name]
- [phase]-[plan]: [name]
...

### Remaining Plans:
- [phase]-[plan]: [name] (if any)

### Key Learnings:
[Top 3-5 learnings from ralph-progress.txt]

### Next Steps:
[Suggest appropriate next command based on status]
```

**Status definitions:**
- COMPLETE: All plans in scope executed successfully
- PARTIAL: Some plans complete, stopped before finishing all
- STOPPED: Loop terminated due to errors or user intervention
</step>

</process>

<ralph_loop_orchestration>
**Orchestrator vs Subagent responsibilities:**

**Orchestrator (ralph-loop.md):**
- Loop control and iteration counting
- Finding next incomplete plan
- Spawning subagents
- Collecting and persisting learnings to ralph-progress.txt
- Updating STATE.md and ROADMAP.md
- Checking completion criteria and stop conditions
- Reporting final status

**Subagent (execute-plan.md):**
- Full plan execution (all tasks)
- Checkpoint handling (interactive)
- Per-task commits
- SUMMARY.md creation
- Return learnings to orchestrator

This separation ensures:
- Orchestrator stays lean (~10-15% context)
- Each subagent gets fresh 100% context
- No context degradation across iterations
</ralph_loop_orchestration>

<error_handling>
**Failure scenarios:**

**Plan execution fails:**
- Increment consecutive failures counter
- Log error to ralph-progress.txt
- If < 3 consecutive: continue to next plan
- If >= 3 consecutive: stop loop, report to user

**Build/test failures:**
- Subagent should fix and retry within plan
- If cannot fix: return failure
- Orchestrator logs and continues or stops per consecutive failure rule

**Checkpoint requires user input:**
- Subagent returns to orchestrator with checkpoint
- Orchestrator presents to user
- User responds
- Orchestrator resumes subagent
- Loop pauses but doesn't count as failure

**Max iterations reached:**
- Stop loop
- Report partial completion
- List remaining plans
- User can re-run with higher max or execute remaining manually
</error_handling>

<verification>
**Verification levels:**

**basic:**
- SUMMARY.md created
- Git commit successful
- No syntax errors

**standard (default):**
- SUMMARY.md created
- All commits successful
- Build succeeds (if build command available)
- Basic smoke tests pass

**comprehensive:**
- SUMMARY.md created
- All commits successful
- Full build succeeds
- All tests pass
- Linter passes (if configured)
- No new warnings

Configure in ralph-config.json. Higher levels take longer but ensure quality.
</verification>

<anti_patterns>
**Avoid these mistakes:**

❌ **Loading all plan content into orchestrator**
- Keeps orchestrator context bloated
- Defeats fresh context benefit

✅ **Reference plans by path, load in subagent**
- Orchestrator: track paths and status
- Subagent: load and execute full plan

❌ **Accumulating execution history in loop**
- Context fills with logs
- Quality degrades

✅ **Persist to ralph-progress.txt, load summary only**
- Full history in file
- Loop reads high-level summary only

❌ **Skipping failure tracking**
- Loop continues indefinitely on errors
- Wastes tokens and time

✅ **Track consecutive failures, stop at threshold**
- Prevents infinite loops
- Surfaces blockers quickly
</anti_patterns>

<success_criteria>
- [ ] Loop executes specified number of iterations or until complete
- [ ] Each iteration spawns fresh execute-plan subagent
- [ ] Learnings persist to ralph-progress.txt
- [ ] STATE.md and ROADMAP.md stay synchronized
- [ ] Completion criteria checked each iteration
- [ ] Final status reported with actionable next steps
</success_criteria>
