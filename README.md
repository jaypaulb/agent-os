<img width="1280" height="640" alt="agent-os-og" src="https://github.com/user-attachments/assets/f70671a2-66e8-4c80-8998-d4318af55d10" />

# Agent OS - Atomic Design + Beads Fork

> **Note:** This is a fork of [buildermethods/agent-os](https://github.com/buildermethods/agent-os) with custom enhancements for atomic design principles and beads issue tracking integration.

## Your system for spec-driven agentic development.

[Agent OS](https://buildermethods.com/agent-os) transforms AI coding agents from confused interns into productive developers. With structured workflows that capture your standards, your stack, and the unique details of your codebase, Agent OS gives your agents the specs they need to ship quality code on the first try—not the fifth.

Use it with:

✅ Claude Code, Cursor, or any other AI coding tool.

✅ New products or established codebases.

✅ Big features, small fixes, or anything in between.

✅ Any language or framework.

---

## 🚀 Fork Enhancements

This fork adds powerful capabilities for atomic design development and distributed issue tracking:

### Atomic Design Architecture (10 New Agents)

**Tier 1 - Foundation Builders:**
- `atom-writer` - Pure functions, constants, utilities (zero dependencies)
- `molecule-composer` - Compose 2-3 atoms into simple helpers

**Tier 2 - Domain Builders (Organisms):**
- `database-layer-builder` - Models, migrations, associations
- `api-layer-builder` - Controllers, endpoints, authentication
- `ui-component-builder` - Components, forms, pages, styles

**Tier 3 - Test Specialists:**
- `test-writer-molecule` - Unit tests for molecules
- `test-writer-organism` - Integration tests for organisms
- `test-gap-analyzer` - Identify and fill test coverage gaps

**Tier 4 - Integration:**
- `template-designer` - Define interfaces, schemas, protocols
- `integration-assembler` - Wire organisms together, E2E verification

### Beads Issue Tracking Integration

Integrates [Beads](https://github.com/steveyegge/beads) - a distributed, git-backed issue tracker designed for AI agents:

- **Auto-creation** - Automatically generate beads issues from specs with atomic design hierarchy
- **Ready queue** - Agents query `bd ready` to find unblocked work
- **Cross-session context** - Agents recover context across sessions via beads history
- **Discovered-from links** - Agents create dependency links when discovering new work

### Dual Tracking Mode

Choose your workflow at spec initialization:
- **tasks.md mode** - Traditional hierarchical task lists (simple features)
- **beads mode** - Distributed issue tracking (complex features, multi-session work)

All workflows support both modes via conditional compilation.

---

## 📦 Installation (Fork)

### Quick Start

```bash
# Clone this fork
git clone https://github.com/jaypaulb/agent-os.git ~/agent-os

# Install into your project
cd /path/to/your/project
~/agent-os/scripts/project-install.sh
```

### With Beads Support (Optional)

**Beads installs automatically when needed:**
- When you choose beads mode during spec initialization
- If beads is not installed, you'll be prompted to:
  - **(i)nstall beads now** - Automatic installation
  - **(f)all back to tasks.md** - Use tasks.md instead

**Or install beads manually beforehand:**
```bash
curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash
bd --version
```

### Update to Latest

```bash
cd ~/agent-os
git pull origin master

# Re-install in your project
cd /path/to/your/project
~/agent-os/scripts/project-install.sh --re-install
```

---

## 🔄 Syncing with Upstream

To pull updates from the original `buildermethods/agent-os`:

```bash
cd ~/agent-os
git fetch upstream
git merge upstream/master
git push origin master
```

---

## 📚 Documentation

**Fork-specific documentation:**
- See [FORK-CHANGES.md](FORK-CHANGES.md) for detailed technical changes
- Atomic design standard: `profiles/default/standards/global/atomic-design.md`
- Beads workflows: `profiles/default/workflows/implementation/`

**Upstream documentation:**
- Docs, installation, usage, & best practices 👉 [buildermethods.com/agent-os](https://buildermethods.com/agent-os)
- Original repo: [buildermethods/agent-os](https://github.com/buildermethods/agent-os)

---

### Follow updates & releases

Read the [changelog](CHANGELOG.md)

[Subscribe to be notified of major new releases of Agent OS](https://buildermethods.com/agent-os)

---

### Created by Brian Casel @ Builder Methods

Created by Brian Casel, the creator of [Builder Methods](https://buildermethods.com), where Brian helps professional software developers and teams build with AI.

Get Brian's free resources on building with AI:
- [Builder Briefing newsletter](https://buildermethods.com)
- [YouTube](https://youtube.com/@briancasel)

Join [Builder Methods Pro](https://buildermethods.com/pro) for official support and connect with our community of AI-first builders:
