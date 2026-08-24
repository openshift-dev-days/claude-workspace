---
name: dev-days-triage
description: >
  Triage open issues and PRs across OpenShift Dev Days repositories, inspect
  code, draft fixes, and open PRs against dev/main branches.
---

# Skill: OpenShift Dev Days Issue Triage & Resolution

Use this skill when analyzing open issues, bug reports, or feature requests for the **OpenShift Dev Days Roadshow**.

## Monitored Repositories & Paths

| Repository | Path | Purpose |
|---|---|---|
| [ocp-dev-days-rdshw-gitops](https://github.com/rhpds/ocp-dev-days-rdshw-gitops) | `./ocp-dev-days-rdshw-gitops/` | Helm charts for cluster & tenant bootstrapping |
| [ocp-dev-days-rdshw-automation](https://github.com/rhpds/ocp-dev-days-rdshw-automation) | `./ocp-dev-days-rdshw-automation/` | Ansible roles for cluster & tenant provisioning |
| [ocp-dev-days-rdshw-showroom](https://github.com/rhpds/ocp-dev-days-rdshw-showroom) | `./ocp-dev-days-rdshw-showroom/` | Antora workshop lab guides |
| [agnosticv](https://github.com/rhpds/agnosticv) | `./agnosticv/` | RHDP catalog definitions & variable overrides |

## Workflow

### Step 1: Gather Open Issues

Fetch open issues across all monitored repos and present a summary table:

```bash
for repo in ocp-dev-days-rdshw-gitops ocp-dev-days-rdshw-automation ocp-dev-days-rdshw-showroom agnosticv; do
  gh issue list --repo rhpds/$repo --state open --json number,title,labels,createdAt
done
```

Present a numbered summary to the user:

```
#  | Repo        | Issue | Title                          | Labels
1  | gitops      | #42   | Broken HPA in tenant chart     | bug
2  | showroom    | #15   | Typo in module 3               | docs
3  | automation  | #8    | Keycloak role fails on retry   | bug
```

### Step 2: User Selects Issues

Ask the user which issues to work on before proceeding. Do not start fixes without the user selecting issues first. Example prompt:

> Which issues would you like me to work on? (e.g., "1, 3" or "all")

### Step 3: Inspect & Plan

For each selected issue:

1. Fetch full issue details: `gh issue view <number> --repo rhpds/<repo> --comments`
2. Locate affected files:
   - **Helm / Argo CD changes:** `ocp-dev-days-rdshw-gitops/cluster/` or `tenant/`
   - **Ansible / Role changes:** `ocp-dev-days-rdshw-automation/roles/`
   - **Lab Guide / Showroom text:** `ocp-dev-days-rdshw-showroom/modules/`
   - **AgnosticV Config:** `agnosticv/openshift_cnv/ocp-dev-days-rdshw-combined/common.yaml`
3. Present a brief plan for each issue before implementing.

### Step 4: Branch, Fix, Verify

For each approved issue, work in a dedicated branch:

1. **Fetch latest and create branch** from the target branch (`dev` if it exists, `main` otherwise):
   ```bash
   cd <repo-dir>
   git fetch origin
   git checkout dev  # or main
   git pull origin dev  # or main
   git checkout -b fix/<issue-number>-<short-slug>
   ```
2. **Implement the fix** — keep changes minimal and focused.
3. **Lint & verify:**
   - **Helm:** `helm lint ocp-dev-days-rdshw-gitops/cluster/*`
   - **YAML:** `python3 -c "import yaml; yaml.safe_load(open('file.yaml'))"`
4. **Commit** using conventional format: `fix(gitops): resolve HPA in tenant chart (#42)`

### Step 5: Open PR

```bash
gh pr create --repo rhpds/<repo> --base dev --title "fix(<scope>): <subject>" \
  --body "$(cat <<'EOF'
Closes #<issue-number>

## Summary
<Brief description of what changed and why>

## Notes
<Any caveats, follow-up work, testing considerations, or deployment impact worth calling out. Omit this section if there's nothing notable.>
EOF
)"
```

Once the user has selected issues, execute steps 3-5 autonomously — branching, fixing, committing, pushing, and opening PRs without further confirmation. Subagents may be used to work on independent issues in parallel.
