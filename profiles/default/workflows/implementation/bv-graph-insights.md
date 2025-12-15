# BV Graph Insights Workflow

Extract structural intelligence from the dependency graph to identify bottlenecks, keystones, critical paths, and foundational work.

## Overview

BV analyzes the issue dependency graph using multiple graph algorithms to surface structural patterns that aren't obvious from priority or status alone. This helps agents and users make better decisions about what to work on next.

## Graph Metrics Explained

### 1. Bottlenecks (Betweenness Centrality)

**What it measures:** Issues that bridge different parts of the graph

**Mathematical definition:** Number of shortest paths that pass through this node

**Agent use case:**
> "If this blocker isn't resolved, it will delay multiple independent work streams"

**Example:**
```
Feature A ──> [API Auth Middleware] ──> User Management
Feature B ──> [API Auth Middleware] ──> Payment Processing
Feature C ──> [API Auth Middleware] ──> Admin Dashboard
```

The API auth middleware has high betweenness because it sits between multiple independent features. Completing it unblocks three parallel work streams.

**When to prioritize:**
- High betweenness score (>0.3)
- Multiple teams/agents waiting on this issue
- Blocks parallel work tracks

### 2. Keystones (Critical Path Score)

**What it measures:** Issues on the longest dependency chain from start to finish

**Mathematical definition:** Length of the longest path through this node

**Agent use case:**
> "This is on the critical path - delays here push back the entire project timeline"

**Example:**
```
Database Schema (keystone=5)
  └─> ORM Models (keystone=4)
      └─> Business Logic (keystone=3)
          └─> API Endpoints (keystone=2)
              └─> UI Components (keystone=1)
```

The database schema is a keystone because it's at the start of the longest dependency chain. Any delay cascades through all downstream work.

**When to prioritize:**
- High critical path score (>4)
- Long dependency chains behind this issue
- Project deadline is tight

### 3. Influencers (Eigenvector Centrality)

**What it measures:** Issues connected to other important issues (recursive importance)

**Mathematical definition:** Importance based on importance of neighbors (PageRank-like)

**Agent use case:**
> "This issue is foundational - many high-priority items depend on it"

**Example:**
```
[Email Validator Utility] (influencer)
  ├─> User Registration (high priority)
  ├─> Password Reset (high priority)
  ├─> Contact Form (high priority)
  └─> Newsletter Signup (medium priority)
```

The email validator has high eigenvector centrality because it's used by multiple high-priority features. It's not just widely depended-on - it's depended-on by important things.

**When to prioritize:**
- High eigenvector score (>0.2)
- Used by many critical features
- Foundational utility/library

### 4. Hubs (HITS Hub Score)

**What it measures:** Issues that depend on many other issues (aggregators)

**Mathematical definition:** HITS hub score - points to many authorities

**Agent use case:**
> "This epic/organism pulls together many components"

**Example:**
```
[User Dashboard] (hub)
  └─ depends on ─> Authentication Service
  └─ depends on ─> User Profile Service
  └─ depends on ─> Notification Service
  └─ depends on ─> Activity Feed Service
  └─ depends on ─> Settings Service
```

The user dashboard is a hub because it aggregates many services. It's the integration point.

**When to prioritize:**
- Save hubs for late in the project (after dependencies complete)
- Good candidate for dedicated integration agent
- Needs comprehensive testing (many integration points)

### 5. Authorities (HITS Authority Score)

**What it measures:** Issues that many others depend on (prerequisites)

**Mathematical definition:** HITS authority score - pointed to by many hubs

**Agent use case:**
> "This is a prerequisite for many downstream features"

**Example:**
```
Multiple features depend on:
  [User Authentication Service] (authority)
```

The auth service is an authority because many features depend on it. It's a common prerequisite.

**When to prioritize:**
- High authority score (>0.3)
- Blocks many downstream issues
- Early in the project (unblocks others)

## Usage

### Query All Insights

```bash
cd agent-os/specs/[this-spec]/

# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if bv_available; then
    INSIGHTS=$(get_graph_insights)

    # Display summary
    echo "$INSIGHTS" | jq -r '
        "=== Graph Insights Summary ===",
        "",
        "Top 5 Bottlenecks (bridges between work streams):",
        (.bottlenecks[:5] | .[] |
         "  • \(.id): \(.title) (betweenness: \(.value | tonumber | . * 100 | round / 100))"),
        "",
        "Top 5 Keystones (critical path items):",
        (.keystones[:5] | .[] |
         "  • \(.id): \(.title) (path length: \(.value | tonumber | round))"),
        "",
        "Top 5 Influencers (foundational items):",
        (.influencers[:5] | .[] |
         "  • \(.id): \(.title) (eigenvector: \(.value | tonumber | . * 100 | round / 100))"),
        "",
        "Top 5 Authorities (widely depended-on):",
        (.authorities[:5] | .[] |
         "  • \(.id): \(.title) (authority: \(.value | tonumber | . * 100 | round / 100))"),
        "",
        "Graph Health:",
        "  Density: \(.clusterDensity | tonumber | . * 100 | round / 100)%",
        "  Cycles detected: \(.cycles | length)"
    '
fi
```

### Integration with Task Selection

Enhance task selection to prefer high-impact structural work:

```bash
# Get both execution plan and insights
PLAN=$(get_execution_plan)
INSIGHTS=$(get_graph_insights)

# Find ready work that is ALSO a bottleneck or keystone
CRITICAL_READY=$(jq -n \
    --argjson plan "$PLAN" \
    --argjson insights "$INSIGHTS" \
    '
    # Build map of critical issues
    (($insights.bottlenecks + $insights.keystones) | map({(.id): .value}) | add) as $critical |

    # Find actionable items that are also critical
    $plan.tracks[].items[] as $item |
    ($critical[$item.id] // null) as $score |
    select($score != null) |
    {
        id: $item.id,
        title: $item.title,
        type: (if ($insights.bottlenecks | map(.id) | index($item.id)) then "bottleneck" else "keystone" end),
        score: $score,
        unblocks: $item.unblocks_count,
        priority: $item.priority
    }
    ')

# Sort by score and show top recommendation
BEST_CRITICAL=$(echo "$CRITICAL_READY" | jq -s 'sort_by(-.score) | .[0]')

if [[ -n "$BEST_CRITICAL" && "$BEST_CRITICAL" != "null" ]]; then
    echo ""
    echo "=== Critical Structural Work Available ==="
    echo "$BEST_CRITICAL" | jq -r '
        "Issue: \(.id) - \(.title)",
        "Type: \(.type | ascii_upcase)",
        "Score: \(.score | tonumber | . * 100 | round / 100)",
        "Unblocks: \(.unblocks) downstream tasks",
        "Priority: P\(.priority)",
        "",
        "This work is both ready AND structurally critical - strong candidate for next task"
    '
fi
```

## Agent Decision Examples

### Scenario 1: Choose Between Two Atoms

**Context:**
- Atom A: `bd-101` Email validator (P2, ready)
- Atom B: `bd-102` Phone formatter (P2, ready)

**Graph insights:**
```
bd-101 (email validator):
  - Eigenvector: 0.35 (high influencer)
  - Used by: User registration, password reset, contact form, newsletter

bd-102 (phone formatter):
  - Eigenvector: 0.05 (low influencer)
  - Used by: User profile (optional field)
```

**Decision:** Prioritize Atom A (email validator)

**Reasoning:** Despite same priority, bd-101 is a foundational influencer used by multiple critical features. bd-102 is used in only one place for an optional field.

### Scenario 2: Identify Hidden Critical Work

**Context:**
- Molecule X: `bd-201` Data validation service (P3, ready)
- Appears low priority, but has high betweenness

**Graph insights:**
```
bd-201 (data validation service):
  - Betweenness: 0.48 (high bottleneck)
  - Bridges: User input flows AND API data flows
```

**Decision:** Escalate bd-201 to P1

**Reasoning:** This molecule sits between two major feature areas (user input and API integration). Completing it unblocks parallel work in both areas. The low priority doesn't reflect its structural importance.

### Scenario 3: Detect Foundational Work

**Context:**
- Utility U: `bd-050` Error handling library (P2, ready)
- Seems like "nice-to-have"

**Graph insights:**
```
bd-050 (error handling library):
  - Eigenvector: 0.42 (high influencer)
  - Authority: 0.38 (widely depended-on)
  - Used by: 12 organisms across all layers
```

**Decision:** Implement with extra care and comprehensive tests

**Reasoning:** This utility is foundational - many organisms depend on it. A bug here affects 12 downstream components. Invest in quality even though it's "just a utility."

### Scenario 4: Defer Hub Work

**Context:**
- Hub H: `bd-500` Admin dashboard (P1, ready)
- High priority but also high hub score

**Graph insights:**
```
bd-500 (admin dashboard):
  - Hub score: 0.65 (very high)
  - Depends on: 8 services (6 not yet complete)
```

**Decision:** Defer bd-500 despite P1 priority

**Reasoning:** This is an aggregator that depends on 8 services, 6 of which aren't complete yet. Working on it now would hit blockers. Better to let dependencies complete first, then integrate.

## Real-World Example: User Authentication Feature

### Initial State

```
bd-101: Email validator atom                 P3
bd-102: Password hasher atom                  P3
bd-103: Token generator atom                  P3
bd-201: Auth service molecule                 P2
bd-202: Session manager molecule              P2
bd-301: Login API endpoint organism           P1
bd-302: Signup API endpoint organism          P1
bd-303: User profile UI organism              P1
```

### Dependency Graph

```
bd-101 (email) ──┐
bd-102 (password) ┤──> bd-201 (auth service) ──┐
bd-103 (token) ───┘                             ├──> bd-301 (login API)
                                                ├──> bd-302 (signup API)
bd-202 (session) ────────────────────────────────┘

bd-301 (login API) ────> bd-303 (profile UI)
bd-302 (signup API) ───> bd-303 (profile UI)
```

### BV Graph Insights

```json
{
  "bottlenecks": [
    {"id": "bd-201", "value": 0.52, "title": "Auth service molecule"}
  ],
  "keystones": [
    {"id": "bd-101", "value": 5, "title": "Email validator"},
    {"id": "bd-201", "value": 4, "title": "Auth service"}
  ],
  "influencers": [
    {"id": "bd-101", "value": 0.38, "title": "Email validator"},
    {"id": "bd-201", "value": 0.42, "title": "Auth service"}
  ],
  "hubs": [
    {"id": "bd-303", "value": 0.45, "title": "User profile UI"}
  ],
  "authorities": [
    {"id": "bd-201", "value": 0.40, "title": "Auth service"}
  ]
}
```

### Insights Interpretation

1. **bd-201 (auth service) is CRITICAL**
   - Bottleneck (0.52): Bridges email/password/token atoms with all API endpoints
   - Keystone (4): On critical path, long dependency chain behind it
   - Influencer (0.42): Connected to many important nodes
   - Authority (0.40): 2 organisms depend on it
   - **Action:** Escalate from P2 to P1, assign to experienced developer

2. **bd-101 (email validator) is FOUNDATIONAL**
   - Keystone (5): At the start of the longest path
   - Influencer (0.38): Used transitively by many features
   - **Action:** Implement early with comprehensive tests

3. **bd-303 (profile UI) is a HUB**
   - Hub (0.45): Depends on both login and signup APIs
   - **Action:** Defer until bd-301 and bd-302 are complete

4. **bd-102, bd-103, bd-202 are SUPPORTING**
   - Low scores across all metrics
   - **Action:** Normal priority, implement in dependency order

### Recommended Execution Order

Based on graph insights (not just priority):

1. **bd-101** (email validator) - Keystone + influencer
2. **bd-102, bd-103** (password, token) - Prerequisites for bd-201
3. **bd-201** (auth service) - Critical bottleneck
4. **bd-202** (session manager) - Prerequisite for APIs
5. **bd-301, bd-302** (login, signup APIs) - Parallel tracks
6. **bd-303** (profile UI) - Hub, defer until dependencies complete

This order unblocks work faster than strictly following P1/P2/P3 priorities.

## Combining with Priority Recommendations

Graph insights and priority recommendations work together:

```bash
# Get both
INSIGHTS=$(get_graph_insights)
PRIORITIES=$(get_priority_recommendations)

# Find issues that are BOTH structurally critical AND priority-misaligned
CRITICAL_MISALIGNED=$(jq -n \
    --argjson insights "$INSIGHTS" \
    --argjson priorities "$PRIORITIES" \
    '
    ($insights.bottlenecks + $insights.keystones) as $critical |
    $priorities.recommendations[] |
    select(.direction == "increase") |
    select([.issue_id] | inside($critical | map(.id)))
    ')

if [[ -n "$CRITICAL_MISALIGNED" && "$CRITICAL_MISALIGNED" != "null" ]]; then
    echo "=== High-Confidence Priority Escalations ==="
    echo "$CRITICAL_MISALIGNED" | jq -r '
        "• \(.issue_id): P\(.current_priority) → P\(.suggested_priority)",
        "  Structural: Critical path + bottleneck",
        "  Recommendation: \(.reasoning)",
        ""
    '
fi
```

## Integration Points

This workflow is used by:

- **`/workflows/implementation/implement-with-beads.md`** - Task selection with structural warnings
- **`/commands/implement-tasks/*/determine-tasks.md`** - Display insights during task determination
- **`/commands/orchestrate-tasks/orchestrate-tasks.md`** - Assign critical work to experienced agents

## Fallback Behavior

If BV unavailable, graph insights return empty:

```json
{
  "bottlenecks": [],
  "keystones": [],
  "influencers": [],
  "hubs": [],
  "authorities": [],
  "cycles": [],
  "clusterDensity": 0
}
```

Agents gracefully continue with priority-based selection.

## Limitations

Graph insights are based on structure only. They **do not** consider:

- Business value beyond dependencies
- Time-sensitive work (deadlines, releases)
- Developer expertise (some work needs specialists)
- External blockers (third-party APIs, vendor delays)
- Political/organizational priorities

Always combine structural insights with human judgment.

## Best Practices

1. **Review insights weekly** - Graph structure changes as work completes
2. **Escalate bottlenecks** - High betweenness deserves priority
3. **Implement keystones early** - Critical path items set the timeline
4. **Test influencers thoroughly** - Foundational work affects many dependents
5. **Defer hubs** - Let dependencies complete before integration work
6. **Recognize authorities** - Widely depended-on work needs extra care

## Troubleshooting

### All scores are very low (<0.1)

**Causes:**
- Graph is too small (<5 issues)
- Flat structure (no or few dependencies)
- Disconnected issues (no common paths)

**Solution:** Graph metrics meaningful only for ≥10 interconnected issues

### Scores don't match intuition

**Causes:**
- Missing dependencies in graph
- Graph doesn't reflect business priorities
- Structural importance ≠ business importance

**Solution:** Use insights as input, not gospel. Add missing dependencies if they exist.

### Different metrics disagree

**Causes:**
- Issue has different roles (e.g., bottleneck but not keystone)
- Graph has complex structure

**Solution:** Normal - consider which metric matters most for current context (unblocking parallel work = prioritize betweenness, timeline = prioritize critical path)
