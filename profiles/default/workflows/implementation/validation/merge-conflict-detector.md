# Merge Conflict Detector

This workflow detects merge conflicts before committing an organism. It's Gate 3 of the validation pipeline.

## Overview

When multiple agents work in parallel, they may modify the same files (especially shared files like type definitions, barrel files, config files). This detector identifies conflicts early so they can be resolved before committing.

**Common conflict scenarios:**
- Type definition files (e.g., `interfaces/index.ts`)
- Barrel/index files (e.g., `components/index.ts`)
- Configuration files (e.g., `routes.ts`, `app.config.ts`)
- Shared constants or enums

---

## Usage

Called by organism-validator.md (Gate 3) or directly by orchestrator.

**Input:**
- Current working branch with uncommitted changes
- Main branch (or target branch) to merge into

**Output:**
- Exit code 0: No conflicts
- Exit code 3: Conflicts detected
- `/tmp/conflict-details.txt`: Conflict diff and affected files

---

## Detection Process

### Step 1: Stash Current Changes

```bash
echo "=== Merge Conflict Detection ==="
echo ""

# Stash current changes (if any uncommitted)
HAS_UNCOMMITTED=$(git status --porcelain 2>/dev/null | wc -l)

if [ "$HAS_UNCOMMITTED" -gt 0 ]; then
  echo "Stashing uncommitted changes..."
  git stash push -m "autonomous-build-conflict-check" > /dev/null 2>&1
  STASHED=true
else
  STASHED=false
fi

# Record current branch
CURRENT_BRANCH=$(git branch --show-current 2>/dev/null || echo "master")
CURRENT_COMMIT=$(git rev-parse HEAD 2>/dev/null)
```

### Step 2: Attempt Merge with Main

```bash
# Fetch latest main
echo "Fetching latest main..."
git fetch origin main > /dev/null 2>&1 || git fetch origin master > /dev/null 2>&1

# Determine main branch name
MAIN_BRANCH="main"
if ! git rev-parse origin/main >/dev/null 2>&1; then
  MAIN_BRANCH="master"
fi

echo "Checking for conflicts with origin/$MAIN_BRANCH..."

# Try merge (dry run)
git merge --no-commit --no-ff "origin/$MAIN_BRANCH" > /tmp/merge-output.txt 2>&1
MERGE_EXIT_CODE=$?
```

### Step 3: Analyze Merge Result

```bash
if [ $MERGE_EXIT_CODE -ne 0 ]; then
  # Merge conflicts detected
  echo "⚠️  Merge conflicts detected"
  echo ""

  # Extract conflicted files
  CONFLICTED_FILES=$(git diff --name-only --diff-filter=U 2>/dev/null || echo "")

  if [ -n "$CONFLICTED_FILES" ]; then
    echo "Conflicted files:"
    echo "$CONFLICTED_FILES" | while read FILE; do
      echo "  • $FILE"
    done
    echo ""

    # Get conflict diff for each file
    echo "Conflict details:" > /tmp/conflict-details.txt
    echo "" >> /tmp/conflict-details.txt

    echo "$CONFLICTED_FILES" | while read FILE; do
      echo "=== $FILE ===" >> /tmp/conflict-details.txt
      git diff "$FILE" >> /tmp/conflict-details.txt 2>/dev/null || true
      echo "" >> /tmp/conflict-details.txt
    done

    # Categorize conflict type
    categorize_conflicts "$CONFLICTED_FILES"

  else
    echo "❌ Merge failed but no conflicted files found"
    echo "This may indicate a non-conflict merge failure"
    echo ""
    cat /tmp/merge-output.txt
  fi

  # Abort merge
  git merge --abort > /dev/null 2>&1

  # Restore stashed changes
  if [ "$STASHED" = true ]; then
    git stash pop > /dev/null 2>&1
  fi

  # Exit with conflict code
  exit 3

else
  # No conflicts
  echo "✓ No merge conflicts"

  # Abort merge (we're just checking, not actually merging)
  git merge --abort > /dev/null 2>&1 || git reset --hard "$CURRENT_COMMIT" > /dev/null 2>&1

  # Restore stashed changes
  if [ "$STASHED" = true ]; then
    git stash pop > /dev/null 2>&1
  fi

  echo ""
  exit 0
fi
```

---

## Conflict Categorization

Identify conflict patterns to help with resolution:

```bash
categorize_conflicts() {
  local CONFLICTED_FILES=$1

  # Initialize categories
  TYPE_CONFLICTS=0
  BARREL_CONFLICTS=0
  CONFIG_CONFLICTS=0
  CODE_CONFLICTS=0

  echo "$CONFLICTED_FILES" | while read FILE; do
    case "$FILE" in
      *interfaces*.ts | *types*.ts | *.d.ts)
        TYPE_CONFLICTS=$((TYPE_CONFLICTS + 1))
        echo "  Type definition conflict: $FILE"
        ;;
      */index.ts | */index.js | */index.tsx | */index.jsx)
        BARREL_CONFLICTS=$((BARREL_CONFLICTS + 1))
        echo "  Barrel file conflict: $FILE"
        ;;
      *.config.* | *routes* | *app.*)
        CONFIG_CONFLICTS=$((CONFIG_CONFLICTS + 1))
        echo "  Configuration conflict: $FILE"
        ;;
      *)
        CODE_CONFLICTS=$((CODE_CONFLICTS + 1))
        echo "  Code conflict: $FILE"
        ;;
    esac
  done

  echo ""
  echo "Conflict summary:" >> /tmp/conflict-details.txt
  echo "  Type conflicts: $TYPE_CONFLICTS" >> /tmp/conflict-details.txt
  echo "  Barrel conflicts: $BARREL_CONFLICTS" >> /tmp/conflict-details.txt
  echo "  Config conflicts: $CONFIG_CONFLICTS" >> /tmp/conflict-details.txt
  echo "  Code conflicts: $CODE_CONFLICTS" >> /tmp/conflict-details.txt
  echo "" >> /tmp/conflict-details.txt

  # Add resolution hints
  if [ $TYPE_CONFLICTS -gt 0 ]; then
    echo "Type conflict resolution hint:" >> /tmp/conflict-details.txt
    echo "  Both agents likely added new type definitions." >> /tmp/conflict-details.txt
    echo "  Merge both additions, ensuring no duplicate names." >> /tmp/conflict-details.txt
    echo "" >> /tmp/conflict-details.txt
  fi

  if [ $BARREL_CONFLICTS -gt 0 ]; then
    echo "Barrel file resolution hint:" >> /tmp/conflict-details.txt
    echo "  Both agents likely added new exports." >> /tmp/conflict-details.txt
    echo "  Merge both export lists in alphabetical order." >> /tmp/conflict-details.txt
    echo "" >> /tmp/conflict-details.txt
  fi

  if [ $CONFIG_CONFLICTS -gt 0 ]; then
    echo "Config conflict resolution hint:" >> /tmp/conflict-details.txt
    echo "  Both agents likely modified configuration." >> /tmp/conflict-details.txt
    echo "  Carefully merge both changes, testing functionality." >> /tmp/conflict-details.txt
    echo "" >> /tmp/conflict-details.txt
  fi
}
```

---

## Conflict Details Output

The `/tmp/conflict-details.txt` file contains:

```
Conflict details:

=== src/templates/interfaces/index.ts ===
<<<<<<< HEAD (origin/main)
export interface User {
  id: string;
  passwordHash: string;
}
=======
export interface User {
  id: string;
  token?: string;
}
>>>>>>> current-branch

=== src/organisms/components/index.ts ===
<<<<<<< HEAD (origin/main)
export { LoginForm } from './LoginForm';
export { SignupForm } from './SignupForm';
=======
export { LoginForm } from './LoginForm';
export { PasswordReset } from './PasswordReset';
>>>>>>> current-branch

Conflict summary:
  Type conflicts: 1
  Barrel conflicts: 1
  Config conflicts: 0
  Code conflicts: 0

Type conflict resolution hint:
  Both agents likely added new type definitions.
  Merge both additions, ensuring no duplicate names.

Barrel file resolution hint:
  Both agents likely added new exports.
  Merge both export lists in alphabetical order.
```

---

## Integration with Validation Pipeline

Called from organism-validator.md Gate 3:

```bash
# In organism-validator.md:

echo "=== Gate 3: Merge Detection ==="
echo ""

# Run detector
bash profiles/default/workflows/implementation/validation/merge-conflict-detector.md
CONFLICT_EXIT_CODE=$?

if [ $CONFLICT_EXIT_CODE -eq 3 ]; then
  # Conflicts detected
  echo "❌ Gate 3 FAILED: Merge conflicts detected"

  # Read conflict details
  CONFLICT_DETAILS=$(cat /tmp/conflict-details.txt 2>/dev/null || echo "No details available")
  echo "$CONFLICT_DETAILS"

  # Return to orchestrator for conflict resolution (Phase 4)
  exit 3
else
  # No conflicts
  echo "✓ Gate 3 PASSED: No merge conflicts"
fi
```

---

## Integration with Orchestrator

The orchestrator handles conflicts in STEP 3:

```bash
# In autonomous-build.md STEP 3 (Handle Agent Completions):

# Run validation pipeline
bash profiles/default/workflows/implementation/validation/organism-validator.md
VALIDATION_EXIT_CODE=$?

if [ $VALIDATION_EXIT_CODE -eq 3 ]; then
  # Merge conflict detected
  echo "⚠️  Merge conflict detected for $ORGANISM_ID"

  # Read conflict details
  CONFLICT_DETAILS=$(cat /tmp/conflict-details.txt 2>/dev/null)
  echo "$CONFLICT_DETAILS"

  # Phase 4: Trigger conflict resolution workflow
  # Options:
  # 1. Retry with conflict awareness (feed diff to agent)
  # 2. Serialize conflicting organism (block until other complete)
  # 3. Escalate to manual review (create issue)

  # For now (Phase 3): Move to blocked queue
  ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
  BLOCKED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
    blocked_by: \"merge-conflict\",
    reason: \"Merge conflict with main branch\",
    blocked_at: \"$(date -Iseconds)\",
    conflict_details: \"$(cat /tmp/conflict-details.txt | jq -sR .)\"
  }")

  jq "
    .blocked += [$BLOCKED_DATA] |
    .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
  " .beads/autonomous-state/work-queue.json > /tmp/wq.json
  mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  echo "Organism moved to blocked queue"
  echo "Phase 4 will implement conflict resolution strategies"
fi
```

---

## Limitations

**Current implementation:**
- Detects conflicts but doesn't resolve them automatically
- Requires Phase 4 (Merge Conflict Resolution) for automatic resolution
- Blocks organism until conflict is resolved

**Phase 4 will add:**
- Conflict-aware retry (feed diff to agent)
- Sequential fallback (serialize conflicting organisms)
- Manual review escalation (create issue for Jaypaul)

---

## See Also

- `/profiles/default/workflows/implementation/validation/organism-validator.md` - Full 5-gate pipeline
- `/profiles/default/commands/autonomous-build/multi-agent/autonomous-build.md` - Orchestrator (STEP 3)
- `/home/jaypaulb/.claude/plans/vast-pondering-iverson.md` - Phase 4: Merge Conflict Resolution
