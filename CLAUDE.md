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

Clone these as sibling directories alongside this CLAUDE.md:

| Repository | Purpose | Branch model |
|---|---|---|
| [ocp-dev-days-rdshw-gitops](https://github.com/rhpds/ocp-dev-days-rdshw-gitops) | Helm charts for cluster (`cluster/`) and tenant (`tenant/`) bootstrapping | `dev` → `main` |
| [ocp-dev-days-rdshw-automation](https://github.com/rhpds/ocp-dev-days-rdshw-automation) | Ansible collection with cluster bootstrap and custom roles | `main` |
| [ocp-dev-days-rdshw-showroom](https://github.com/rhpds/ocp-dev-days-rdshw-showroom) | Antora-based workshop modules and lab guide | `main` |
| [agnosticv](https://github.com/rhpds/agnosticv) | RHDP catalog config — defines workloads, variables, and prod/dev tags | `master` |

Optional reference repos (read-only, for inspecting upstream workload roles):

| Repository | Purpose |
|---|---|
| [core_workloads](https://github.com/rhpds/core_workloads) | Reusable RHDP cluster-level workload roles |
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

The primary config file is `agnosticv/openshift_cnv/ocp-dev-days-rdshw-combined/common.yaml`.
It defines:

- `workloads` — cluster-level roles run once (auth, gitops, litellm, bootstrap, then the multi-tenant bridge)
- `tenant_workloads` — per-user roles run by the bridge (keycloak user, gitops appproject, bootstrap, showroom)
- `tenant_workload_vars` — per-user variable overrides evaluated per iteration
- `requirements_content.collections` — Ansible collections to install (pinned by tag)

Production tags are set in `prod.yaml`. Dev uses `main` for everything via
`{{ tag }}` in `common.yaml`.

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

- **Tag format (gitops):** `cluster-vX.Y.Z` and `tenant-vX.Y.Z` — versioned independently
- **Tag format (automation):** `ocp-dev-days-X.Y.Z`
- **Showroom:** uses `main` in prod (not tagged)
- **`__meta__.deployer.scm_ref`** in prod.yaml references the agnosticd deployer, not our workshop repos — do not change it during releases
- **`common_password`:** generated once per provision, shared across all roles and users
- **Keycloak realm:** `sso`, user group: `users`
