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

See `skills/release-dev-days/SKILL.md` for the full checklist.

## Key Conventions

- **Tag format (gitops):** `cluster-vX.Y.Z` and `tenant-vX.Y.Z` — versioned independently
- **Tag format (automation):** `ocp-dev-days-X.Y.Z`
- **Showroom:** uses `main` in prod (not tagged)
- **`__meta__.deployer.scm_ref`** in prod.yaml references the agnosticd deployer, not our workshop repos — do not change it during releases
- **`common_password`:** generated once per provision, shared across all roles and users
- **Keycloak realm:** `sso`, user group: `users`
