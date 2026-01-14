---
name: gsd:ralph-loop
description: Execute plans in autonomous loop until completion criteria met
argument-hint: "[phase-number]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - TodoWrite
  - AskUserQuestion
---

<objective>
Execute plans in an autonomous Ralph loop pattern until all plans in a phase (or entire milestone) are complete.

The Ralph loop runs GSD's execute-plan workflow repeatedly with fresh context each iteration, persisting state via git history, progress file, and planning artifacts. Each iteration picks the next incomplete plan, executes it, verifies completion, and loops until all plans pass or max iterations reached.

Inspired by Geoffrey Huntley's Ralph pattern and optimized for GSD's context engineering approach.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/ralph-loop.md
@~/.claude/get-shit-done/templates/ralph-config.md
</execution_context>

<context>
Phase (optional): $ARGUMENTS

@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/ralph-progress.txt (if exists)
</context>

<process>
1. **Initialize Ralph loop**
   - Check if ralph-config.json exists, create if not
   - Parse max iterations (default: 10)
   - Check for existing ralph-progress.txt

2. **Determine scope**
   - If phase number provided: loop over that phase only
   - If no argument: loop over all incomplete phases in current milestone
   - Confirm scope with user

3. **Configure loop**
   - Present AskUserQuestion for:
     * Max iterations (default: 10)
     * Auto-commit mode (yes/no, default: yes)
     * Verification level (basic/standard/comprehensive, default: standard)
   - Save to .planning/ralph-config.json

4. **Execute loop**
   - Delegate to ralph-loop workflow
   - Workflow handles:
     * Finding next incomplete plan
     * Spawning fresh execute-plan subagent
     * Capturing results to ralph-progress.txt
     * Updating STATE.md and ROADMAP.md
     * Checking completion criteria
     * Looping until complete or max iterations

5. **Report completion**
   - Show summary of executed plans
   - Show ralph-progress.txt learnings
   - Update STATE.md with loop results
   - Offer next steps
</process>

<ralph_pattern>
**Core Ralph Loop Pattern:**

Each iteration is a fresh subagent with clean context. Memory persists through:
- Git history (commits from previous iterations)
- ralph-progress.txt (learnings and patterns discovered)
- ROADMAP.md and STATE.md (which plans are complete)

The loop:
1. Identifies next incomplete plan (from ROADMAP)
2. Spawns fresh execute-plan subagent for that plan
3. Subagent executes, commits, creates SUMMARY
4. Captures learnings to ralph-progress.txt
5. Updates STATE.md with iteration results
6. Checks if all plans complete
7. If not complete and iterations remain, loop to step 1
8. If complete, output <promise>COMPLETE</promise> and exit
</ralph_pattern>

<completion_criteria>
Loop exits when:
- All plans in scope have SUMMARY.md files, OR
- Max iterations reached, OR
- User cancels, OR
- Critical error prevents continuation

Output `<promise>COMPLETE</promise>` only when all plans successfully executed.
</completion_criteria>

<stop_conditions>
- All targeted plans have SUMMARY.md
- Max iterations exceeded
- Build/test failures in 3 consecutive iterations
- User intervention required (e.g., architectural decisions)
</stop_conditions>

<success_criteria>
- [ ] Ralph config created or loaded
- [ ] Loop scope confirmed with user
- [ ] Ralph loop workflow executed
- [ ] Progress tracked in ralph-progress.txt
- [ ] Final status reported to user
- [ ] Next steps presented
</success_criteria>
