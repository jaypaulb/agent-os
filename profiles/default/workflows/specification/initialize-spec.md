# Spec Initialization

## Core Responsibilities

1. **Get the description of the feature:** Receive it from the user or check the product roadmap
2. **Initialize Spec Structure**: Create the spec folder with date prefix
3. **Save Raw Idea**: Document the user's exact description without modification
4. **Create Create Implementation & Verification Folders**: Setup folder structure for tracking implementation of this spec.
5. **Prepare for Requirements**: Set up structure for next phase

## Workflow

### Step 1: Get the description of the feature

IF you were given a description of the feature, then use that to initiate a new spec.

OTHERWISE follow these steps to get the description:

1. Check `@agent-os/product/roadmap.md` to find the next feature in the roadmap.
2. OUTPUT the following to user and WAIT for user's response:

```
Which feature would you like to initiate a new spec for?

- The roadmap shows [feature description] is next. Go with that?
- Or provide a description of a feature you'd like to initiate a spec for.
```

**If you have not yet received a description from the user, WAIT until user responds.**

### Step 2: Initialize Spec Structure

Determine a kebab-case spec name from the user's description, then create the spec folder:

```bash
# Get today's date in YYYY-MM-DD format
TODAY=$(date +%Y-%m-%d)

# Determine kebab-case spec name from user's description
SPEC_NAME="[kebab-case-name]"

# Create dated folder name
DATED_SPEC_NAME="${TODAY}-${SPEC_NAME}"

# Store this path for output
SPEC_PATH="agent-os/specs/$DATED_SPEC_NAME"

# Create folder structure following architecture
mkdir -p $SPEC_PATH/planning
mkdir -p $SPEC_PATH/planning/visuals

echo "Created spec folder: $SPEC_PATH"
```

### Step 3: Create Implementation Folder

Create 2 folders:
- `$SPEC_PATH/implementation/`

Leave this folder empty, for now. Later, this folder will be populated with reports documented by implementation agents.

### Step 4: Choose Issue Tracking Mode

{{IF beads_enabled}}
Ask the user which issue tracking mode to use for this spec:

```
Which issue tracking mode would you like to use for this spec?

**Option 1: tasks.md (Recommended for simpler features)**
- Hierarchical task groups in tasks.md
- Checkbox-based progress tracking
- Best for: Single-developer work, straightforward features

**Option 2: beads (Recommended for complex features)**
- Distributed issue tracking with atomic design hierarchy
- Auto-dependency management, cross-session context
- Best for: Multi-session work, complex features, team collaboration
- Requires: beads CLI installed (https://github.com/steveyegge/beads)

Your choice?
```

Wait for user response.

**After receiving choice:**

Create spec configuration file:

```bash
# Create spec-config.yml
cat > $SPEC_PATH/spec-config.yml <<EOF
# Spec Configuration
tracking_mode: [user-choice]  # beads or tasks_md
created: $(date -u +%Y-%m-%dT%H:%M:%SZ)
beads_initialized: false
EOF
```

**If user chose beads:**

Check if beads is installed and initialize:

```bash
# Check if beads is installed
if ! command -v bd &> /dev/null; then
    echo ""
    echo "⚠️  Beads is not installed. Please install it first:"
    echo "   curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash"
    echo ""
    echo "After installing, run this command again."
    exit 1
fi

# Initialize beads in spec folder
cd $SPEC_PATH
bd init --stealth

# Update config to mark beads as initialized
sed -i 's/beads_initialized: false/beads_initialized: true/' spec-config.yml

echo "✓ Beads initialized in spec folder"
cd - > /dev/null
```

**If user chose tasks_md:**

```bash
echo "✓ Will use tasks.md for issue tracking"
```

{{ELSE}}
Create spec configuration file with tasks_md (beads is disabled):

```bash
# Create spec-config.yml
cat > $SPEC_PATH/spec-config.yml <<EOF
# Spec Configuration
tracking_mode: tasks_md
created: $(date -u +%Y-%m-%dT%H:%M:%SZ)
beads_initialized: false
EOF

echo "✓ Will use tasks.md for issue tracking (beads is disabled in config)"
```
{{ENDIF beads_enabled}}

### Step 5: Output Confirmation

Return or output the following:

```
Spec folder initialized: `[spec-path]`

Structure created:
- planning/ - For requirements and specifications
- planning/visuals/ - For mockups and screenshots
- implementation/ - For implementation documentation
- spec-config.yml - Spec configuration (tracking mode: [chosen-mode])

{{IF beads_enabled}}
Issue tracking: [chosen-mode]
{{IF tracking_mode_beads}}
- Beads initialized and ready
- Run `bd list` in spec folder to see issues
{{ELSE}}
- Will use tasks.md for task breakdown
{{ENDIF tracking_mode_beads}}
{{ELSE}}
Issue tracking: tasks.md
{{ENDIF beads_enabled}}

Ready for requirements research phase.
```

## Important Constraints

- Always use dated folder names (YYYY-MM-DD-spec-name)
- Pass the exact spec path back to the orchestrator
- Follow folder structure exactly
- Implementation folder should be empty, for now
