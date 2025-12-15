# Create Beads Issues from Spec

This workflow auto-generates beads issues from spec.md with atomic design hierarchy and proper dependency relationships.

## Prerequisites

- Beads must be installed: `bd --version`
- Spec folder must exist at `agent-os/specs/[this-spec]/`
- `spec.md` must be complete
- Beads repository initialized in spec folder

## Overview

This workflow analyzes `spec.md` to:
1. Identify organisms (database layer, API layer, UI layer)
2. Extract required molecules from organism requirements
3. Identify atoms needed by molecules
4. Create hierarchical beads issues with parent-child relationships
5. Set blocking dependencies (atoms block molecules, molecules block organisms)
6. Result: `bd ready` shows only unblocked atoms initially

---

## Step 1: Initialize Beads in Spec Folder

```bash
# Navigate to spec folder
cd agent-os/specs/[this-spec]/

# Initialize beads (if not already done)
bd init --stealth

# Verify initialization
ls .beads/
# Should show: issues.jsonl, config.yaml, beads.db
```

**What this does:**
- Creates `.beads/` directory structure
- Initializes SQLite cache and JSONL source of truth
- Configures git merge driver for conflict resolution
- Uses `--stealth` mode (local only, no repo changes if unwanted)

---

## Step 2: Read and Analyze Spec

Read `spec.md` to identify:

1. **Organisms** - Look for major system components:
   - Database models and schema requirements
   - API endpoints and controller logic
   - UI components, forms, and pages

2. **Molecules** - Look for composition needs:
   - Helpers that combine multiple utilities
   - Services that orchestrate business logic
   - Complex validators or formatters

3. **Atoms** - Look for pure utilities:
   - Simple validators (email, phone, etc.)
   - Formatters (date, currency, etc.)
   - Constants and configuration values
   - Mathematical calculations

**Analysis Questions:**
- What database tables/models are needed?
- What API endpoints must be created?
- What UI components are described?
- What shared logic appears multiple times (candidates for molecules)?
- What pure functions would be useful (atoms)?

---

## Step 3: Create Parent Epic

Create the parent epic issue for the entire feature:

```bash
bd create "Feature: [Feature Name from spec]" \
  -t epic \
  -p 1 \
  --description "$(cat <<EOF
[1-2 sentence summary from spec.md]

Spec location: agent-os/specs/[this-spec]/spec.md
EOF
)"
```

**Capture the epic ID** for use as parent:
```bash
EPIC_ID=$(bd list --format=json | jq -r '.[0].id')
echo "Epic created: $EPIC_ID"
```

---

## Step 4: Create Organism Issues

Create one issue per organism layer. Use epic as parent.

### 4A: Database Layer Organism

```bash
bd create "[Organism] Database Layer" \
  -t organism \
  -p 2 \
  --parent "$EPIC_ID" \
  --tag database \
  --tag backend \
  --assignee database-layer-builder \
  --description "$(cat <<EOF
Implement database models, migrations, and associations.

**Models to create:**
- [Model 1: fields, validations]
- [Model 2: fields, validations]
- [Associations between models]

**Migrations:**
- [Migration 1 description]
- [Migration 2 description]

**Standards:** backend/models.md, backend/migrations.md, global/atomic-design.md
EOF
)"

DB_ORGANISM_ID=$(bd list --format=json | jq -r '.[0].id')
```

### 4B: API Layer Organism

```bash
bd create "[Organism] API Layer" \
  -t organism \
  -p 2 \
  --parent "$EPIC_ID" \
  --tag api \
  --tag backend \
  --assignee api-layer-builder \
  --description "$(cat <<EOF
Implement API endpoints, controllers, auth, and response formatting.

**Endpoints to create:**
- [GET /endpoint1 - description]
- [POST /endpoint2 - description]
- [PUT /endpoint3 - description]

**Auth requirements:**
- [Auth method, protected routes]

**Standards:** backend/api.md, global/atomic-design.md
EOF
)"

API_ORGANISM_ID=$(bd list --format=json | jq -r '.[0].id')

# Set dependency: API depends on database
bd dep add "$API_ORGANISM_ID" "$DB_ORGANISM_ID" --type blocks
```

### 4C: UI Component Organism

```bash
bd create "[Organism] UI Component Layer" \
  -t organism \
  -p 2 \
  --parent "$EPIC_ID" \
  --tag ui \
  --tag frontend \
  --assignee ui-component-builder \
  --description "$(cat <<EOF
Implement UI components, forms, pages, styles, and interactions.

**Components to create:**
- [Component 1: purpose, props]
- [Component 2: purpose, props]

**Pages:**
- [Page 1: route, components used]
- [Page 2: route, components used]

**Interactions:**
- [User flow 1]
- [User flow 2]

**Standards:** frontend/components.md, frontend/css.md, frontend/responsive.md, global/atomic-design.md
EOF
)"

UI_ORGANISM_ID=$(bd list --format=json | jq -r '.[0].id')

# Set dependency: UI depends on API
bd dep add "$UI_ORGANISM_ID" "$API_ORGANISM_ID" --type blocks
```

---

## Step 5: Create Molecule Issues (Children of Organisms)

For each organism, create child issues for required molecules.

### 5A: Database Molecules

```bash
# Example: User validation helpers (molecule)
bd create "[Molecule] User validation helpers" \
  -t molecule \
  -p 3 \
  --parent "$DB_ORGANISM_ID" \
  --tag database \
  --assignee molecule-composer \
  --description "Compose atom validators into user validation service (email + password + phone)"

DB_MOLECULE_1=$(bd list --format=json | jq -r '.[0].id')

# Set dependency: Database organism blocked by this molecule
bd dep add "$DB_ORGANISM_ID" "$DB_MOLECULE_1" --type blocks
```

### 5B: API Molecules

```bash
# Example: Auth middleware composer (molecule)
bd create "[Molecule] Auth middleware composer" \
  -t molecule \
  -p 3 \
  --parent "$API_ORGANISM_ID" \
  --tag api \
  --assignee molecule-composer \
  --description "Compose token validator + user loader into auth middleware chain"

API_MOLECULE_1=$(bd list --format=json | jq -r '.[0].id')

bd dep add "$API_ORGANISM_ID" "$API_MOLECULE_1" --type blocks
```

### 5C: UI Molecules

```bash
# Example: Login form component (molecule)
bd create "[Molecule] Login form component" \
  -t molecule \
  -p 3 \
  --parent "$UI_ORGANISM_ID" \
  --tag ui \
  --assignee molecule-composer \
  --description "Compose email input + password input + submit button into login form"

UI_MOLECULE_1=$(bd list --format=json | jq -r '.[0].id')

bd dep add "$UI_ORGANISM_ID" "$UI_MOLECULE_1" --type blocks
```

**Repeat** for each molecule identified in Step 2.

---

## Step 6: Create Atom Issues (Children of Molecules)

For each molecule, create child issues for required atoms.

### Example: Atoms for User Validation Molecule

```bash
# Atom 1: Email validator
bd create "[Atom] Email validator function" \
  -t atom \
  -p 4 \
  --parent "$DB_MOLECULE_1" \
  --tag validation \
  --assignee atom-writer \
  --description "Pure function: isValidEmail(email: string) => boolean. RFC 5322 compliant."

EMAIL_ATOM=$(bd list --format=json | jq -r '.[0].id')

# Block molecule until atom complete
bd dep add "$DB_MOLECULE_1" "$EMAIL_ATOM" --type blocks

# Atom 2: Password strength checker
bd create "[Atom] Password strength checker" \
  -t atom \
  -p 4 \
  --parent "$DB_MOLECULE_1" \
  --tag validation \
  --assignee atom-writer \
  --description "Pure function: validatePassword(pwd: string) => {valid: boolean, errors: string[]}. Min 8 chars, 1 uppercase, 1 number."

PWD_ATOM=$(bd list --format=json | jq -r '.[0].id')

bd dep add "$DB_MOLECULE_1" "$PWD_ATOM" --type blocks

# Atom 3: Phone formatter
bd create "[Atom] Phone number formatter" \
  -t atom \
  -p 4 \
  --parent "$DB_MOLECULE_1" \
  --tag formatting \
  --assignee atom-writer \
  --description "Pure function: formatPhone(phone: string) => string. Convert to (555) 123-4567 format."

PHONE_ATOM=$(bd list --format=json | jq -r '.[0].id')

bd dep add "$DB_MOLECULE_1" "$PHONE_ATOM" --type blocks
```

**Repeat** for atoms needed by each molecule.

---

## Step 7: Create Test Issues

Create issues for test specialists.

### Test Writer - Molecule Level

```bash
bd create "[Tests] Molecule-level unit tests" \
  -t test \
  -p 3 \
  --parent "$EPIC_ID" \
  --assignee test-writer-molecule \
  --description "Write 2-5 unit tests per molecule. Minimal mocking. Test composition logic."

TEST_MOLECULE_ID=$(bd list --format=json | jq -r '.[0].id')

# Depends on all molecules being complete
bd dep add "$TEST_MOLECULE_ID" "$DB_MOLECULE_1" --type blocks
bd dep add "$TEST_MOLECULE_ID" "$API_MOLECULE_1" --type blocks
bd dep add "$TEST_MOLECULE_ID" "$UI_MOLECULE_1" --type blocks
```

### Test Writer - Organism Level

```bash
bd create "[Tests] Organism-level integration tests" \
  -t test \
  -p 2 \
  --parent "$EPIC_ID" \
  --assignee test-writer-organism \
  --description "Write 2-8 integration tests per organism layer (database, API, UI). Test layer integration."

TEST_ORGANISM_ID=$(bd list --format=json | jq -r '.[0].id')

# Depends on all organisms being complete
bd dep add "$TEST_ORGANISM_ID" "$DB_ORGANISM_ID" --type blocks
bd dep add "$TEST_ORGANISM_ID" "$API_ORGANISM_ID" --type blocks
bd dep add "$TEST_ORGANISM_ID" "$UI_ORGANISM_ID" --type blocks
```

### Test Gap Analyzer

```bash
bd create "[Tests] Gap analysis and additional tests" \
  -t test \
  -p 1 \
  --parent "$EPIC_ID" \
  --assignee test-gap-analyzer \
  --description "Review all tests, identify gaps, write up to 10 additional strategic tests. Focus on integration gaps and critical paths."

TEST_GAP_ID=$(bd list --format=json | jq -r '.[0].id')

# Depends on all test writers being complete
bd dep add "$TEST_GAP_ID" "$TEST_MOLECULE_ID" --type blocks
bd dep add "$TEST_GAP_ID" "$TEST_ORGANISM_ID" --type blocks
```

---

## Step 8: Create Integration Issue

Final integration and E2E verification.

```bash
bd create "[Integration] Wire organisms and E2E verification" \
  -t integration \
  -p 1 \
  --parent "$EPIC_ID" \
  --assignee integration-assembler \
  --description "Wire all organisms together, verify E2E flows, run full project test suite, take screenshots."

INTEGRATION_ID=$(bd list --format=json | jq -r '.[0].id')

# Depends on all organisms and tests being complete
bd dep add "$INTEGRATION_ID" "$DB_ORGANISM_ID" --type blocks
bd dep add "$INTEGRATION_ID" "$API_ORGANISM_ID" --type blocks
bd dep add "$INTEGRATION_ID" "$UI_ORGANISM_ID" --type blocks
bd dep add "$INTEGRATION_ID" "$TEST_GAP_ID" --type blocks
```

---

## Step 9: Verify Issue Structure

```bash
# View dependency tree
bd dep tree "$EPIC_ID"

# Should show hierarchical structure:
# Epic
#   ├─ Database Organism
#   │   └─ Database Molecule
#   │       ├─ Email Atom
#   │       ├─ Password Atom
#   │       └─ Phone Atom
#   ├─ API Organism (blocked by Database)
#   │   └─ API Molecule
#   ├─ UI Organism (blocked by API)
#   │   └─ UI Molecule
#   ├─ Test (Molecules)
#   ├─ Test (Organisms)
#   ├─ Test (Gap Analysis)
#   └─ Integration (blocked by all)

# View ready work (should show only atoms initially)
bd ready

# Expected: Only atom issues show up (nothing blocks them)
```

---

## Step 9.5: Validate Dependency Graph

After creating all issues and dependencies, verify no circular dependencies exist:

```bash
# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if bv_available; then
    echo ""
    echo "Validating dependency graph for circular dependencies..."

    if ! check_cycles; then
        echo ""
        echo "❌ ERROR: Circular dependencies detected in issue structure!"
        echo ""
        get_graph_insights | jq -r '.cycles[] | "  Cycle: " + (. | join(" → "))'
        echo ""
        echo "This indicates a bug in the spec or issue creation logic."
        echo "Review the dependency structure and remove circular dependencies."
        echo ""
        echo "Common causes:"
        echo "  • Bidirectional dependencies between issues"
        echo "  • Layering violations (API depends on UI which depends on API)"
        echo "  • Incorrect dependency direction"
        echo ""
        echo "Fix cycles before proceeding with implementation."
        exit 1
    else
        echo "✓ No cycles detected - dependency graph is valid"
    fi
else
    echo ""
    echo "BV unavailable - skipping cycle detection"
    echo "Manually verify dependencies with: bd dep tree $EPIC_ID"
fi
```

---

## Step 10: Output Summary

Display summary for user:

```bash
echo "Beads issues created successfully for spec: [this-spec]"
echo ""
echo "Issue hierarchy:"
bd list --format=table

echo ""
echo "Ready to start (unblocked work):"
bd ready

echo ""
echo "Next steps:"
echo "1. Agents can use 'bd ready' to find unblocked work"
echo "2. Start with atom-writer (atoms have no blockers)"
echo "3. As atoms complete, molecules become unblocked"
echo "4. Follow the atomic design workflow: atoms → molecules → organisms → integration"
echo ""
echo "Tracking:"
echo "- View all issues: bd list"
echo "- View dependency tree: bd dep tree $EPIC_ID"
echo "- View ready work: bd ready"
echo "- Cross-session recovery: bd list --json | jq '.[] | select(.status==\"in_progress\")'"
```

---

## Success Criteria

✅ Beads initialized in spec folder (`.beads/` exists)
✅ Epic issue created for feature
✅ Organism issues created for each layer (database, API, UI)
✅ Molecule issues created as children of organisms
✅ Atom issues created as children of molecules
✅ Test issues created with proper dependencies
✅ Integration issue created (blocked by all organisms + tests)
✅ Blocking dependencies set (atoms block molecules, molecules block organisms)
✅ `bd ready` shows only unblocked atoms initially
✅ Dependency tree shows proper hierarchy

This structure ensures work flows bottom-up naturally through the atomic design dependency graph.
