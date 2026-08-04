---
description: Cut a release of the OpenShift Dev Days Roadshow workshop — verify dev, merge, tag, update prod.yaml.
---

## Name

release-dev-days

## Synopsis

```
/release-dev-days
```

## Description

Walks through the full release checklist for the Dev Days workshop environment:
verify the dev catalog item, identify changed repos, merge dev to main (gitops),
tag changed repos with the appropriate convention, update prod.yaml in AgnosticV,
and verify the prod deployment.

Covers three independently-versioned repos (gitops, automation, showroom) with
independent cluster/tenant tagging for the gitops repo.

## Implementation

Load and execute `.claude/skills/release-dev-days/SKILL.md`.
