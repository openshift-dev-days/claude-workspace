# OpenShift Dev Days Claude Workspace

Claude Code plugin and workspace configuration for OpenShift Dev Days Roadshow development and operations.

## What's Included

### Skills

- **`release-dev-days`** - Cut production releases, tag repos, update AgnosticV prod.yaml, and submit PRs
- **`dev-days-triage`** - Fetch open issues across roadshow repos, inspect code, draft fixes, and open PRs
- **`rhdp-lab-validation`** - Validate provisioned OpenShift Dev Days clusters ordered from RHDP
- **`cluster-test-push`** - Push local repo changes to a live cluster's GitLab for rapid testing

### Context

- **`CLAUDE.md`** - Project context with architecture overview, repo layout, conventions, and RHDH patterns

## Installation

### 1. Install the Plugin

Add this repo as a marketplace and install the plugin:

```bash
/plugin marketplace add https://github.com/openshift-dev-days/claude-workspace.git
/plugin install ocp-dev-days@ocp-dev-days
```

The skills will be available in all your Claude Code sessions.

### 2. Set Up Your Workspace

Clone the Dev Days repos as sibling directories:

```bash
mkdir -p ~/workspaces/dev-days
cd ~/workspaces/dev-days

# Clone the main repos
gh repo clone rhpds/ocp-dev-days-rdshw-showroom
gh repo clone rhpds/ocp-dev-days-rdshw-gitops
gh repo clone rhpds/ocp-dev-days-rdshw-automation
gh repo clone rhpds/agnosticv

# Copy CLAUDE.md for project context
curl -O https://raw.githubusercontent.com/openshift-dev-days/claude-workspace/main/CLAUDE.md
```

### 3. Use in Claude Code

Navigate to your workspace and start using the skills:

```bash
cd ~/workspaces/dev-days
claude

# Use skills via slash commands
/release-dev-days
/dev-days-triage
/rhdp-lab-validation
/cluster-test-push
```

## Development

To test changes to skills locally before pushing:

```bash
cd /path/to/claude-workspace
claude --plugin-dir .
```

## Architecture Overview

See [CLAUDE.md](CLAUDE.md) for complete architecture documentation, including:

- Cluster and tenant provisioning flow
- Repository layout and purposes
- AgnosticV configuration structure
- RHDH per-user entity patterns for RBAC
- Release conventions and tag formats

## Contributing

Submit PRs to add new skills or improve existing ones. Each skill should:

- Follow the [SKILL.md format](https://code.claude.com/docs/en/skills)
- Include clear usage instructions
- Document prerequisites and dependencies
- Handle errors gracefully and provide helpful feedback
