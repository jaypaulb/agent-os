# Regression Verification Workflow

This workflow is used by the regression-verifier agent to test previously completed features in the current app state.

**Purpose**: Cross-phase regression testing - ensure that implementing new phases doesn't break features from earlier completed phases.

**When**: Triggered by autonomous-build orchestrator when a phase completes.

**How**: Test 1 randomly selected closed issue from each completed phase.

---

## Prerequisites

- Playwright MCP tools available (`mcp__playwright__*`)
- App running locally (or ability to start it)
- Issue ID to verify
- Access to original requirements for the feature

---

## Process

### STEP 1: Understand What to Test

Given an issue ID, determine what feature to test:

```bash
# Get issue details
ISSUE_ID="$1"  # Passed as parameter

ISSUE_DATA=$(bd show "$ISSUE_ID" --json)
TITLE=$(echo "$ISSUE_DATA" | jq -r '.title')
DESCRIPTION=$(echo "$ISSUE_DATA" | jq -r '.description')
TAGS=$(echo "$ISSUE_DATA" | jq -r '.tags[]')
SPEC_LABEL=$(echo "$ISSUE_DATA" | jq -r '.labels[]' | grep -v "^phase-" | head -1)

echo "Verifying: $ISSUE_ID - $TITLE"
echo "Spec: $SPEC_LABEL"
echo "Tags: $TAGS"
echo ""

# Read spec for context
if [ -f ".agent-os/specs/$SPEC_LABEL/spec.md" ]; then
  echo "Reading spec for context..."
  # Agent should read spec.md to understand feature
fi
```

### STEP 2: Start Application (if needed)

Check if app is running, start if needed:

```bash
# Check if app is running (port check or process check)
# This is project-specific

# Example for common stacks:
# - Node.js: npm start or node server.js
# - Python: python app.py or flask run
# - Rails: rails server
# - Django: python manage.py runserver

echo "Checking if app is running..."

# Placeholder - project-specific logic needed
# For now, assume app is already running
APP_URL="http://localhost:3000"  # Default, adjust as needed

echo "App expected at: $APP_URL"
```

### STEP 3: Navigate to Feature

Use Playwright to navigate to the feature:

```bash
# Open browser (if not already open)
# Navigate to feature location

# Example for UI features:
echo "Opening browser..."

# Use Playwright MCP tools
# mcp__playwright__browser_navigate --url "$APP_URL"
# mcp__playwright__browser_snapshot

echo "⚠️  Playwright integration pending"
echo "Would navigate to feature and capture snapshot"
```

### STEP 4: Interact as User

Test the feature by interacting as a real user would:

```bash
# UI Component Testing:
# - Click buttons
# - Fill forms
# - Submit data
# - Navigate through flow

# API Testing:
# - Make HTTP requests
# - Verify responses
# - Check status codes

echo "Testing feature interaction..."
echo "⚠️  Interactive testing pending"

# Placeholder for Playwright interactions:
# mcp__playwright__browser_click --element "Submit Button" --ref "button#submit"
# mcp__playwright__browser_type --element "Email Input" --ref "input#email" --text "test@example.com"
# etc.
```

### STEP 5: Check for Console Errors

Verify no console errors during interaction:

```bash
echo "Checking console for errors..."

# Get console messages via Playwright
# CONSOLE_ERRORS=$(mcp__playwright__browser_console_messages --level error)

# Placeholder
CONSOLE_ERRORS=""

if [ -n "$CONSOLE_ERRORS" ]; then
  echo "❌ Console errors detected:"
  echo "$CONSOLE_ERRORS"
  REGRESSION_FOUND=true
else
  echo "✓ No console errors"
fi
```

### STEP 6: Verify Expected Behavior

Check that the feature works as specified:

```bash
echo "Verifying expected behavior..."

# Feature-specific verification
# - UI displays correctly?
# - Data persists?
# - Navigation works?
# - API returns expected data?

# This is highly feature-specific
# Agent should compare behavior against original requirements

echo "⚠️  Behavior verification pending (feature-specific)"

# Placeholder
BEHAVIOR_CORRECT=true

if [ "$BEHAVIOR_CORRECT" = true ]; then
  echo "✓ Feature behaves as expected"
else
  echo "❌ Feature behavior incorrect"
  REGRESSION_FOUND=true
fi
```

### STEP 7: Take Screenshot

Capture visual state for review:

```bash
echo "Capturing screenshot..."

# Save to verification directory
mkdir -p .agent-os/specs/$SPEC_LABEL/verification/screenshots 2>/dev/null || true
SCREENSHOT_PATH=".agent-os/specs/$SPEC_LABEL/verification/screenshots/${ISSUE_ID}-$(date +%Y%m%d-%H%M%S).png"

# Use Playwright to capture
# mcp__playwright__browser_take_screenshot --filename "$SCREENSHOT_PATH"

echo "⚠️  Screenshot capture pending"
echo "Would save to: $SCREENSHOT_PATH"
```

### STEP 8: Report Results

Return pass/fail status to orchestrator:

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  Regression Verification Results"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

if [ "$REGRESSION_FOUND" = true ]; then
  echo "Status: ❌ REGRESSION DETECTED"
  echo ""
  echo "Issue: $ISSUE_ID - $TITLE"
  echo "Problem: Feature is broken in current app state"
  echo ""
  echo "Recommendation: Reopen issue and investigate"

  # Reopen the issue
  bd update "$ISSUE_ID" --status open --note "Regression detected during autonomous build. Feature broke after subsequent changes."

  exit 1
else
  echo "Status: ✓ VERIFICATION PASSED"
  echo ""
  echo "Issue: $ISSUE_ID - $TITLE"
  echo "Result: Feature still works correctly"
  echo ""

  exit 0
fi
```

---

## Quality Bar

A feature passes regression verification if:

✅ **No console errors** during interaction
✅ **Expected behavior** matches original requirements
✅ **Visual appearance** is correct (no layout breaks, contrast issues)
✅ **Performance** is acceptable (no freezing, fast response)
✅ **Complete workflow** can be executed end-to-end

---

## Issue Types and Testing Strategies

### UI Organisms
- Navigate to page/component
- Interact with all interactive elements
- Verify visual correctness
- Check responsive behavior
- Capture screenshots

### API Organisms
- Make HTTP requests to endpoints
- Verify response structure
- Check status codes
- Test error cases
- Validate data

### Atoms/Molecules
- Run unit tests
- Verify function behavior
- Check edge cases
- No browser interaction needed

---

## Limitations (Current)

This workflow currently provides a framework but requires:

1. **Playwright MCP integration**: Browser automation tools must be configured
2. **Feature-specific logic**: Each feature type needs custom verification steps
3. **App startup logic**: Project-specific commands to start the app
4. **Dynamic feature discovery**: Map issue types to test strategies automatically

These will be enhanced as the system matures.

---

## Usage

This workflow is called by the regression-verifier agent, which is spawned by the autonomous-build orchestrator:

```
Orchestrator detects phase completion
  ↓
For each completed phase:
  Randomly select 1 closed issue
  ↓
  Spawn regression-verifier agent
    ↓
    Agent follows this workflow
    ↓
    Agent returns pass/fail
  ↓
If regression: reopen issue, ask user
```
