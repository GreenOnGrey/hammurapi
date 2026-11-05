# AGENTS.md — hammurapi

Guide for coding agents and developers working in this repository.

## Workspace

Hammurapi is five repositories checked out side by side: `hammurapi-core` (Go backend),
`hammurapi-web` (SPA), `hammurapi` (this one: public docs, self-hosted chart, the reusable deploy
workflow), `installer` (the stand and charts) and `hammurapi-specs`
(the specifications).

- Features are specified in `hammurapi-specs/specs/HMR/<GROUP>/FTR.HMR.<GROUP>-NNNN/`. Refer to them
  as `FTR.HMR.CMN-0004`, `FTR.HMR.INFRA-0002` (old `PLT.HMR-…`, `PLT.INFRA-…`, bare `HMR.CMN-…` are
  obsolete). Deviations are written into the spec.
- Do not commit, tag or push unless asked.

## Contents

```text
docs/                    user and operator docs (English): configuration, agent, cycle, deployment, …
helm/hammurapi/          self-hosted chart (not the stand: the stand uses installer charts)
rules/                   default rules of a specification repository — keep it
.github/workflows/deploy-component.yml   reusable deploy workflow used by core and web releases
```

Every product change that adds a setting, a service or behaviour visible to operators updates
`docs/` (configuration.md lists every environment variable).

## Checks before a commit

A change is ready when every job of `.github/workflows/checks.yml` passes locally at the versions the workflow pins, plus the
matching lines below. Read the workflow for the exact commands. Install a missing tool outside the repository
at the pinned version. Name any job you could not run in your report, with the reason.

- `deploy-component.yml`: `bats tests/unit/public.bats` in `installer`, and tag this repository before bumping `DEPLOY_WORKFLOW_REF`.
- `helm/**`: `helm lint` and `helm template` with `nabu.url` and `nabu.clientId`, and check the upgrade from the previous chart version: a dropped field can come back through its deprecated twin, so set the new value explicitly.
- Always, after the checks pass: run the `code-review` skill on the diff to be committed. Fix each finding, or name it in your report with the reason it stays. Ask the user when you doubt a finding.

## No demo stack

Hammurapi requires Nabu (FTR.HMR.CMN-0007 R1): there is no compose file, installer or demo here,
and none should come back. The chart refuses to render without `nabu.url` and `nabu.clientId`.

## Reusable deploy workflow

`deploy-component.yml` is called by `hammurapi-core` and `hammurapi-web` releases through
`DEPLOY_WORKFLOW_REF` (a tag of this repository). Rules:

- Configuration and secrets reach the stand as one JSON on the stdin of SSH, built by `jq` from
  `$ENV` — never in arguments or files.
- The runner has jq 1.7: in object literals wrap computed values in parentheses
  (`secrets: ({…} + (if … end))`). `checks.yml` compiles the payload; keep that check passing.
- After a change: tag this repository, then bump `DEPLOY_WORKFLOW_REF` in core and web
  (`deploy/sync-ref.sh`). The infra unit tests run the config step of this workflow from `main`.
- The workflow and the chart are YAML: edit them by hand; do not run generic formatters.

`actionlint` (`checks.yml`) must pass; use pinned action SHAs.
