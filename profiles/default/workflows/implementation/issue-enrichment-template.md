# Issue Enrichment Template: TOKEN-EFFICIENT WORKFLOW

This template provides standardized formats for enriching issues with "TOKEN-EFFICIENT WORKFLOW" comments before spawning agents. Each issue must be enriched this way to ensure agents have clear scope, file locations, and approach steps.

---

## Core Template

Every issue enrichment comment should follow this structure:

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- [absolute/path/to/file.ext]:[line_number or ~approx_line] ([what change needed])
- [another file if applicable]

APPROACH:
1. [Concrete first step with file reference]
2. [Next step with expected outcome]
3. [Test command to verify]
4. [Commit if tests pass]

DO NOT READ: [list of packages/directories/files NOT relevant to this task]

EXPECTED: [N] tool calls, <[X]K tokens
```

### When Adding This Comment

Use `bd comment <ISSUE_ID>` to add the comment:

```bash
bd comment bd-123 "$(cat <<'EOF'
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/handlers.go:41 (remove global 'var config')
- src/monitor.go:30 (add config field)
- src/main.go:100 (pass config to NewMonitor)

APPROACH:
1. Read monitor.go lines 25-35, identify Monitor struct
2. Add 'config *core.Config' field to struct
3. Update NewMonitor() signature to accept config param
4. Read handlers.go, find all 'config.' references
5. Replace with 'm.config.' in handler functions
6. Update main.go NewMonitor() call to pass config
7. Run: go test ./...
8. Commit if all tests pass

DO NOT READ: core/, pdfprocessor/, imagegen/, test files

EXPECTED: 15-20 tool calls, <50K tokens
EOF
)"
```

---

## Template Patterns by Issue Type

### Pattern 1: Single-File Bug Fix

**Use this when:**
- Issue affects only 1 file
- Clear acceptance criteria
- Expected <20K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/utils/validation.go:[start-end] (fix validation logic)

APPROACH:
1. Read validation.go:[start-end] to understand current logic
2. Identify the bug (describe it specifically)
3. Implement fix
4. Run: go test ./utils
5. Commit: "fix: correct validation logic in [function]"

DO NOT READ: Other packages

EXPECTED: 5-10 tool calls, <20K tokens
```

---

### Pattern 2: Small Feature (2-3 Files)

**Use this when:**
- Issue affects 2-3 files maximum
- Feature scope is clear
- Expected <50K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/models/user.go:[line] (add new field/method)
- src/handlers/user.go:[line] (update endpoint)
- src/routes.go:[line] (add/update route if needed)

APPROACH:
1. Read models/user.go:[line], understand User struct
2. Add new field/method to User
3. Add tests in models/user_test.go (inline if new file)
4. Read handlers/user.go:[line], find UserHandler function
5. Implement feature in handler
6. Update routes.go if route changes needed
7. Run: go test ./... -run TestUser
8. Commit: "feat: add [feature name] to user model"

DO NOT READ: Other handlers, middleware, auth packages

EXPECTED: 15-25 tool calls, <50K tokens
```

---

### Pattern 3: Global State Elimination (Multi-File Refactor)

**Use this when:**
- Breaking down a large global state removal
- Targeting 3-5 files
- Each sub-task expected <50K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/handlers.go:[line] (remove global 'var X')
- src/service.go:[line] (add X field to Service struct)
- src/main.go:[line] (inject X via NewService)

APPROACH:
1. Grep: Search for 'var X' pattern in handlers.go
2. Note all references to global X
3. Read service.go, find Service struct
4. Add 'X [Type]' field to struct
5. Update NewService() func signature
6. In handlers.go, replace all 'X.' with 's.X.'
7. Update main.go NewService() call
8. Run: go test ./...
9. Commit: "refactor: inject X dependency in Service"

DO NOT READ: Unrelated packages, full file reads - use Grep to find references

EXPECTED: 12-18 tool calls, <50K tokens
```

---

### Pattern 4: Database Migration / Model Addition

**Use this when:**
- Adding new database model
- Creating migration
- Expected 20-40K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/models/post.go (NEW: create Post model)
- migrations/[timestamp]-create-posts.js (NEW: create migration)
- src/models/index.js (add Post model export)
- src/handlers/posts.go (NEW: create handlers)

APPROACH:
1. Reference models/user.go for model structure pattern
2. Create models/post.go with Post model, validations, associations
3. Create new migration file for posts table
4. Update models/index.js to export Post
5. Create handlers/posts.go with CRUD endpoints
6. Add tests in models/post_test.go
7. Run migrations: npm run migrate
8. Run: npm test -- --grep "Post"
9. Commit: "feat: add Post model and handlers"

DO NOT READ: Unrelated models, full handler files - reference existing patterns

EXPECTED: 20-30 tool calls, <60K tokens
```

---

### Pattern 5: API Endpoint Addition

**Use this when:**
- Adding 1-2 new API endpoints
- Using existing models
- Expected 20-40K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/handlers/users.go:[line] (add new endpoint handler)
- src/routes.js:[line] (add route definition)
- src/handlers/users_test.go (add endpoint tests)

APPROACH:
1. Reference existing endpoint in handlers/users.go
2. Add new handler function following same pattern
3. Add route definition in routes.js
4. Write 2-3 tests for new endpoint
5. Run: npm test -- --grep "new-endpoint"
6. Run full test: npm test
7. Commit: "feat: add [endpoint name] endpoint"

DO NOT READ: Other handlers, authentication, middleware

EXPECTED: 10-15 tool calls, <40K tokens
```

---

### Pattern 6: UI Component Addition

**Use this when:**
- Adding new React/Vue component
- Simple component (not complex logic)
- Expected 25-50K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/components/NewComponent.jsx (NEW: create component)
- src/components/NewComponent.module.css (NEW: add styles)
- src/pages/ParentPage.jsx:[line] (integrate component)
- src/components/__tests__/NewComponent.test.jsx (add tests)

APPROACH:
1. Reference src/components/ExistingComponent.jsx for pattern
2. Create NewComponent.jsx with props, state, render logic
3. Create CSS module with base styles
4. Add 2-3 focused tests
5. Integrate component into ParentPage.jsx
6. Run: npm test -- NewComponent
7. Run: npm run build (check for errors)
8. Commit: "feat: add NewComponent"

DO NOT READ: Other components, full component library, styling files unrelated to this component

EXPECTED: 15-25 tool calls, <50K tokens
```

---

### Pattern 7: Test Gap Coverage

**Use this when:**
- Adding missing test coverage
- No implementation changes
- Expected 15-30K tokens

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/utils/__tests__/helpers.test.js (add gap tests)

APPROACH:
1. Review existing helpers tests to find gaps
2. Identify 3-5 uncovered edge cases
3. Write focused tests for each case
4. Run: npm test -- helpers
5. Verify all new tests pass
6. Commit: "test: add gap coverage for edge cases in helpers"

DO NOT READ: Implementation files, other tests unrelated to this util

EXPECTED: 8-12 tool calls, <25K tokens
```

---

## Token Budget Guidelines (By Issue Type)

| Issue Type | Files | Expected Tokens | Max Acceptable | Red Flag |
|------------|-------|-----------------|----------------|----------|
| Single-file bug fix | 1 | 10-20K | 40K | >60K |
| Small feature | 2-3 | 20-40K | 70K | >100K |
| Database model | 3-4 | 20-40K | 70K | >100K |
| API endpoint | 2-3 | 20-40K | 70K | >100K |
| UI component | 3-4 | 25-50K | 80K | >120K |
| Global state removal | 3-5 | 30-50K | 80K | >120K |
| Test gap coverage | 1 | 15-25K | 50K | >70K |
| Refactor (multiple) | 4-6 | 40-70K | 120K | >150K |

---

## Enrichment Checklist

Before adding enrichment comment, verify:

- [ ] **FILES TO MODIFY:** Are these exact file paths? Can use Glob pattern if needed
- [ ] **Line numbers:** Are they accurate or approximations (~)?
- [ ] **APPROACH:** Are steps concrete and specific? (Not just "implement feature")
- [ ] **DO NOT READ:** Does this list prevent scope creep?
- [ ] **Token budget:** Is it realistic for the approach? (Cross-check table above)
- [ ] **Tool call estimate:** Does N tool calls align with K tokens? (Roughly 1-2K per complex tool call)

---

## Common Enrichment Mistakes (To Avoid)

### ❌ Too Vague
```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/

APPROACH:
1. Add feature

DO NOT READ: (empty)

EXPECTED: Unknown tool calls, unknown tokens
```

**Problem:** No specific files, no concrete steps, scope creep likely

### ✅ Correct - Specific
```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/handlers/user.go:150-170 (add new endpoint)
- src/routes.js:20 (add route)

APPROACH:
1. Read handlers/user.go lines 150-170
2. Add POST /users/:id/verify endpoint
3. Update routes.js to register route
4. Run: npm test -- user-handlers
5. Commit

DO NOT READ: auth/, middleware/, other handlers

EXPECTED: 10-12 tool calls, <40K tokens
```

---

### ❌ Too Broad
```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/handlers/ (entire directory)
- src/models/ (entire directory)
- src/routes.js

APPROACH:
1. Refactor all handlers and models

DO NOT READ: (none specified)

EXPECTED: ???
```

**Problem:** Scope too large, should be broken into sub-tasks

### ✅ Correct - Focused
```markdown
(Break this into 3-4 separate issues, each with specific files and <50K token budget)
```

---

## Examples from Real Issues

### Example 1: Extract Helper Function

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/utils/string.js:45-60 (extract formatPhoneNumber)
- src/utils/__tests__/string.test.js (add tests)

APPROACH:
1. Read string.js lines 45-60 (current phone formatting logic)
2. Extract logic into formatPhoneNumber(phone: string): string
3. Replace inline logic with function call
4. Add 3 tests for edge cases
5. Run: npm test -- string
6. Commit: "refactor: extract formatPhoneNumber utility"

DO NOT READ: Other utils, models

EXPECTED: 6-8 tool calls, <20K tokens
```

### Example 2: Add Model Field & Validation

```markdown
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- src/models/user.js:50 (add verified_email field)
- src/models/user.js:70 (add validation)
- migrations/20250115-add-verified-email.js (NEW)
- src/models/user_test.js (add validation test)

APPROACH:
1. Read user.js to understand model structure
2. Create migration file to add verified_email column
3. Add verified_email field to User model (default: false)
4. Add validation: email must be verified for certain operations
5. Write test: validate email required
6. Run: npm run migrate && npm test -- user
7. Commit: "feat: add email verification tracking to User model"

DO NOT READ: Auth module, other models

EXPECTED: 12-16 tool calls, <40K tokens
```

---

## Quick Reference

**When enriching an issue:**

1. Use the pattern that matches the issue type (above)
2. Fill in specific file paths and line numbers
3. Make approach steps concrete (file references, function names)
4. Set realistic token budget (use table)
5. Add it via: `bd comment <ID> "$(cat <<'EOF'...EOF)"`

**Key principles:**
- Specific > vague
- Concrete steps > abstract descriptions
- With file paths > without
- With scope boundaries > without
- Realistic tokens > inflated estimates
