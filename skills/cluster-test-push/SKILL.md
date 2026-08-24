---
name: cluster-test-push
description: >
  Push local repo changes to a live RHDP cluster for testing. Covers
  three levels: cluster-level gitops (Argo CD apps), upstream parasol/
  repos on cluster GitLab, and per-tenant user repos with template
  variable substitution.
---

# Skill: Push Changes to Live RHDP Cluster for Testing

Use this skill to push local changes to a live RHDP cluster for rapid testing without waiting for a full CI/deploy cycle. Covers three levels of changes:

1. **Cluster-level gitops** — Helm charts in `ocp-dev-days-rdshw-gitops` (developer-hub config, gitlab init, tenant bootstrap, etc.)
2. **Upstream `parasol/` repos** — base repos on cluster GitLab that tenant bootstraps fork from (`parasol-insurance`, `parasol-catalog-entities`, `rhdh-templates`, `devfiles`)
3. **Per-tenant user repos** — individual user forks with rendered template variables

## Prerequisites

- `oc` CLI logged into the target cluster
- `curl` and `jq` available

## Repo Mapping

### Parasol app repos (cluster GitLab)

| Local directory | Cluster GitLab project | Has templates? | Per-user fork? |
|---|---|---|---|
| `parasol-insurance/` | `parasol/parasol-insurance` | Yes (`catalog-info.yaml.template`, `devfile.yaml`) | Yes (`<user>/parasol-insurance`) |
| `parasol-insurance-manifests/` | `parasol/parasol-insurance-manifests` | No | Yes (`<user>/parasol-insurance-manifests`) |
| `parasol-catalog-entities/` | `parasol/parasol-catalog-entities` | No | No (shared) |

### Gitops charts (GitHub, deployed via Argo CD)

| Local directory | Argo CD app | Scope |
|---|---|---|
| `ocp-dev-days-rdshw-gitops/cluster/` | `app-of-apps` children | Cluster-level (RHDH, GitLab, pipelines, etc.) |
| `ocp-dev-days-rdshw-gitops/tenant/` | `bootstrap-tenant-<user>` children | Per-tenant (parasol-insurance app, namespaces) |

## Workflow

### Step 1: Discover Cluster Credentials

```bash
# GitLab host
GITLAB_HOST=$(oc get route gitlab -n gitlab-system -o jsonpath='{.spec.host}')

# GitLab root PAT
GITLAB_TOKEN=$(oc get secret root-user-personal-token -n gitlab-system -o jsonpath='{.data.token}' | base64 -d)

# Cluster subdomain (for template variable substitution)
CLUSTER_SUBDOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')

# Quay host (needed for devfile/template variable substitution)
QUAY_HOST=$(oc get route quay-quay -n quay -o jsonpath='{.spec.host}' 2>/dev/null)
```

Enumerate existing tenant users and GitLab projects:

```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/users?per_page=100" --insecure | jq '.[] | {id, username}'

curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/projects?per_page=100" --insecure | jq '.[] | {id, path_with_namespace}'
```

### Step 2: Identify Changes

Determine which local repos have changes to push. Present a summary to the user and ask:

1. Which repos/files to push
2. Which existing tenant user(s) to update (for repos with per-user forks)
3. Whether cluster-level gitops changes need to be applied

### Step 3: Push to Upstream `parasol/` Repos (Cluster GitLab)

Update the base repos in the `parasol/` group so future tenant bootstraps use the new code.

Use the GitLab files API (always with `--insecure` for self-signed certs):

```bash
# Look up project ID
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/projects/parasol%2Fparasol-insurance" --insecure | jq '.id'

# Update an existing file
curl -X PUT --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/projects/<id>/repository/files/<url-encoded-path>" \
  --insecure --form "branch=main" --form "content=<content>" \
  --form "commit_message=chore: update <file> for testing"

# Create / Delete: same structure with POST / DELETE
```

> [!IMPORTANT]
> For `parasol/parasol-insurance`: push the raw template file (`catalog-info.yaml.template`) with `{{user_guid}}` and `{{cluster_subdomain}}` placeholders intact. Never rename or substitute variables in the upstream repo — the tenant bootstrap job handles that.

> [!NOTE]
> The cluster-level `initialize-gitlab` Job (`gitlab` namespace, sync wave 3) imports repos from GitHub at provision time. Re-running it does NOT update existing repos — the import task accepts 400/409 ("already taken") and skips. For updating content in existing `parasol/` repos, always use the GitLab files API directly.

### Step 4: Re-run GitLab Templates Job (if needed)

A separate `gitlab-templates` Job (sync wave 4) clones specific files from `parasol/` repos and substitutes cluster-specific variables:

| Repo | File | Variables substituted |
|---|---|---|
| `parasol/parasol-insurance` | `devfile.yaml` | `{{gitlab_host}}`, `{{cluster_subdomain}}`, `{{quay_host}}` |
| `parasol/rhdh-templates` | `templates/parasol-dev-environment/template.yaml` | `{{gitlab_host}}`, `{{cluster_subdomain}}`, `{{quay_host}}` |

If you pushed changes to any of these files in step 3, re-run the templates job to apply cluster-specific substitutions:

```bash
# Delete the completed job — Argo CD auto-heal will recreate and re-run it
oc delete job gitlab-templates -n gitlab-system
```

This job IS idempotent — it clones, runs `ansible.builtin.replace`, commits, and pushes. Safe to re-run. Argo CD's auto-heal detects the missing Job and recreates it automatically.

If you only changed files not in the templates list (e.g. `catalog-info.yaml.template`), skip this step.

### Step 5: Update Existing Tenant User Repos

For each selected tenant user, push the rendered version of changes to their forked repos.

**For templated files** (`catalog-info.yaml.template` → `catalog-info.yaml`):

1. Read the local template content
2. Apply variable substitutions:
   | Placeholder | Replace with |
   |---|---|
   | `{{user_guid}}` | tenant username (e.g. `user1`) |
   | `{{cluster_subdomain}}` | cluster apps domain from step 1 |
3. Push the rendered content as `catalog-info.yaml` (not `.template`) to the user's repo

**For non-templated repos** (e.g. `parasol-catalog-entities`): these aren't forked per-user, so step 3 already handled them.

### Step 6: Apply Cluster-Level Gitops Changes (if needed)

For changes to Helm charts in `ocp-dev-days-rdshw-gitops`, the Argo CD app-of-apps points to a specific Git ref (tag or branch). Options:

Push changes to the gitops repo on GitHub, then Argo CD picks them up (auto-sync or manual sync depending on the app config):

```bash
# Check current target revision
oc get application app-of-apps -n openshift-gitops -o jsonpath='{.spec.source.targetRevision}'

# If it tracks a branch (e.g. dev/main), push to that branch on GitHub
# Argo CD auto-sync will detect the change and apply it

# To force an immediate sync
oc get application <app-name> -n openshift-gitops -o name | \
  xargs -I{} oc patch {} -n openshift-gitops --type merge -p '{"operation":{"sync":{}}}'
```

> [!WARNING]
> Do not edit ConfigMaps or other Argo CD-managed resources directly on the cluster — auto-heal will revert them. All cluster-level gitops changes must go through the Git repo.

### Step 7: Verify

Confirm pushed files are correct:

```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/projects/<id>/repository/files/<path>/raw?ref=main" --insecure
```

For RHDH catalog changes, suggest the user check Developer Hub — the GitLab discovery provider re-scans periodically (usually within a minute or two).

## Important Notes

- **TLS**: Demo clusters use self-signed certs. Always use `--insecure` with `curl`.
- **Scope**: Only updates repos and users you explicitly select. Other existing tenants keep their old content.
- **No tenant setup job re-runs**: The per-tenant setup job (`tenant-<user>-setup`) is not idempotent — `git push --mirror` overwrites user repos and the template rename step fails on re-run. For content updates, always push directly via the GitLab files API.
- **GitLab templates job IS safe to re-run**: Unlike the tenant setup job, `gitlab-templates` is idempotent (clone → replace → push). Re-run it after pushing changes to `devfile.yaml` or `template.yaml` in upstream repos.
- **Branch protection**: If a push fails with 403, the `main` branch may be protected. Unprotect first:
  ```bash
  curl -X DELETE --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "https://$GITLAB_HOST/api/v4/projects/<id>/protected_branches/main" --insecure
  ```
