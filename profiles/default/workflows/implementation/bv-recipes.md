# BV Recipe System Integration

Use predefined and custom recipes for quick filtered views of beads issues.

## Overview

BV's recipe system provides pre-built, commonly-used filters that answer specific questions:

- "What can I work on right now?" → `actionable` recipe
- "What has the most impact?" → `high-impact` recipe
- "What's blocked and why?" → `blocked` recipe
- "Are there forgotten issues?" → `stale` recipe
- "What changed recently?" → `recent` recipe

Recipes are faster than manually constructing complex filters and provide consistent semantics across projects.

## Built-in Recipes

### 1. Actionable Recipe

**Question:** "What can I work on right now?"

**Shows:** Unblocked, ready-to-work issues

**Usage:**
```bash
bv --recipe actionable
```

**Equivalent filter:**
- Status: `open` or `in_progress`
- Blocked: `false` (no unresolved blockers)
- Ready: `true` (all dependencies complete)

**Agent use case:**
> "I just started a work session - show me what's available to work on immediately"

**Example output:**
```
bd-101: Email validator atom (P1)
bd-102: Password hasher atom (P1)
bd-103: UUID generator atom (P2)
```

### 2. High-Impact Recipe

**Question:** "What work will have the most structural impact?"

**Shows:** Top PageRank/betweenness scores

**Usage:**
```bash
bv --recipe high-impact
```

**Equivalent filter:**
- Sort by: PageRank (descending)
- Secondary sort: Betweenness (descending)
- Limit: Top 10

**Agent use case:**
> "I want to work on something that unblocks the most downstream work"

**Example output:**
```
bd-201: Auth service molecule (PageRank: 0.42, Betweenness: 0.35)
bd-101: Email validator (PageRank: 0.38, Betweenness: 0.15)
bd-301: API layer organism (PageRank: 0.28, Betweenness: 0.48)
```

### 3. Blocked Recipe

**Question:** "What's blocked and why?"

**Shows:** Issues waiting on dependencies

**Usage:**
```bash
bv --recipe blocked
```

**Equivalent filter:**
- Status: `open`
- Blocked: `true` (has unresolved blockers)
- Shows blocker info

**Agent use case:**
> "Why isn't more work available? What's waiting?"

**Example output:**
```
bd-202: Session manager molecule
  Blocked by: bd-101 (Email validator)

bd-301: Login API organism
  Blocked by: bd-201 (Auth service), bd-202 (Session manager)
```

### 4. Stale Recipe

**Question:** "Are there forgotten issues?"

**Shows:** Open issues untouched for 30+ days

**Usage:**
```bash
bv --recipe stale
```

**Equivalent filter:**
- Status: `open`
- Last updated: > 30 days ago
- Sort by: Age (oldest first)

**Agent use case:**
> "Find abandoned work that needs attention"

**Example output:**
```
bd-105: Logging utility (last updated: 45 days ago)
bd-203: Cache layer (last updated: 38 days ago)
```

### 5. Recent Recipe

**Question:** "What changed recently?"

**Shows:** Issues updated in last 7 days

**Usage:**
```bash
bv --recipe recent
```

**Equivalent filter:**
- Last updated: < 7 days ago
- Sort by: Update time (most recent first)

**Agent use case:**
> "Catch up on what happened while I was away"

**Example output:**
```
bd-101: Email validator (closed 2 hours ago)
bd-102: Password hasher (status changed to in_progress 1 day ago)
bd-201: Auth service (created 3 days ago)
```

## Agent Integration

### Quick Task Discovery

Use recipes to find work at session start:

```bash
# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if bv_available; then
    echo "=== Quick Work Discovery ==="
    echo ""

    # Show actionable work
    ACTIONABLE=$(get_recipe actionable)
    ACTIONABLE_COUNT=$(echo "$ACTIONABLE" | jq -r '. | length')

    echo "Actionable items: $ACTIONABLE_COUNT"
    if [[ "$ACTIONABLE_COUNT" -gt 0 ]]; then
        echo "$ACTIONABLE" | jq -r '.[] | "  • \(.id): \(.title) (P\(.priority))"'
    fi

    echo ""

    # Show high-impact work
    HIGH_IMPACT=$(get_recipe high-impact)
    echo "High-impact items (top 3):"
    echo "$HIGH_IMPACT" | jq -r '.[:3] | .[] | "  • \(.id): \(.title)"'
fi
```

### Priority Check: Is High-Impact Work Being Ignored?

Check if high-impact work is actionable but not being worked on:

```bash
if bv_available; then
    HIGH_IMPACT=$(get_recipe high-impact)
    ACTIONABLE=$(get_recipe actionable)

    # Find high-impact work that's actionable but not in progress
    IGNORED=$(jq -n \
        --argjson hi "$HIGH_IMPACT" \
        --argjson act "$ACTIONABLE" \
        '
        ($act | map(.id)) as $actionable_ids |
        $hi[] |
        select(.status == "open") |
        select([.id] | inside($actionable_ids))
        ')

    if [[ -n "$IGNORED" && "$IGNORED" != "null" ]]; then
        echo ""
        echo "⚠️  High-impact work available but not in progress:"
        echo "$IGNORED" | jq -r '"  • \(.id): \(.title)"'
        echo ""
        echo "Consider prioritizing these items - they have high structural impact"
    fi
fi
```

### Combine Recipes with Execution Plan

Use recipes to filter execution plan results:

```bash
# Get both execution plan and actionable recipe
PLAN=$(get_execution_plan)
ACTIONABLE=$(get_recipe actionable)

# Find which track has the most actionable high-impact work
BEST_TRACK=$(jq -n \
    --argjson plan "$PLAN" \
    --argjson actionable "$ACTIONABLE" \
    '
    ($actionable | map(.id)) as $actionable_ids |

    $plan.tracks[] |
    {
        track: .track_id,
        items: [.items[] | select([.id] | inside($actionable_ids))],
        count: ([.items[] | select([.id] | inside($actionable_ids))] | length),
        impact: ([.items[] | select([.id] | inside($actionable_ids)) | .unblocks_count] | add // 0)
    } |
    select(.count > 0)
    ' | jq -s 'sort_by(-.impact) | .[0]')

if [[ -n "$BEST_TRACK" && "$BEST_TRACK" != "null" ]]; then
    echo "=== Recommended Track ==="
    echo "$BEST_TRACK" | jq -r '
        "Track \(.track): \(.count) actionable items",
        "Total impact: \(.impact) downstream tasks unblocked"
    '
fi
```

## Listing Available Recipes

Query BV for all available recipes (built-in + custom):

```bash
# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if bv_available; then
    RECIPES=$(list_recipes)

    echo "=== Available Recipes ==="
    echo "$RECIPES" | jq -r '.recipes[] | "• \(.name): \(.description)"'
fi
```

**Example output:**
```
• actionable: Unblocked issues ready to work on
• high-impact: Issues with highest PageRank/betweenness scores
• blocked: Issues waiting on dependencies
• stale: Open issues untouched for 30+ days
• recent: Issues updated in last 7 days
```

## Recipe Quick Checks in Commands

**Already integrated in Phase 2:**

File: `/commands/implement-tasks/single-agent/1-determine-tasks.md`

```bash
# Quick recipe checks
echo ""
echo "=== Quick Views ==="
ACTIONABLE=$(get_recipe actionable)
HIGH_IMPACT=$(get_recipe high-impact)
BLOCKED=$(get_recipe blocked)

echo "Actionable items: $(echo "$ACTIONABLE" | jq -r '. | length')"
echo "High-impact items: $(echo "$HIGH_IMPACT" | jq -r '. | length')"
echo "Blocked items: $(echo "$BLOCKED" | jq -r '. | length')"
```

This was already added in Phase 2, lines 59-68.

## Custom Recipes (Advanced)

Users can define custom recipes in `.beads/recipes.yml` (if BV supports it):

**Example custom recipe:**
```yaml
# .beads/recipes.yml
recipes:
  - name: my-work
    description: Issues assigned to me that are actionable
    filter:
      assignee: atom-writer
      status: [open, in_progress]
      blocked: false

  - name: critical
    description: P1 issues on critical path
    filter:
      priority: 1
      keystone_score: ">3"
```

**Usage:**
```bash
bv --recipe my-work
bv --recipe critical
```

Note: Custom recipes are a BV feature - consult BV docs for syntax.

## Integration Examples

### 1. Session Startup

```bash
#!/bin/bash
# session-start.sh

source bv-helpers.md

echo "=== Work Session Started ==="
echo ""

# Show actionable work
ACTIONABLE=$(get_recipe actionable)
echo "Actionable: $(echo "$ACTIONABLE" | jq -r '. | length') items"
echo "$ACTIONABLE" | jq -r '.[] | "  • \(.id): \(.title)"' | head -5

# Warn if stale issues
STALE=$(get_recipe stale)
STALE_COUNT=$(echo "$STALE" | jq -r '. | length')
if [[ "$STALE_COUNT" -gt 0 ]]; then
    echo ""
    echo "⚠️  $STALE_COUNT stale issues (30+ days old)"
    echo "   Consider reviewing: bd list --tag stale"
fi

# Show recent activity
echo ""
echo "Recent activity (last 7 days):"
get_recipe recent | jq -r '.[:3] | .[] | "  • \(.id): \(.title) (\(.status))"'
```

### 2. Daily Standup Summary

```bash
#!/bin/bash
# daily-standup.sh

source bv-helpers.md

echo "=== Daily Standup Summary ==="
echo ""

# What's done
RECENT_CLOSED=$(get_recipe recent | jq -r '[.[] | select(.status == "closed")] | length')
echo "Completed yesterday: $RECENT_CLOSED issues"

# What's in progress
IN_PROGRESS=$(bd list --status in_progress --json | jq -r '. | length')
echo "In progress: $IN_PROGRESS issues"

# What's blocked
BLOCKED=$(get_recipe blocked)
BLOCKED_COUNT=$(echo "$BLOCKED" | jq -r '. | length')
echo "Blocked: $BLOCKED_COUNT issues"

if [[ "$BLOCKED_COUNT" -gt 0 ]]; then
    echo ""
    echo "Blockers to resolve:"
    echo "$BLOCKED" | jq -r '.[:3] | .[] | "  • \(.id) blocked by \(.blocked_by)"'
fi

# What's available
ACTIONABLE=$(get_recipe actionable)
echo ""
echo "Available to work on: $(echo "$ACTIONABLE" | jq -r '. | length') items"
```

### 3. Work Prioritization

```bash
#!/bin/bash
# prioritize-work.sh

source bv-helpers.md

# Get high-impact and actionable recipes
HIGH_IMPACT=$(get_recipe high-impact)
ACTIONABLE=$(get_recipe actionable)

# Find intersection: high-impact AND actionable
PRIORITY_WORK=$(jq -n \
    --argjson hi "$HIGH_IMPACT" \
    --argjson act "$ACTIONABLE" \
    '
    ($act | map(.id)) as $actionable_ids |
    $hi[] |
    select([.id] | inside($actionable_ids))
    ')

echo "=== Priority Work (High-Impact + Actionable) ==="
echo "$PRIORITY_WORK" | jq -r '
    "• \(.id): \(.title)",
    "  Priority: P\(.priority)",
    "  Impact: PageRank \(.pagerank // "N/A")",
    ""
'
```

## Fallback Behavior

If BV unavailable, recipe functions return empty arrays:

```bash
get_recipe actionable    # Returns []
get_recipe high-impact   # Returns []
get_recipe blocked       # Returns []
list_recipes             # Returns {"recipes": []}
```

Agents gracefully fall back to basic `bd ready` and `bd list` commands.

## Best Practices

1. **Use actionable recipe at session start** - Quick way to find available work
2. **Check stale recipe weekly** - Prevent forgotten issues from aging
3. **Review blocked recipe when work slows** - Identify bottlenecks
4. **Combine recipes for smart filtering** - Intersection of high-impact + actionable = priority work
5. **Use recent recipe after breaks** - Catch up on changes quickly

## Comparison: Recipes vs. Manual Filters

### Without Recipes (Manual)

```bash
# Find actionable work (manual)
bd list --json | jq -r '
    .[] |
    select(.status == "open") |
    select(.blocked == false) |
    "\(.id): \(.title)"
'
```

### With Recipes (Simpler)

```bash
# Find actionable work (recipe)
bv --recipe actionable
```

Recipes provide:
- **Shorter syntax** - One command instead of complex jq
- **Consistent semantics** - "actionable" means the same across projects
- **Performance** - BV can optimize recipe execution
- **Composability** - Easy to combine with other BV features

## Summary

Recipes provide quick, pre-built views of common issue queries:

| Recipe | Question | Use Case |
|--------|----------|----------|
| `actionable` | What can I work on? | Session start, finding ready work |
| `high-impact` | What has most impact? | Prioritization, identifying critical work |
| `blocked` | What's waiting? | Bottleneck analysis, dependency debugging |
| `stale` | What's forgotten? | Cleanup, abandoned work review |
| `recent` | What changed? | Catching up, session summaries |

Use recipes to answer common questions quickly without constructing complex filters.
