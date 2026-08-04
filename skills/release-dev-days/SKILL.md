---
name: release-dev-days
description: >
  Cut a release of the OpenShift Dev Days Roadshow workshop. Walks through
  verifying dev, identifying changed repos, merging, tagging, and updating
  the prod AgnosticV config.
---

# Release: OpenShift Dev Days Roadshow

Guide for cutting a release of the workshop environment. The workshop spans
multiple independently-versioned repositories — only repos with changes
since the last release need to be tagged.

## Prerequisites

- **GitHub CLI (`gh`)** must be installed and authenticated. Used for creating
  PRs and GitHub Releases (tags with release notes).

## Repositories

| Repository | What it contains | Tag convention | Dev branch |
|---|---|---|---|
| [ocp-dev-days-rdshw-gitops](https://github.com/rhpds/ocp-dev-days-rdshw-gitops) | Helm charts for cluster and tenant bootstrapping | `cluster-vX.Y.Z` / `tenant-vX.Y.Z` | `dev` |
| [ocp-dev-days-rdshw-automation](https://github.com/rhpds/ocp-dev-days-rdshw-automation) | Ansible roles for cluster and tenant provisioning | `vX.Y.Z` | `main` |
| [ocp-dev-days-rdshw-showroom](https://github.com/rhpds/ocp-dev-days-rdshw-showroom) | Antora workshop modules/guide | `vX.Y.Z` (optional) | `main` |

The **gitops** repo uses independent tags for cluster-level (`cluster/`) and
tenant-level (`tenant/`) content because they can change at different rates and
are referenced by separate AgnosticV variables.

## AgnosticV Tag Variables

These variables in [`agnosticv/openshift_cnv/ocp-dev-days-rdshw-combined/prod.yaml`](https://github.com/rhpds/agnosticv/blob/main/openshift_cnv/ocp-dev-days-rdshw-combined/prod.yaml)
control which version of each repo is deployed to production:

| Variable | Controls | Example |
|---|---|---|
| `ocp4_workload_dev_days_rdshw_gitops_repo_tag` | Gitops cluster charts | `cluster-v1.1.0` |
| `ocp4_workload_gitops_bootstrap_repo_revision` | Gitops tenant charts | `tenant-v1.1.0` |
| `automation_tag` | Automation collection | `ocp-dev-days-1.1.0` |

Showroom uses `{{ tag }}` (resolves to `main`) in the combined common.yaml
and does not require tagging unless you want to pin a specific version.

---

## Release Checklist

### Step 1: Verify Dev Environment

Order the dev combined catalog item and run through the full workshop:

**URL:** https://catalog.demo.redhat.com/catalog/all?item=babylon-catalog-dev%2Fopenshift-cnv.ocp-dev-days-rdshw-combined.dev

Verify:

- [ ] Cluster provisions successfully (Argo CD apps all healthy)
- [ ] Tenant provisions successfully (user created in Keycloak, namespaces created)
- [ ] Dev Spaces workspace starts and extensions load
- [ ] Pipelines execute successfully
- [ ] Showroom content renders correctly with correct user/password substitution
- [ ] Developer Hub loads and shows catalog entities
- [ ] GitOps tenant applications sync and are healthy

Do not proceed until the dev environment passes all checks.

### Step 2: Fetch Remotes and Update Local Branches

Before comparing changes, ensure every local repo is up to date with its
remote. Fetch all remotes and tags, then fast-forward local tracking
branches that will be used for comparisons (e.g. `main`, `dev`):

```bash
# For each repo:
git fetch --all --tags
git checkout main && git pull
# If the repo uses a dev branch:
git checkout dev && git pull
```

Do this for **all four repos** (gitops, automation, showroom, agnosticv)
before proceeding. Stale local state leads to incorrect change detection.

### Step 3: Identify Changed Repos

For each repo, compare the last release tag to current HEAD. Only repos
with meaningful changes need merging and tagging.

**Gitops** (compare dev branch to last cluster/tenant tags):
```bash
cd ocp-dev-days-rdshw-gitops

# Check cluster-level changes
git log <last-cluster-tag>..origin/dev -- cluster/

# Check tenant-level changes
git log <last-tenant-tag>..origin/dev -- tenant/
```

**Automation** (compare to last automation tag):
```bash
cd ocp-dev-days-rdshw-automation
git log <last-automation-tag>..origin/main
```

**Showroom** (compare to last release or main):
```bash
cd ocp-dev-days-rdshw-showroom
git log <last-showroom-tag>..origin/main
```

Record which repos have changes — only those need the remaining steps.

### Step 4: Merge (Gitops Only)

The gitops repo uses a `dev` branch. Automation and showroom work directly
on `main`, so they skip this step.

- [ ] Open a PR to merge `dev` → `main` in ocp-dev-days-rdshw-gitops
- [ ] Review the diff — this is everything going to production
- [ ] Merge the PR

### Step 5: Tag Changed Repos

Create GitHub Releases (which also create the tag) using `gh release create`.
This produces a tag with release notes in one step. Determine the next
version number by looking at the previous tags and the nature of the changes:

- **Patch (Z):** Bug fixes, small tweaks, config value changes
- **Minor (Y):** New capabilities, infrastructure changes, behavior changes
- **Major (X):** Breaking changes, major rearchitecture

Propose a version for each repo and **confirm with the user** before tagging.
Show the commits and your reasoning for the bump level.

**Gitops** — create releases for whichever paths changed:
```bash
cd ocp-dev-days-rdshw-gitops

# If cluster/ changed:
gh release create cluster-vX.Y.Z --target main --title "cluster-vX.Y.Z" --notes "..."

# If tenant/ changed:
gh release create tenant-vX.Y.Z --target main --title "tenant-vX.Y.Z" --notes "..."
```

**Automation:**
```bash
cd ocp-dev-days-rdshw-automation
gh release create ocp-dev-days-X.Y.Z --target main --title "ocp-dev-days-X.Y.Z" --notes "..."
```

**Showroom** (optional — only if pinning):
```bash
cd ocp-dev-days-rdshw-showroom
gh release create vX.Y.Z --target main --title "vX.Y.Z" --notes "..."
```

### Step 6: Update prod.yaml

Edit `agnosticv/openshift_cnv/ocp-dev-days-rdshw-combined/prod.yaml` and
update **only** the tag variables for repos that changed:

```yaml
# If automation changed:
automation_tag: "ocp-dev-days-X.Y.Z"

# If gitops cluster/ changed:
ocp4_workload_dev_days_rdshw_gitops_repo_tag: cluster-vX.Y.Z

# If gitops tenant/ changed:
ocp4_workload_gitops_bootstrap_repo_revision: tenant-vX.Y.Z
```

**Do NOT modify `__meta__.deployer.scm_ref`** — that references the
agnosticd deployer version, not our workshop repos.

### Step 7: PR and Merge AgnosticV

- [ ] Commit the prod.yaml changes
- [ ] Open a PR in [agnosticv](https://github.com/rhpds/agnosticv)
- [ ] Get review and merge

### Step 8: Post-Release Verification

After the AgnosticV merge propagates:

- [ ] Order the [prod combined catalog item](https://integration.demo.redhat.com/catalog/all?item=babylon-catalog-prod%2Fpublished.ocp-dev-days-rdshw.prod)
- [ ] Verify cluster and tenant provision successfully
- [ ] Spot-check one or two workshop modules end-to-end
