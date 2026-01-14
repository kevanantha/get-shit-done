# Ralph Loop Configuration Template

<template>
Ralph loop configuration for autonomous plan execution.

## Config File: `.planning/ralph-config.json`

```json
{
  "ralph": {
    "max_iterations": 10,
    "auto_commit": true,
    "verification_level": "standard",
    "scope": "milestone",
    "stop_on_checkpoint": false,
    "created": "YYYY-MM-DD HH:MM:SS UTC",
    "last_run": "YYYY-MM-DD HH:MM:SS UTC"
  }
}
```

## Progress File: `.planning/ralph-progress.txt`

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
Time: 2024-01-14 10:35:42 UTC
Executing: .planning/phases/01-foundation/01-02-PLAN.md

✓ Plan completed: 01-02-PLAN.md

### Plan 01-02 Completed
Files: src/db/client.ts, src/db/schema.ts
Commits: ghi789j, klm012n

Patterns:
- Database client is singleton with lazy initialization
- Schema files use Drizzle ORM conventions
- Migrations stored in src/db/migrations/

Gotchas:
- Connection pool must be properly closed in tests
- Schema changes need migration generation via `npm run db:generate`

Decisions:
- Chose Drizzle over Prisma for this project (lighter weight, closer to SQL)
- Implemented connection retry logic with exponential backoff

---

<promise>COMPLETE</promise>

All plans in Phase 1 executed successfully.
```
</template>

<configuration_options>

## max_iterations
- **Type:** number
- **Default:** 10
- **Description:** Maximum number of loop iterations before stopping
- **Range:** 1-100
- **Recommendation:** 5-10 for single phase, 20-50 for full milestone

## auto_commit
- **Type:** boolean
- **Default:** true
- **Description:** Automatically commit after each task completion
- **Note:** When false, commits are deferred until plan completion

## verification_level
- **Type:** string
- **Default:** "standard"
- **Options:**
  - `basic` - File creation and git commit only
  - `standard` - Includes build and smoke tests
  - `comprehensive` - Full test suite, linter, type checking
- **Recommendation:** Use "standard" for most cases

## scope
- **Type:** string | number
- **Default:** "milestone"
- **Options:**
  - `"milestone"` - All phases in current milestone
  - `number` - Specific phase number (e.g., `1`, `2.1`)
- **Description:** Defines which plans to execute in loop

## stop_on_checkpoint
- **Type:** boolean
- **Default:** false
- **Description:** Whether to stop loop when checkpoint encountered
- **Note:** When false, orchestrator handles checkpoint and continues
- **Note:** When true, loop pauses for user interaction

</configuration_options>

<initialization>
To initialize Ralph loop configuration:

1. **Via command:**
   ```
   /gsd:ralph-loop [phase-number]
   ```
   Command will prompt for configuration options.

2. **Manual creation:**
   ```bash
   cat > .planning/ralph-config.json << 'EOF'
   {
     "ralph": {
       "max_iterations": 10,
       "auto_commit": true,
       "verification_level": "standard",
       "scope": "milestone",
       "stop_on_checkpoint": false,
       "created": "$(date -u +"%Y-%m-%d %H:%M:%S UTC")",
       "last_run": "$(date -u +"%Y-%m-%d %H:%M:%S UTC")"
     }
   }
   EOF
   ```

3. **Progress file:**
   ```bash
   cat > .planning/ralph-progress.txt << 'EOF'
   # Ralph Loop Progress
   # Each iteration appends learnings, patterns, and context
   
   Started: $(date -u +"%Y-%m-%d %H:%M:%S UTC")
   Scope: [Describe scope here]
   Max Iterations: 10
   
   ---
   EOF
   ```
</initialization>

<usage_examples>

## Example 1: Execute single phase with Ralph loop

```bash
# Start Ralph loop for phase 1
/gsd:ralph-loop 1

# Ralph will:
# - Execute all plans in phase 1 sequentially
# - Each plan gets fresh subagent context
# - Learnings accumulate in ralph-progress.txt
# - Loop until all phase 1 plans complete
```

## Example 2: Execute entire milestone

```bash
# Start Ralph loop for all phases
/gsd:ralph-loop

# Ralph will:
# - Execute all incomplete plans across all phases
# - Continue until milestone complete or max iterations
# - Track progress across phases
```

## Example 3: Custom configuration

```json
{
  "ralph": {
    "max_iterations": 50,
    "auto_commit": true,
    "verification_level": "comprehensive",
    "scope": "milestone",
    "stop_on_checkpoint": true
  }
}
```

```bash
/gsd:ralph-loop

# With this config:
# - Can run up to 50 iterations (good for large milestones)
# - Full verification (tests, linting, type checking)
# - Stops at checkpoints for user input
```

</usage_examples>

<best_practices>

**Start small:**
- First Ralph loop: single phase, max_iterations: 5
- Learn the pattern, then scale up

**Monitor progress:**
- Check ralph-progress.txt periodically
- Look for repeated patterns/gotchas
- These indicate opportunities to improve plans

**Use appropriate verification:**
- Development: "basic" (fast iteration)
- Staging: "standard" (balanced)
- Production-ready: "comprehensive" (high confidence)

**Set realistic max_iterations:**
- Single phase (3-5 plans): 5-10 iterations
- Multiple phases (10-15 plans): 20-30 iterations
- Full milestone (20+ plans): 40-50 iterations

**Handle checkpoints wisely:**
- stop_on_checkpoint: false (default) - Good for autonomous "walk away" mode
- stop_on_checkpoint: true - Good when architectural decisions needed

**Review learnings:**
- After loop completes, review ralph-progress.txt
- Extract patterns to add to codebase docs
- Identify gotchas to prevent in future phases

</best_practices>
