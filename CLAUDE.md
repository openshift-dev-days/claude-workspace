# OpenShift Dev Days Roadshow

A hands-on workshop focused on developer experience on OpenShift (4.20+),
covering Keycloak, Dev Spaces, GitOps (Argo CD), Pipelines, Developer Hub,
and AI-assisted development via LiteLLM.

## Architecture

The workshop runs on an OpenShift cluster provisioned by RHDP (Red Hat Demo
Platform). We deploy infrastructure into the base cluster, then provision
tenants on-demand as attendees sign up.

**Cluster provisioning:** AgnosticV config triggers Ansible Automation Platform,
which runs a sequence of workloads — Keycloak for auth, OpenShift GitOps,
LiteLLM for AI inference keys, and a cluster bootstrap role that deploys an
app-of-apps Argo CD Application from our gitops repo.

**Tenant provisioning:** A bridge role loops per-user workloads — creates a
Keycloak user, an Argo CD AppProject, a bootstrap GitOps Application (Helm
charts from our gitops repo), and a Showroom lab UI instance.

The combined CI (`ocp-dev-days-rdshw-combined`) handles both cluster and
tenant provisioning in a single catalog item.

## Repositories

### Workshop Content & Infrastructure

Clone these as sibling directories alongside this CLAUDE.md:

| Repository | Purpose | Branch model | Tagging |
|---|---|---|---|
| [ocp-dev-days-rdshw-gitops](https://github.com/rhpds/ocp-dev-days-rdshw-gitops) | Helm charts for cluster (`cluster/`) and tenant (`tenant/`) bootstrapping | `dev` → `main` | Independent: `cluster-vX.Y.Z`, `tenant-vX.Y.Z` |
| [ocp-dev-days-rdshw-automation](https://github.com/rhpds/ocp-dev-days-rdshw-automation) | Ansible collection with cluster bootstrap and custom roles | `main` | `ocp-dev-days-X.Y.Z` |
| [ocp-dev-days-rdshw-showroom](https://github.com/rhpds/ocp-dev-days-rdshw-showroom) | Antora-based workshop modules and lab guide | `main` | `ocp-dev-days-rdshw-X.Y.Z` |
| [agnosticv](https://github.com/rhpds/agnosticv) | RHDP catalog config — defines workloads, variables, and environment overrides | `master` | Not tagged (catalog configs reference our other repos' tags) |

### Application Repos (openshift-dev-days org)

Workshop application code and RHDH catalog entities:

| Repository | Purpose | Visibility |
|---|---|---|
| [parasol-insurance](https://github.com/openshift-dev-days/parasol-insurance) | Per-user Quarkus microservices app with catalog-info.yaml | Forked per attendee |
| [parasol-catalog-entities](https://github.com/openshift-dev-days/parasol-catalog-entities) | Shared RHDH System and Resource entities | Single shared repo |
| [module3-dev-workspace](https://github.com/openshift-dev-days/module3-dev-workspace) | Module 3 Dev Spaces workspace template | Shared reference |
| [rhdh-templates](https://github.com/openshift-dev-days/rhdh-templates) | RHDH software templates for Module 4 | Shared reference |
| [parasol-insurance-manifests](https://github.com/openshift-dev-days/parasol-insurance-manifests) | GitOps manifests for parasol-insurance deployments | Per-user repos |

### Reference Repos (RHDP upstream workloads)

Read-only, for inspecting reusable workload roles:

| Repository | Purpose |
|---|---|
| [core_workloads](https://github.com/rhpds/core_workloads) | Reusable RHDP cluster-level workload roles (includes multi_tenant_loop bridge) |
| [namespaced_workloads](https://github.com/rhpds/namespaced_workloads) | Reusable RHDP tenant-level workload roles |

## Skills & Agent Automation

This workspace includes interoperable skills (`.claude/skills/`) and subagent specs (`agents/`) for both Claude Code and Hermes Agent:

- **`.claude/skills/release-dev-days/SKILL.md`** — Cut production releases, tag repos, update `prod.yaml`, and submit AgnosticV PRs.
- **`.claude/skills/dev-days-triage/SKILL.md`** — Fetch open issues across roadshow repos, inspect code, draft fixes, and open PRs.
- **`.claude/skills/rhdp-lab-validation/SKILL.md`** — Validate provisioned OpenShift Dev Days clusters ordered from RHDP.
- **`.claude/skills/cluster-test-push/SKILL.md`** — Push local repo changes to a live cluster's GitLab for rapid testing.
- **`agents/dev-agent.md`** — Developer subagent for code fixes and feature implementation.
- **`agents/order-test-agent.md`** — Lab validation subagent for cluster health verification (`scripts/validate_dev_days_cluster.py`).

## AgnosticV Configuration

AgnosticV uses a **base + environment override** pattern for the catalog item at:
```
agnosticv/openshift_cnv/ocp-dev-days-rdshw-combined/
├── common.yaml    # Base configuration (workloads, variables, collections)
├── dev.yaml       # Development environment overrides
└── prod.yaml      # Production environment overrides (pinned release tags)
```

### common.yaml (Base Configuration)

Defines the workshop structure that applies to **all environments**:

- **`tag: main`** — Default variable used throughout. Dev uses as-is, prod overrides.
- **`automation_tag: "{{ tag }}"`** — Resolves to `main` in dev, overridden in prod.
- **`workloads`** — Cluster-level roles run **once** per provision:
  - Keycloak SSO realm setup
  - OpenShift GitOps operator + ArgoCD instance
  - LiteLLM virtual key provisioning (shared cluster-wide)
  - Cluster bootstrap (app-of-apps from gitops repo)
  - Multi-tenant bridge (loops through tenant workloads per user)
- **`tenant_workloads`** — Per-user roles run **num_users times** by the bridge:
  - Keycloak user creation
  - ArgoCD AppProject per user
  - Tenant bootstrap (GitOps Application from gitops repo tenant charts)
  - Showroom lab UI instance
- **`tenant_workload_vars`** — Per-user variable overrides evaluated each iteration
- **`requirements_content.collections`** — Ansible collections to install, pinned by `{{ tag }}`

### dev.yaml (Development Overrides)

Development environment uses **`dev` branches** and allows failures for rapid iteration:

```yaml
purpose: development

__meta__:
  deployer:
    scm_ref: main  # agnosticd deployer branch (not our repos)

# Use dev branch for gitops
ocp4_workload_dev_days_rdshw_gitops_repo_tag: dev
ocp4_workload_gitops_bootstrap_repo_revision: dev

# Provision only 1 user for fast testing
num_users: 1

# Ignore ArgoCD sync failures for rapid iteration
ocp4_workload_dev_days_rdshw_wait_for_apps_ignore_errors: true
```

### prod.yaml (Production Overrides)

Production environment uses **pinned release tags** for stability:

```yaml
# Automation collection tag (references ocp-dev-days-rdshw-automation release)
automation_tag: "ocp-dev-days-1.1.0"

# Gitops cluster and tenant charts — versioned independently
ocp4_workload_dev_days_rdshw_gitops_repo_tag: cluster-v1.3.1
ocp4_workload_gitops_bootstrap_repo_revision: tenant-v1.2.0

# Showroom content pinned to release tag
ocp4_workload_showroom_content_git_repo_ref: ocp-dev-days-rdshw-1.0.1

__meta__:
  deployer:
    scm_ref: "ocp-dev-days-1.0.0"  # agnosticd deployer version (not our repos)
```

> [!NOTE]
> Tags shown above are **illustrative examples** from a point in time. Always check
> `agnosticv/openshift_cnv/ocp-dev-days-rdshw-combined/prod.yaml` for current production
> tags and `dev.yaml` for development configuration.

### How Overrides Work

When RHDP processes an order:

1. **Loads `common.yaml`** — establishes base structure and `{{ tag }}` variable
2. **Merges environment file** — `dev.yaml` or `prod.yaml` based on catalog item environment
3. **Variable resolution** — `{{ tag }}` resolves to:
   - Dev: `main` (from common.yaml, no override)
   - Prod: Still `main` in common.yaml, but **specific variables** are overridden with release tags
4. **Collections install** — uses resolved `{{ tag }}` or explicit version strings
5. **Workloads execute** — with merged configuration

## Release Process

Use the `/release-dev-days` skill to cut a release. It walks through:
verifying the dev environment, identifying changed repos, tagging with
release notes via `gh release create`, updating `prod.yaml`, and opening
an AgnosticV PR.

See `.claude/skills/release-dev-days/SKILL.md` for the full checklist.

## Automation Preferences

- **Browser Automation**: Use Playwright over chrome-devtools MCP for web automation and validation tasks.

## RHDH Per-User Entity Pattern (RBAC)

When RHDH RBAC uses `IS_ENTITY_OWNER` to isolate per-user Components, any
shared entity (API, Resource) that has relations to multiple users' Components
will trigger "entities that couldn't be found" warnings — each user can only
see their own Component but the shared entity references all of them.

**Fix:** Create per-user entities by embedding them in `catalog-info.yaml.template`
as additional YAML documents (separated by `---`). The tenant bootstrap job's
`sed`-based template substitution handles multi-document YAML transparently.

- Use `metadata.name: <entity>-{{user_guid}}` for unique identity
- Keep `metadata.title` identical across users (e.g. "Parasol Insurance API") —
  each user only sees their own, so the display name stays clean
- Set `owner: user:default/{{user_guid}}` so RBAC grants access
- Update the Component's `providesApis` / `consumesApis` to reference the
  per-user entity name
- Remove or stop importing the shared entity from `parasol-catalog-entities`
  if it's fully replaced

The shared `parasol-catalog-entities` repo (`system.yaml`, `resources.yaml`)
still provides entities that are intentionally visible to all users (System,
Resources). Only replace shared entities that cause RBAC cross-user warnings.

## Key Conventions

### Tagging & Versioning

- **Gitops repo:** `cluster-vX.Y.Z` and `tenant-vX.Y.Z` — versioned independently
- **Automation repo:** `ocp-dev-days-X.Y.Z`
- **Showroom repo:** `ocp-dev-days-rdshw-X.Y.Z`
- **AgnosticV:** Not tagged — catalog configs reference our repos' tags via variable overrides

### Important Variables

- **`tag`:** Base variable in common.yaml (default: `main`). Controls collection versions via `{{ tag }}`.
- **`automation_tag`:** References automation collection release. Dev: `main`, Prod: pinned (e.g., `ocp-dev-days-1.1.0`)
- **`__meta__.deployer.scm_ref`:** References the **agnosticd deployer**, not our workshop repos — **never change during releases**
- **`common_password`:** Generated once per provision, shared across all roles and users

### Keycloak

- **Realm:** `sso`
- **User group:** `users`
- **User pattern:** `user1`, `user2`, ... `user{num_users}`

### RHDH Per-User Entities

For RBAC with `IS_ENTITY_OWNER` policy, create per-user entities in `catalog-info.yaml.template`:

- Use `metadata.name: <entity>-{{user_guid}}` for unique identity
- Keep `metadata.title` identical across users (display name)
- Set `owner: user:default/{{user_guid}}` for RBAC
- Embed as additional YAML documents separated by `---`
- Reference in Component's `providesApis`/`consumesApis` by per-user name
