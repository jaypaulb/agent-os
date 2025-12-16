# Create Beads Issues from Spec

This workflow auto-generates beads issues from spec.md with atomic design hierarchy and proper dependency relationships.

## Prerequisites

- Beads must be installed: `bd --version`
- Spec folder must exist at `agent-os/specs/[this-spec]/`
- `spec.md` must be complete
- **Beads repository initialized at PROJECT ROOT** (not in spec folder)

## Setup: Source BV Helpers

**IMPORTANT:** Source BV helpers at the start of this workflow for graph intelligence and graceful fallback.

```bash
# Source BV helpers (adjust path relative to your location)
source ../../../workflows/implementation/bv-helpers.md

# Verify BV availability
if bv_available; then
    echo "✓ BV available - using graph intelligence"
else
    echo "⚠️  BV unavailable - using basic beads commands"
fi
```

This provides:
- `check_cycles()` - Detect circular dependencies
- `get_graph_insights()` - Get bottlenecks, cycles, influencers
- `get_execution_plan()` - Get optimized execution plan
- Automatic fallback to `bd` commands if BV unavailable

## Overview

This workflow analyzes `spec.md` to:
1. Identify organisms (database layer, API layer, UI layer)
2. Extract required molecules from organism requirements
3. Identify atoms needed by molecules
4. Create hierarchical beads issues with parent-child relationships
5. Set blocking dependencies (atoms block molecules, molecules block organisms)
6. Result: `bd ready` shows only unblocked atoms initially

---

## Step 1: Initialize Beads at Project Root (Once)

```bash
# From project root

# Check if already initialized
if [ ! -d ".beads" ]; then
  echo "Initializing Beads at project root..."
  bd init --stealth
  echo "✓ Beads initialized at project root"
else
  echo "✓ Beads already initialized at project root"
fi

# Verify initialization
ls .beads/
# Should show: issues.jsonl, config.yaml, beads.db
```

**What this does:**
- Creates `.beads/` at PROJECT ROOT (not in spec folder)
- All phases will share the same Beads repository
- Enables tracking dependencies between phases
- `bd ready` shows unblocked work across ALL phases
- Uses `--stealth` mode (local only, no repo changes if unwanted)

**IMPORTANT:** This should only initialize once. If called multiple times (e.g., for different phases), it will skip if already initialized.

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

## Step 3: Create Phase Epic

Create the parent epic for THIS phase (not the whole product):

```bash
# Determine phase number and name from spec location
# Example: agent-os/specs/user-authentication/ → Phase 1: User Authentication
PHASE_NUM="[phase-number]"  # e.g., 1, 2, 3
PHASE_NAME="[phase-name]"    # e.g., "User Authentication"
SPEC_SLUG="[this-spec]"      # e.g., "user-authentication"

bd create "Phase $PHASE_NUM: $PHASE_NAME" \
  -t epic \
  -p 1 \
  --label "phase-$PHASE_NUM" \
  --label "$SPEC_SLUG" \
  --description "$(cat <<EOF
[1-2 sentence summary from spec.md]

Spec location: agent-os/specs/$SPEC_SLUG/spec.md
Phase: $PHASE_NUM of [total-phases]

This epic represents all work for Phase $PHASE_NUM.
EOF
)"
```

**Capture the epic ID** for use as parent:
```bash
EPIC_ID=$(bd list --format=json | jq -r '.[0].id')
echo "Phase $PHASE_NUM Epic created: $EPIC_ID"
```

**IMPORTANT:** All issues created in subsequent steps should include:
- `--label "phase-$PHASE_NUM"` - to filter by phase
- `--label "$SPEC_SLUG"` - to identify the spec

This enables:
- `bd list --label phase-1` - see all Phase 1 work
- `bd list --label user-authentication` - see all work for this spec
- `bd ready --label phase-1` - see ready work for Phase 1 only

**NOTE:** The examples below show `--label "phase-$PHASE_NUM"` and `--label "$SPEC_SLUG"` added to the first organism. Add these labels to ALL issues (organisms, molecules, atoms, tests, integration).

---

## Step 4: Create Organism Issues

Create one issue per organism layer. Use epic as parent.

### 4A: Database Layer Organism

```bash
bd create "[Organism] Database Layer" \
  -t organism \
  -p 2 \
  --parent "$EPIC_ID" \
  --label "phase-$PHASE_NUM" \
  --label "$SPEC_SLUG" \
  --label database \
  --label backend \
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
  --label api \
  --label backend \
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
  --label "ui" \
  --label "frontend" \
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
  --labeldatabase \
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
  --labelapi \
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
  --label "ui" \
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
  --labelvalidation \
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
  --labelvalidation \
  --assignee atom-writer \
  --description "Pure function: validatePassword(pwd: string) => {valid: boolean, errors: string[]}. Min 8 chars, 1 uppercase, 1 number."

PWD_ATOM=$(bd list --format=json | jq -r '.[0].id')

bd dep add "$DB_MOLECULE_1" "$PWD_ATOM" --type blocks

# Atom 3: Phone formatter
bd create "[Atom] Phone number formatter" \
  -t atom \
  -p 4 \
  --parent "$DB_MOLECULE_1" \
  --labelformatting \
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

## Step 9: Set Phase Dependencies (Multi-Phase Projects Only)

If this is part of a multi-phase project, set dependencies between phase epics based on roadmap order.

**When to use:**
- When autonomous-plan creates tasks for multiple phases
- When phases have logical dependencies (Phase 2 depends on Phase 1, etc.)

**Skip if:**
- This is a single-phase project
- Phases are completely independent

```bash
# Get this phase's integration issue ID (last issue created)
INTEGRATION_ID=$(bd list --label "phase-$PHASE_NUM" --format=json | \
  jq -r '.[] | select(.title | contains("[Integration]")) | .id')

# If this is NOT Phase 1, block this phase's epic on previous phase's integration
if [ "$PHASE_NUM" -gt 1 ]; then
  PREV_PHASE=$((PHASE_NUM - 1))

  echo "Linking Phase $PHASE_NUM dependencies to Phase $PREV_PHASE..."

  # Find previous phase's integration issue
  PREV_INTEGRATION=$(bd list --label "phase-$PREV_PHASE" --format=json | \
    jq -r '.[] | select(.title | contains("[Integration]")) | .id' | head -1)

  if [ -z "$PREV_INTEGRATION" ]; then
    echo "⚠️  Warning: No integration issue found for Phase $PREV_PHASE"
    echo "   Phase $PHASE_NUM epic will NOT be blocked"
    echo "   This may allow phases to run out of order"
  else
    # Set dependency: This phase's epic blocks on previous phase's integration
    bd dep add "$EPIC_ID" "$PREV_INTEGRATION" --type blocks

    echo "✓ Phase $PHASE_NUM epic ($EPIC_ID) blocks on Phase $PREV_PHASE integration ($PREV_INTEGRATION)"

    # Verify dependency was created
    if bd show "$EPIC_ID" --json | jq -e '.blocks | contains(["'"$PREV_INTEGRATION"'"])' > /dev/null; then
      echo "✓ Dependency verified in graph"
    else
      echo "❌ ERROR: Dependency creation failed"
      exit 1
    fi
  fi
else
  echo "✓ Phase 1 - no dependencies (foundation phase)"
fi
```

**What this does:**
- Phase 1: No blockers (starts immediately)
- Phase 2: Blocked by Phase 1's integration issue
- Phase 3: Blocked by Phase 2's integration issue
- etc.

This ensures phases execute in order from roadmap, with later phases only starting after earlier phases are fully integrated.

**Alternative: Manual dependencies**
If phases have complex dependencies (e.g., Phase 3 depends on both Phase 1 AND Phase 2), manually set them:
```bash
bd dep add "$PHASE_3_EPIC" "$PHASE_1_INTEGRATION" --type blocks
bd dep add "$PHASE_3_EPIC" "$PHASE_2_INTEGRATION" --type blocks
```

---

## Step 10: Verify Issue Structure

```bash
# View dependency tree (use BV for enhanced visualization if available)
if bv_available; then
    echo "Dependency tree (BV enhanced):"
    bv --filter "parent:$EPIC_ID" --show-deps --format tree 2>/dev/null || bd dep tree "$EPIC_ID"
else
    bd dep tree "$EPIC_ID"
fi

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
# Use BV for optimized execution plan if available
if bv_available; then
    echo ""
    echo "Ready work (optimized by BV):"
    READY=$(bv --unblocked --filter "label:phase-$PHASE_NUM" --format json 2>/dev/null)
    echo "$READY" | jq -r '.[] | "  • \(.id): \(.title) (P\(.priority // "?"))"'
else
    echo ""
    echo "Ready work:"
    bd ready --label "phase-$PHASE_NUM"
fi

# Expected: Only atom issues show up (nothing blocks them)
```

**What this does:**
- Uses BV for enhanced tree visualization if available
- Uses BV execution plan for optimized ready work
- Falls back to `bd` commands if BV unavailable

---

## Step 10.5: Validate Dependency Graph

After creating all issues and dependencies, verify no circular dependencies exist:

```bash
echo ""
echo "Validating dependency graph for circular dependencies..."

if ! check_cycles; then
    echo ""
    echo "❌ ERROR: Circular dependencies detected in issue structure!"
    echo ""

    # Get detailed cycle information from BV
    INSIGHTS=$(get_graph_insights)
    echo "$INSIGHTS" | jq -r '.cycles[] | "  Cycle: " + (. | join(" → "))'

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
```

**What this does:**
- Uses `check_cycles()` from BV helpers (automatically falls back to permissive if BV unavailable)
- If cycles detected, uses `get_graph_insights()` to show cycle paths
- Exits with error if cycles found (prevents broken dependency graph)

---

## Step 11: Output Summary

Display summary for user:

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Beads issues created for Phase $PHASE_NUM: $PHASE_NAME"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Spec: agent-os/specs/$SPEC_SLUG/spec.md"
echo "Epic: $EPIC_ID"
echo ""

# Show phase issues (use BV if available for better formatting)
echo "Phase $PHASE_NUM issues:"
if bv_available; then
    # Use BV recipe system for better organization
    PHASE_ISSUES=$(bv --filter "label:phase-$PHASE_NUM" --format json 2>/dev/null || bd list --label "phase-$PHASE_NUM" --json)
    echo "$PHASE_ISSUES" | jq -r '.[] | "  [\(.type | ascii_upcase)] \(.title) (P\(.priority // "?"))"'
else
    bd list --label "phase-$PHASE_NUM" --format=table
fi

echo ""
# Show ready work using BV execution plan or bd ready
echo "Ready to start (Phase $PHASE_NUM unblocked work):"
if bv_available; then
    READY=$(bv --filter "label:phase-$PHASE_NUM,status:pending" --unblocked --format json 2>/dev/null || bd ready --label "phase-$PHASE_NUM" --json)
    READY_COUNT=$(echo "$READY" | jq '. | length')
    if [ "$READY_COUNT" -gt 0 ]; then
        echo "$READY" | jq -r '.[] | "  • \(.id): \(.title)"'
    else
        echo "  (No unblocked work yet - dependencies not complete)"
    fi
else
    bd ready --label "phase-$PHASE_NUM"
fi

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📊 All-Phase Tracking (Project Root)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Commands for agents:"
if bv_available; then
    echo "  bv --unblocked                    # Best work to do next (BV optimized)"
    echo "  bv --recipe actionable            # All actionable work"
    echo "  bv --recipe high-impact           # High-impact tasks"
    echo "  bv --filter 'label:phase-1'       # Phase 1 only"
fi
echo "  bd ready                          # All ready work (any phase)"
echo "  bd list                           # All issues across all phases"
echo "  bd ready --label phase-1          # Phase-specific ready work"
echo ""
echo "Next steps:"
echo "1. Agents use 'bd ready' or 'bv --unblocked' to find work"
echo "2. Start with atoms (no blockers)"
echo "3. As atoms complete, molecules become unblocked"
echo "4. Follow atomic design: atoms → molecules → organisms → integration"
echo "5. When Phase $PHASE_NUM integration completes, Phase $((PHASE_NUM + 1)) becomes unblocked"
```

**What changed:**
- Uses BV functions (`bv --filter`, `bv --unblocked`) when available
- Falls back gracefully to `bd` commands if BV unavailable
- Works with JSON directly instead of echoing command success/failure
- Shows BV-specific commands in output if BV is available

---

## Success Criteria

✅ Beads initialized at PROJECT ROOT (`.beads/` exists at project root)
✅ Phase epic created with proper labels (`phase-$PHASE_NUM`, `$SPEC_SLUG`)
✅ Organism issues created for each layer (database, API, UI) with phase labels
✅ Molecule issues created as children of organisms with phase labels
✅ Atom issues created as children of molecules with phase labels
✅ Test issues created with proper dependencies and phase labels
✅ Integration issue created (blocked by all organisms + tests)
✅ Phase dependencies set (if multi-phase project)
✅ Blocking dependencies set (atoms block molecules, molecules block organisms)
✅ `bd ready` shows only unblocked atoms initially (across all phases)
✅ `bd ready --label phase-$PHASE_NUM` shows only this phase's ready work
✅ Dependency tree shows proper hierarchy
✅ All issues filterable by phase

**Multi-Phase Success:**
✅ All phases tracked in single `.beads/` repository at project root
✅ Phase epics blocked by previous phase's integration issues
✅ `bd list` shows ALL issues across ALL phases
✅ `bd list --type epic` shows all phase epics with dependencies

This structure ensures:
- Work flows bottom-up through atomic design within each phase
- Phases execute in roadmap order (Phase 2 starts after Phase 1 completes)
- Unified tracking across entire product
