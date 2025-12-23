# Scope Validation: Red Flags and Issue Breakdown

This guide helps identify when an issue is too large or vague to spawn an agent directly. When red flags are detected, the issue must be broken down into smaller sub-tasks with dependencies.

---

## Red Flag Detection Checklist

Before spawning an agent, check if the issue has ANY of these red flags. **If ANY are true, the issue is BLOCKED and requires breakdown.**

### Red Flag 1: Vague Description (<50 characters)

**Examples of too-vague descriptions:**
- "Refactor code"
- "Clean up handlers.go"
- "Add missing tests"
- "Improve performance"

**Why it's a problem:**
- Agents spend 40-60K tokens exploring to understand what "refactor code" means
- No clear acceptance criteria
- Open-ended scope leads to failures

**Solution:**
```bash
bd comment <ID> "❌ BLOCKED - Description too vague (<50 chars)"
bd comment <ID> "Please specify: Which code? What's the goal? What changes are expected?"
bd update <ID> --status blocked
```

---

### Red Flag 2: Too Many Files (>8 files)

**Examples:**
- "Eliminate global state across entire codebase"
- "Refactor all API endpoints"
- "Add TypeScript to entire frontend"

**Why it's a problem:**
- Scope affects 10+ files? It's actually 3-5 separate features
- Agents can't hold context for all dependencies
- Hard to test incrementally
- 10x more likely to fail

**Check:**
```bash
# Grep to count likely files
grep -r "var config" src/ | wc -l  # ~6 files? OK
grep -r "var config" src/ | wc -l  # ~15 files? TOO MANY - needs breakdown
```

**Solution:**
```bash
bd comment <ID> "❌ BLOCKED - Scope too broad (affects 15+ files)"
bd comment <ID> "This needs to be broken into 3-5 focused sub-tasks:"
bd comment <ID> "1. Extract config global (3-5 files)"
bd comment <ID> "2. Extract logging global (2-3 files)"
bd comment <ID> "3. etc."
bd update <ID> --status blocked
```

---

### Red Flag 3: Refactor Entire Subsystem

**Examples:**
- "Refactor entire authentication system"
- "Rewrite the database layer"
- "Restructure all components"
- "Migrate from X to Y across codebase"

**Why it's a problem:**
- Inherently touches 8+ files
- Multiple domains of change
- Hard to test safely
- Too much coupling

**Solution:**
Break into atomic pieces by layer/domain:
- Phase 1: Models (database layer)
- Phase 2: Controllers (API layer)
- Phase 3: Validation (shared utilities)
- Etc.

---

### Red Flag 4: No Clear Acceptance Criteria

**Examples:**
- "Clean up this file"
- "Make it better"
- "Optimize"
- "Improve code quality"

**Why it's a problem:**
- Agents don't know when they're done
- Can't verify success
- Leads to endless exploration

**Solution:**
```bash
bd comment <ID> "❌ BLOCKED - No clear acceptance criteria"
bd comment <ID> "Please define what 'done' looks like:"
bd comment <ID> "- Specific test name that should pass?"
bd comment <ID> "- Specific functions to refactor?"
bd comment <ID> "- Performance metrics (before/after)?"
bd update <ID> --status blocked
```

---

### Red Flag 5: Estimated >80K Tokens

**How to estimate:**
- Count concrete implementation steps (from issue description)
- Estimate tool calls per step (Grep=0.5K, Read=1K, Edit=1K, Bash=0.5K)
- Add planning/exploration overhead (20%)

**Formula:** `(tool_calls * avg_tokens) * 1.2 = estimated_tokens`

**Examples:**
- 15 tool calls × 1.5K avg = 22.5K tokens → ✅ OK
- 40 tool calls × 2K avg = 80K tokens → ⚠️ WARNING (at threshold)
- 60 tool calls × 2K avg = 144K tokens → ❌ TOO LARGE (needs breakdown)

**Solution:**
```bash
# If estimated >80K:
bd comment <ID> "⚠️  ESTIMATED >80K TOKENS - needs breakdown"
bd comment <ID> "Recommended split:"
bd comment <ID> "- Sub-task 1: [specific goal] - est. 40K"
bd comment <ID> "- Sub-task 2: [specific goal] - est. 35K"
bd update <ID> --status blocked
```

---

### Red Flag 6: Missing TOKEN-EFFICIENT WORKFLOW

**Check:**
```bash
# In pre-flight enrichment phase
WORKFLOW_EXISTS=$(bd show <ID> | grep -c "TOKEN-EFFICIENT WORKFLOW:" || echo "0")

if [ "$WORKFLOW_EXISTS" -eq "0" ]; then
    echo "⚠️  Issue <ID> lacks TOKEN-EFFICIENT WORKFLOW"
    # Enrich it (see issue-enrichment-template.md)
    # OR skip spawning until enriched
fi
```

---

## Issue Breakdown Strategy

When red flags detected, break the issue into 3-5 focused sub-tasks:

### Pattern 1: By File/Module

**Large issue affecting multiple modules:**

```
Parent Issue: "Refactor authentication module"

Sub-task 1: Extract config dependency from auth.go
  - Files: auth.go, config.js
  - Token est: 30K
  - Dependencies: none

Sub-task 2: Add dependency injection to auth handlers
  - Files: auth_handler.go, main.go
  - Token est: 35K
  - Dependencies: Sub-task 1

Sub-task 3: Update tests for new auth structure
  - Files: auth_test.go
  - Token est: 20K
  - Dependencies: Sub-task 2
```

### Pattern 2: By Layer

**Large issue affecting multiple layers:**

```
Parent Issue: "Add user roles system to entire app"

Sub-task 1: Database layer - create Role model & migrations
  - Files: models/role.js, migrations/
  - Token est: 30K
  - Dependencies: none

Sub-task 2: API layer - add role endpoints
  - Files: handlers/role.js, routes.js
  - Token est: 40K
  - Dependencies: Sub-task 1

Sub-task 3: UI layer - add role management page
  - Files: pages/admin/roles.jsx, components/
  - Token est: 45K
  - Dependencies: Sub-task 2

Sub-task 4: Testing - write integration tests
  - Files: __tests__/e2e/roles.test.js
  - Token est: 25K
  - Dependencies: Sub-task 3
```

### Pattern 3: By Feature Scope

**Large feature affecting multiple parts:**

```
Parent Issue: "Add email notifications to platform"

Sub-task 1: Create notification model & service
  - Files: models/notification.js, services/notifier.js
  - Token est: 35K
  - Dependencies: none

Sub-task 2: Add email template system
  - Files: templates/emails/, services/emailer.js
  - Token est: 30K
  - Dependencies: none (can run in parallel with Sub-task 1)

Sub-task 3: Wire notifications into user events
  - Files: models/user.js, hooks/, main.js
  - Token est: 40K
  - Dependencies: Sub-task 1

Sub-task 4: Create admin notification controls
  - Files: pages/admin/notifications.jsx, handlers/
  - Token est: 35K
  - Dependencies: Sub-task 3
```

### Creating Sub-Tasks with Dependencies

```bash
# Create sub-task 1 (no dependencies)
bd create "bd-456: Sub-task 1 - Extract config" \
  --label "parent:bd-123" \
  --priority 2 \
  --estimate 30k

# Create sub-task 2 (depends on sub-task 1)
bd create "bd-457: Sub-task 2 - Inject dependencies" \
  --label "parent:bd-123" \
  --priority 2 \
  --estimate 35k

# Add dependency
bd dep add bd-457 bd-456 --type "blocked-by"

# Close parent issue
bd close bd-123 --reason "Split into sub-tasks: bd-456, bd-457"
```

---

## Token Budget Guidelines by Task Type

Use these benchmarks when estimating token consumption:

### Bug Fixes
| Scope | Files | Steps | Est. Tokens | Max Acceptable |
|-------|-------|-------|------------|----------------|
| Single file, obvious fix | 1 | 5-7 | 10-20K | 40K |
| Single file, complex fix | 1 | 8-12 | 20-30K | 50K |
| Multi-file fix | 2-3 | 12-15 | 30-40K | 70K |
| System-level fix | 4+ | 15+ | 50K+ | 100K+ |

### Features
| Scope | Files | Steps | Est. Tokens | Max Acceptable |
|-------|-------|-------|------------|----------------|
| Single model/function | 2 | 8-10 | 20-30K | 50K |
| Single layer (API/UI/DB) | 3-5 | 12-15 | 35-50K | 80K |
| Multi-layer feature | 6-8 | 20-25 | 60-80K | 120K |
| Full feature + tests | 8+ | 25+ | 80K+ | 150K+ |

### Refactoring
| Scope | Files | Steps | Est. Tokens | Max Acceptable |
|-------|-------|-------|------------|----------------|
| Single function | 1-2 | 5-8 | 15-25K | 50K |
| Single module | 2-4 | 10-15 | 30-50K | 80K |
| Multi-module | 5+ | 15-20 | 60K+ | 120K+ |
| System-wide | 8+ | 20+ | 100K+ | ❌ TOO LARGE |

**Rule of thumb:**
- <40K tokens = Safe, agent will likely succeed
- 40-80K tokens = Acceptable, watch carefully
- 80-120K tokens = Risk zone, should be broken down
- 120K+ tokens = ❌ DO NOT SPAWN - must break down

---

## Breakdown Template

When creating sub-tasks, use this checklist:

### For Each Sub-Task:
- [ ] **Clear title:** "Extract X from Y" (not "Refactor Y")
- [ ] **Files specified:** Exactly which files? (not "handlers/")
- [ ] **Scope bounded:** Will touch ≤6 files?
- [ ] **Dependencies listed:** Depends on (none) or (sub-task N)?
- [ ] **Token estimate:** <80K?
- [ ] **Acceptance criteria:** Clear way to verify done?
- [ ] **TOKEN-EFFICIENT WORKFLOW:** Will be enriched before spawn?

### Parent Issue Update:
```bash
bd comment bd-123 "Split into focused sub-tasks:

Sub-task 1 (bd-456): Extract config dependency
  - Token est: 30K
  - Dependencies: none
  - Priority: 1

Sub-task 2 (bd-457): Update handlers for injection
  - Token est: 35K
  - Dependencies: bd-456
  - Priority: 2

Sub-task 3 (bd-458): Update tests
  - Token est: 25K
  - Dependencies: bd-457
  - Priority: 3
"

bd close bd-123 --reason "Broken into focused sub-tasks"
```

---

## Red Flag Response Flowchart

```
Is issue ready to spawn?
  ├─ Description <50 chars? → ❌ BLOCKED - Ask for clarification
  ├─ Affects >8 files? → ❌ BLOCKED - Break into sub-tasks
  ├─ "Refactor entire X"? → ❌ BLOCKED - Break by layer/domain
  ├─ No acceptance criteria? → ❌ BLOCKED - Ask for specifics
  ├─ Est. >80K tokens? → ❌ BLOCKED - Break into smaller tasks
  ├─ Missing TOKEN-EFFICIENT WORKFLOW? → ⚠️ ENRICH - Add workflow comment
  └─ All checks pass? → ✅ READY - Spawn agent
```

---

## Common Breakdown Examples

### Example 1: "Eliminate all global state"

**Original (TOO LARGE):**
```
Issue: Eliminate all global state in handlers.go
Scope: 6 globals across 2,497 lines
Est. Tokens: 1.8M
Status: ❌ BLOCKED
```

**Broken Down (GOOD):**
```
Issue 1: Extract config global (bd-456)
  - Files: handlers.go:41, monitor.go:30, main.go:100
  - Est: 30K tokens
  - Dependencies: none

Issue 2: Extract logging global (bd-457)
  - Files: handlers.go:58, logger setup
  - Est: 25K tokens
  - Dependencies: bd-456

Issue 3: Extract metrics global (bd-458)
  - Files: handlers.go:60, metrics setup
  - Est: 25K tokens
  - Dependencies: bd-457

Issue 4: Extract dashboard state (bd-459)
  - Files: handlers.go:62-63, dashboard wiring
  - Est: 25K tokens
  - Dependencies: bd-458

Total: 105K tokens (vs 1.8M) - ✅ MUCH BETTER
```

---

### Example 2: "Refactor entire authentication system"

**Broken Down by Layer:**
```
Database Layer (bd-460):
  - Create User, Role, Permission models
  - Create migrations
  - Est: 35K tokens
  - Dependencies: none

API Layer (bd-461):
  - Create auth endpoints
  - Implement JWT
  - Update user endpoints with auth
  - Est: 50K tokens
  - Dependencies: bd-460

UI Layer (bd-462):
  - Create login/logout pages
  - Add auth context
  - Protect routes
  - Est: 45K tokens
  - Dependencies: bd-461

Tests (bd-463):
  - Write auth flow tests
  - Write integration tests
  - Est: 30K tokens
  - Dependencies: bd-462
```

---

## When to Skip Breakdown

You DON'T need to break down if:

- ✅ Description is specific (>50 chars, clear goal)
- ✅ Affects ≤6 files
- ✅ Has clear acceptance criteria
- ✅ Estimated <80K tokens
- ✅ Has TOKEN-EFFICIENT WORKFLOW comment

Example of issue that doesn't need breakdown:
```
Title: Add email validation to User model
Description: Add isValidEmail() method to User model, validate emails on create/update
Files: models/user.js, migrations/
Est: 30K tokens
Status: ✅ READY - No breakdown needed
```

---

## Scope Validation Checklist (For Orchestrator)

Run this for each issue in pre-flight enrichment:

```bash
check_scope() {
    local id=$1
    local issue=$(bd show $id)

    # Check description length
    desc_len=$(echo "$issue" | head -1 | wc -c)
    if [ $desc_len -lt 50 ]; then
        echo "❌ $id: Description too vague"
        return 1
    fi

    # Check for red flags
    if echo "$issue" | grep -qE "refactor entire|eliminate all|rewrite"; then
        echo "❌ $id: Likely too large"
        return 1
    fi

    # Check for TOKEN-EFFICIENT WORKFLOW
    if ! echo "$issue" | grep -q "TOKEN-EFFICIENT WORKFLOW:"; then
        echo "⚠️  $id: Missing enrichment"
        return 2
    fi

    echo "✅ $id: Ready"
    return 0
}

# Usage
check_scope "bd-123"  # Returns 0 if ready, 1 if blocked, 2 if needs enrichment
```

---

## Key Principles

1. **Specific > Vague:** Always ask for specifics before spawning
2. **Smaller > Larger:** Four focused issues beat one large issue 10x
3. **Dependent > Coupled:** Use dependencies, not coupled changes
4. **Incremental > Big Bang:** Let agents commit after each step
5. **Before spawn > During execution:** Catch scope issues early

Remember: **5 minutes breaking down an issue saves 1M tokens of failed exploration.**
