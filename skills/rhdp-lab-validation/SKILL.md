---
name: rhdp-lab-validation
description: >
  Validate provisioned OpenShift Dev Days clusters ordered from RHDP. Checks
  nodes, Argo CD GitOps apps, routes (Showroom, Keycloak, RHDH), and pod health.
---

# Skill: RHDP Lab Order & Cluster Validation

Use this skill after ordering `openshift-cnv.ocp-dev-days-rdshw-combined.dev` from RHDP to verify that cluster and tenant workloads converged successfully.

## Prerequisites

- `oc` CLI installed and authenticated to the target cluster (`oc login https://api... -u kubeadmin -p ...`).

## Automated Validation Command

Run the built-in validation script:
```bash
python3 /Users/eshortis/workspaces/dev-days/scripts/validate_dev_days_cluster.py "oc login <api-url> -u <user> -p <pass>"
```

## Validation Checklist

1. **Cluster Nodes:** All nodes in `Ready` status.
2. **Argo CD App-of-Apps:** Every application in `openshift-gitops` namespace is `Healthy` and `Synced`.
3. **Key Routes Active:**
   - Showroom Lab Guide (`oc get route -n ocp-dev-days-rdshw-showroom`)
   - Keycloak SSO (`oc get route -n keycloak`)
   - Red Hat Developer Hub (`oc get route -n ai-rhdh` / `rhdh`)
