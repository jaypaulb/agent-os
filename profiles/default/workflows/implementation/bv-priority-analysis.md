# BV Priority Analysis Workflow

This workflow guides agents in using BV's graph-based priority recommendations to identify high-impact work.

## Overview

BV analyzes the dependency graph to detect priority misalignments - cases where human-assigned priority doesn't match structural importance in the graph. This helps:

- Identify bottleneck work that deserves higher priority
- Detect over-prioritized work with low actual impact
- Make data-driven priority decisions based on graph metrics

## When to Use This Workflow

- At the start of a new implementation session
- When unsure which task to prioritize among multiple options
- When multiple tasks have the same nominal priority
- During orchestration to assign work to agents
- After discovering new work to calibrate priorities

## Prerequisites

- Beads mode enabled (`tracking_mode: beads`)
- BV installed and enabled (`bv_enabled: true`)
- At least 3+ issues with dependencies in the graph

## Workflow

### Step 1: Query Priority Recommendations

```bash
cd agent-os/specs/[this-spec]/

# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

# Get priority recommendations
PRIORITY_RECS=$(get_priority_recommendations)

# Display summary
echo "$PRIORITY_RECS" | jq -r '
    "=== Priority Analysis ===",
    "",
    "Total issues analyzed: \(.summary.total_issues)",
    "Recommendations generated: \(.summary.recommendations)",
    "High confidence (>0.8): \(.summary.high_confidence)",
    "",
    "Top Recommendations:",
    (.recommendations[:5] | .[] |
     "  • \(.issue_id): P\(.current_priority) → P\(.suggested_priority) (\(.confidence*100 | round)% confidence)",
     "    Reason: \(.reasoning)")
'
```

### Step 2: Understand Recommendation Types

Priority recommendations come in two directions:

#### Increase Priority (Low priority but high importance)

**What it means:** Issue has low human-assigned priority but high structural importance

**Common reasons:**
- **Bottleneck**: High betweenness centrality - bridges different parts of graph
- **Critical path**: On the longest dependency chain (keystone)
- **Wide impact**: Blocks many downstream tasks
- **Foundation**: High eigenvector centrality - connected to other important issues

**Action:** Consider escalating to higher priority

**Example:**
```
• bd-a1b2: P3 → P1 (95% confidence)
  Reason: Bottleneck with betweenness 0.42 - bridges auth and payment flows
```

#### Decrease Priority (High priority but low importance)

**What it means:** Issue has high human-assigned priority but low structural importance

**Common reasons:**
- **Leaf node**: Few or no dependents
- **Not on critical path**: Can be delayed without cascading effects
- **Low eigenvector**: Not connected to other high-value work
- **Isolated**: Completing it doesn't unblock significant downstream work

**Action:** Consider deferring to focus on higher-impact work

**Example:**
```
• bd-c3d4: P1 → P3 (88% confidence)
  Reason: Leaf node with 0 dependents - nice-to-have feature, not blocking
```

### Step 3: Review Misaligned Priorities

Focus on high-confidence recommendations (>0.8 confidence):

```bash
# Extract high-confidence recommendations
HIGH_CONF=$(echo "$PRIORITY_RECS" | jq -r '
    .recommendations[] |
    select(.confidence > 0.8) |
    "\(.issue_id)\t\(.current_priority)\t\(.suggested_priority)\t\(.direction)\t\(.reasoning)"
')

echo "High-confidence priority misalignments:"
echo "$HIGH_CONF" | column -t -s $'\t'
```

### Step 4: Apply Recommendations (Optional)

**IMPORTANT:** Priority recommendations are suggestions, not mandates. Human judgment still matters. Consider:

- Business priorities not reflected in the dependency graph
- Time-sensitive work (deadlines, releases)
- Customer commitments
- Strategic importance beyond structural metrics

If recommendations align with your judgment, apply updates:

```bash
# Extract high-confidence priority increases
HIGH_CONF_INCREASE=$(echo "$PRIORITY_RECS" | jq -r '
    .recommendations[] |
    select(.confidence > 0.8 and .direction == "increase") |
    "\(.issue_id) \(.suggested_priority) \(.reasoning)"
')

if [[ -n "$HIGH_CONF_INCREASE" ]]; then
    echo ""
    echo "High-confidence priority increases suggested:"
    echo "$HIGH_CONF_INCREASE"
    echo ""

    # Review with user before applying
    read -p "Apply these priority updates? [y/N]: " apply_choice

    if [[ "$apply_choice" =~ ^[Yy]$ ]]; then
        while IFS=' ' read -r issue_id new_priority reasoning; do
            # Extract just the priority number
            reasoning_rest="${reasoning#* }"  # Remove first word
            bd update "$issue_id" --priority "$new_priority" \
                --note "Priority updated based on graph analysis: $reasoning_rest"
            echo "✓ Updated $issue_id to P$new_priority"
        done <<< "$HIGH_CONF_INCREASE"
    fi
fi
```

### Step 5: Integrate with Task Selection

Use priority recommendations to enhance task selection:

```bash
# Get execution plan and priority recommendations
PLAN=$(get_execution_plan)
PRIORITIES=$(get_priority_recommendations)

# Find ready work with high recommended priority
RECOMMENDED_TASK=$(jq -n \
    --argjson plan "$PLAN" \
    --argjson priorities "$PRIORITIES" \
    '
    # Build lookup of recommendations
    ($priorities.recommendations | map({(.issue_id): .}) | add) as $recs |

    # Find actionable items with priority increase recommendations
    $plan.tracks[].items[] |
    select(.id as $id | $recs[$id].direction == "increase") |
    {
        id: .id,
        title: .title,
        current_priority: .priority,
        suggested_priority: ($recs[.id].suggested_priority),
        confidence: ($recs[.id].confidence),
        reasoning: ($recs[.id].reasoning),
        impact: .unblocks_count
    }
    ' | jq -s 'sort_by(-.confidence, .suggested_priority) | .[0]')

if [[ -n "$RECOMMENDED_TASK" && "$RECOMMENDED_TASK" != "null" ]]; then
    echo ""
    echo "=== High-Priority Recommendation ==="
    echo "$RECOMMENDED_TASK" | jq -r '
        "Issue: \(.id) - \(.title)",
        "Current: P\(.current_priority) → Suggested: P\(.suggested_priority)",
        "Confidence: \(.confidence*100 | round)%",
        "Reasoning: \(.reasoning)",
        "Impact: Unblocks \(.impact) downstream tasks"
    '
    echo ""
    echo "This work is both ready AND structurally important - strong candidate for next task"
fi
```

## Integration Points

This workflow is referenced by:

- **`/commands/implement-tasks/*/determine-tasks.md`** - Task selection enhancement
- **`/commands/orchestrate-tasks/orchestrate-tasks.md`** - Work assignment optimization
- **`/workflows/implementation/implement-with-beads.md`** - Ongoing execution guidance

## Real-World Example

### Scenario: User Authentication Feature

Initial state:
```
bd-101: Email validator atom                 P3 (low priority)
bd-102: Password validator atom               P3 (low priority)
bd-103: User profile page                     P1 (high priority)
bd-104: Social media login                    P1 (high priority)
```

Dependency graph:
```
bd-101 (email validator) ──blocks──> bd-201 (auth service) ──blocks──> bd-301 (login flow)
                                            └──blocks──> bd-302 (signup flow)
                                            └──blocks──> bd-303 (password reset)

bd-102 (password validator) ──blocks──> bd-201 (auth service)

bd-103 (profile page) ──no dependencies──

bd-104 (social login) ──no dependencies──
```

BV priority analysis:
```
• bd-101: P3 → P1 (98% confidence)
  Reason: Critical path item - blocks auth service which blocks 3 downstream flows

• bd-102: P3 → P1 (95% confidence)
  Reason: Bottleneck with high betweenness - auth service cannot proceed without it

• bd-103: P1 → P3 (85% confidence)
  Reason: Leaf node with 0 dependents - nice-to-have, not blocking anything

• bd-104: P1 → P2 (80% confidence)
  Reason: Isolated feature - doesn't unblock core authentication flows
```

**Insight:** The atoms (email validator, password validator) are structurally critical despite low human-assigned priority. The high-priority UI work (profile page, social login) is actually low-impact because it doesn't unblock other work.

**Action:** Escalate bd-101 and bd-102 to P1, defer bd-103 and bd-104. This unblocks the auth service, which unblocks 3 downstream flows.

## Understanding Confidence Scores

| Confidence | Meaning | Action |
|------------|---------|--------|
| **>0.9** | Very strong signal - graph structure clearly indicates misalignment | Strongly consider updating priority |
| **0.8-0.9** | Strong signal - multiple graph metrics agree | Review and likely update |
| **0.6-0.8** | Moderate signal - some evidence of misalignment | Review with human judgment |
| **<0.6** | Weak signal - marginal difference | Human priorities likely correct |

High confidence typically means:
- Multiple graph metrics agree (PageRank + betweenness + critical path)
- Large gap between current and suggested priority (P3 → P1, not P2 → P1)
- Clear structural pattern (bottleneck, keystone, or leaf node)

## Fallback Behavior

If BV unavailable, priority analysis returns empty recommendations:

```json
{
  "recommendations": [],
  "summary": {
    "total_issues": 0,
    "recommendations": 0,
    "high_confidence": 0
  }
}
```

Agents should gracefully continue with current priorities and manual judgment.

## Limitations

Priority recommendations are based on graph structure only. They **do not** consider:

- Business priorities (revenue impact, customer commitments)
- Time-sensitive work (deadlines, scheduled releases)
- Strategic initiatives (entering new markets, competitive features)
- External dependencies (third-party APIs, vendor timelines)
- Team capacity and specialization

Always combine graph-based recommendations with human judgment and business context.

## Best Practices

1. **Review recommendations daily** - Run priority analysis at the start of each work session
2. **Batch updates** - Apply priority updates in batches to avoid thrashing
3. **Document decisions** - Use `--note` flag when updating priorities to explain reasoning
4. **Monitor confidence** - Focus on >0.8 confidence recommendations first
5. **Validate with team** - Discuss priority changes that affect multiple people
6. **Re-analyze after changes** - Closing issues or adding dependencies changes the graph

## Troubleshooting

### No recommendations generated

**Causes:**
- Graph is too small (<3 issues)
- All priorities already aligned with graph structure
- No dependency edges (flat structure)

**Solution:** Ensure issues have proper dependencies and tags

### All recommendations have low confidence

**Causes:**
- Priorities are mostly correct
- Graph lacks strong structural patterns (no clear bottlenecks or keystones)
- Conflicting signals (PageRank says increase, betweenness says decrease)

**Solution:** Trust human priorities - graph analysis is supplementary, not authoritative

### Recommendations seem wrong

**Causes:**
- Graph doesn't reflect business priorities
- Missing dependencies (graph incomplete)
- Over-emphasis on structural metrics vs business value

**Solution:** Remember recommendations are based on graph structure alone - human judgment and business context override metrics
