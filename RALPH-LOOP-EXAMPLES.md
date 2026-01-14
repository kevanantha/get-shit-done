# Ralph Loop Usage Examples

This document provides practical examples of using the Ralph loop feature in GSD.

## Example 1: Simple Phase Execution

### Scenario
You have Phase 1 with 3 plans that build on each other. You want autonomous execution with learnings captured.

### Setup
```bash
# Your project structure
.planning/
├── ROADMAP.md
├── STATE.md
└── phases/
    └── 01-foundation/
        ├── 01-01-PLAN.md  # Database setup
        ├── 01-02-PLAN.md  # Schema creation
        └── 01-03-PLAN.md  # Seed data
```

### Execution
```
/gsd:ralph-loop 1
```

### What Happens
1. **Configuration Prompt:**
   - Max iterations: 5 (default: 10)
   - Auto commit: yes
   - Verification: standard
   - Scope: phase 1

2. **Iteration 1:**
   - Executes 01-01-PLAN.md (database setup)
   - Creates 01-01-SUMMARY.md
   - Learns: "PostgreSQL client uses connection pooling"
   - Appends to ralph-progress.txt

3. **Iteration 2:**
   - Loads learnings from iteration 1
   - Executes 01-02-PLAN.md (schema creation)
   - Creates 01-02-SUMMARY.md
   - Learns: "Migrations use db:generate command"
   - Appends to ralph-progress.txt

4. **Iteration 3:**
   - Loads learnings from iterations 1-2
   - Executes 01-03-PLAN.md (seed data)
   - Creates 01-03-SUMMARY.md
   - Learns: "Seed script runs via npm run db:seed"
   - Appends to ralph-progress.txt

5. **Completion:**
   ```
   <promise>COMPLETE</promise>
   All plans in Phase 1 executed successfully.
   ```

### Result
```bash
# Check learnings
cat .planning/ralph-progress.txt

# Shows accumulated knowledge:
# - Connection pooling patterns
# - Migration workflow
# - Seed data approach
```

---

## Example 2: Full Milestone with Multiple Phases

### Scenario
MVP with 4 phases, 10 total plans. You want overnight autonomous execution.

### Setup
```bash
.planning/
├── phases/
    ├── 01-foundation/      # 2 plans
    ├── 02-auth/            # 3 plans
    ├── 03-core-features/   # 4 plans
    └── 04-polish/          # 1 plan
```

### Execution
```
/gsd:ralph-loop
```

### Configuration
- Max iterations: 15 (buffer for 10 plans)
- Auto commit: yes
- Verification: comprehensive (full tests)
- Scope: milestone
- Stop on checkpoint: no (autonomous)

### What Happens
Ralph executes all 10 plans sequentially:
- Phase 1: Plans 01-01, 01-02
- Phase 2: Plans 02-01, 02-02, 02-03
- Phase 3: Plans 03-01, 03-02, 03-03, 03-04
- Phase 4: Plan 04-01

Each iteration learns from previous ones. By Phase 4, Ralph knows:
- Auth patterns from Phase 2
- API conventions from Phase 3
- Testing patterns from all phases

### Progress Tracking
```bash
# Monitor while running
tail -f .planning/ralph-progress.txt

# Check status anytime
/gsd:progress
```

---

## Example 3: Resume After Interruption

### Scenario
Ralph loop was interrupted at iteration 5 of 10. You want to resume.

### Resume
```
/gsd:ralph-loop
```

### What Happens
1. Ralph detects existing ralph-progress.txt
2. Reads previous learnings
3. Checks ROADMAP.md for incomplete plans
4. Continues from where it left off

```
---
## New Session: 2024-01-15 08:00:00 UTC

## Iteration 6 / 10
Executing: .planning/phases/02-auth/02-03-PLAN.md
...
```

All learnings from iterations 1-5 are preserved and loaded.

---

## Example 4: High-Quality Release Build

### Scenario
Final pre-release verification. You want comprehensive verification on all plans.

### Configuration
```json
{
  "ralph": {
    "max_iterations": 20,
    "auto_commit": true,
    "verification_level": "comprehensive",
    "scope": "milestone",
    "stop_on_checkpoint": false
  }
}
```

### Execution
```
/gsd:ralph-loop
```

### Verification Includes
- Full test suite passes
- Linter passes
- Type checking passes
- Build succeeds
- No new warnings

If any verification fails, Ralph logs and continues to next plan. After completion, review ralph-progress.txt for any failures.

---

## Example 5: Debugging with Ralph Loop

### Scenario
Test suite keeps failing. You want Ralph to iterate until all tests pass.

### Configuration
```json
{
  "ralph": {
    "max_iterations": 10,
    "auto_commit": true,
    "verification_level": "comprehensive",
    "scope": 3,
    "stop_on_checkpoint": false
  }
}
```

### Execution
```
/gsd:ralph-loop 3
```

### What Happens
1. **Iteration 1:** Executes plan, tests fail, logs failure
2. **Iteration 2:** Reads failure from ralph-progress.txt, fixes issue, tests pass
3. **Iteration 3:** Continues to next plan

Ralph learns from test failures and applies fixes in subsequent iterations.

---

## Example 6: With Checkpoints (Interactive Mode)

### Scenario
Plans have architectural decisions that need human input.

### Configuration
```json
{
  "ralph": {
    "max_iterations": 10,
    "auto_commit": true,
    "verification_level": "standard",
    "scope": 2,
    "stop_on_checkpoint": true
  }
}
```

### Execution
```
/gsd:ralph-loop 2
```

### What Happens
1. **Iteration 1:** Executes until checkpoint
2. **Pause:** Presents checkpoint to user
   ```
   ## CHECKPOINT: Architectural Decision
   Choose authentication library:
   1. jose (JWT, modern)
   2. jsonwebtoken (mature, widely used)
   ```
3. **User Input:** Selects option 1
4. **Resume:** Ralph continues with choice
5. **Iteration 2:** Proceeds to next plan

Loop pauses at each checkpoint but remains autonomous between them.

---

## Example 7: Learning Patterns Across Iterations

### Real Example from ralph-progress.txt

```
## Iteration 1 / 10
Executing: .planning/phases/01-db/01-01-PLAN.md
✓ Plan completed

Patterns:
- Database client is singleton with lazy init
- Connection string from process.env.DATABASE_URL

## Iteration 2 / 10
Executing: .planning/phases/01-db/01-02-PLAN.md
✓ Plan completed

Patterns:
- Migrations use db:generate and db:migrate
- Schema changes require new migration

## Iteration 3 / 10
Executing: .planning/phases/02-api/02-01-PLAN.md
✓ Plan completed

Patterns:
- API routes follow /api/v1/{resource} pattern
- All routes use database client from iteration 1 (singleton)
- Queries use prepared statements (learned from db setup)

Gotchas:
- Must call db.close() in tests or connections leak
```

**Notice:** By iteration 3, Ralph applies patterns learned in iterations 1-2 (singleton client, prepared statements).

---

## Best Practices from Examples

**1. Start small, scale up**
```bash
/gsd:ralph-loop 1     # First time
/gsd:ralph-loop 2     # Once confident
/gsd:ralph-loop       # Eventually
```

**2. Set max_iterations with buffer**
- Single phase (3 plans): max_iterations: 5
- Multiple phases (10 plans): max_iterations: 15
- Full milestone (20 plans): max_iterations: 30

**3. Use appropriate verification**
- Development: "basic" (fast)
- Pre-merge: "standard" (balanced)
- Release: "comprehensive" (thorough)

**4. Monitor progress**
```bash
tail -f .planning/ralph-progress.txt
```

**5. Review learnings after completion**
```bash
cat .planning/ralph-progress.txt | grep "Patterns:"
cat .planning/ralph-progress.txt | grep "Gotchas:"
```

Extract these to document in codebase or AGENTS.md files.

---

## Troubleshooting

**Loop stuck on same plan:**
```bash
# Check what's failing
cat .planning/ralph-progress.txt | tail -50

# Fix manually if needed
/gsd:execute-plan .planning/phases/XX-name/XX-YY-PLAN.md

# Resume Ralph loop
/gsd:ralph-loop
```

**Max iterations reached:**
```bash
# Increase limit
# Edit .planning/ralph-config.json
{
  "ralph": {
    "max_iterations": 20,  # was 10
    ...
  }
}

# Resume
/gsd:ralph-loop
```

**Tests keep failing:**
```bash
# Check what's been tried
cat .planning/ralph-progress.txt | grep -A 5 "✗"

# Lower verification temporarily
# Edit .planning/ralph-config.json
{
  "ralph": {
    "verification_level": "basic",  # skip tests
    ...
  }
}

# Fix tests manually later
```

---

## Comparison: execute-phase vs ralph-loop

| Aspect | execute-phase | ralph-loop |
|--------|--------------|------------|
| Execution | Parallel | Sequential |
| Speed | Faster | Slower |
| Learning | Per-plan only | Across iterations |
| Best for | Independent plans | Dependent plans |
| Context | Fresh per plan | Fresh per iteration |
| Use case | "Just ship it" | "Learn and improve" |

**Choose execute-phase when:**
- Plans are independent
- Speed is priority
- No cross-plan patterns

**Choose ralph-loop when:**
- Plans build on each other
- Want accumulated learnings
- Debugging complex issues
- Iterative refinement needed
