# Spec: Manifest-Driven CDK Deployments Demo

Status: Draft
Date: 2026-11-23

## Purpose

A small, working repo that shows how to deploy an AWS CDK app through GitHub Actions using:

- a version manifest per environment to control what runs in `stg` and `prd`
- changesets for SemVer versioning and changelogs
- GitHub Environments for deploy targets, approvals and history

`dev` always runs the latest commit on `main`. `stg` and `prd` run a released version named in a manifest file. You promote a version by changing the manifest in a PR.

## Audience

Engineers evaluating this pattern for their own CDK repos. The demo must be small enough to read in one sitting.

## Scope

In scope:

- One deployable CDK app with a Lambda that reports its own version.
- Three environments: `dev`, `stg`, `prd`.
- Release flow from changeset to GitHub Release.
- Promotion by manifest PR, with automated checks.
- Rollback by manifest PR.

Out of scope:

- Publishing packages to npm.
- Multi-region deploys.
- Integration or load testing beyond a post-deploy smoke check.
- Infrastructure for the GitHub repo itself (environments, rules) beyond documenting the setup.

## Toolchain

| Tool | Requirement |
| --- | --- |
| nvm | `.nvmrc` pins the Node major version. CI reads the same file. |
| pnpm | Pinned with the `packageManager` field in root `package.json`. Workspace repo. |
| changesets | Records intended version bumps per PR. Generates versions, changelogs and git tags. |
| AWS CDK v2 | TypeScript. CLI pinned as a dev dependency. |
| GitHub Actions | All builds, releases and deploys. |
| GitHub Environments | `dev`, `stg`, `prd`. Hold per-environment variables and protection rules. |

## Requirements

### Versioning

- Every PR that changes deployable code includes a changeset. CI fails if one is missing. A PR that changes only docs or tests may add an empty changeset.
- Merging changesets to `main` opens or updates a "Version Packages" PR.
- Merging the Version Packages PR bumps the version, updates `CHANGELOG.md`, creates the git tag `demo-app@<version>` and a GitHub Release.
- Each GitHub Release carries the deployable artifact for that version.

### Release artifact

- The artifact is built once per version. `stg` and `prd` deploy the same bytes.
- Deploying a released version does not rebuild or re-bundle code.
- The artifact contains a checksum, verified before deploy.

### Manifests

- One manifest file per promoted environment: `environments/stg.yaml`, `environments/prd.yaml`.
- A manifest names the version to deploy. Nothing else is required.
- `dev` has no manifest. It tracks `main`.
- Manifests change only through PRs to `main`.

### Deploys

- `dev`: deploys on every push to `main` that changes app code.
- `stg`: deploys when `environments/stg.yaml` changes on `main`.
- `prd`: deploys when `environments/prd.yaml` changes on `main`, after approval in the `prd` GitHub Environment.
- Any environment can be redeployed manually from the Actions UI at its current manifest version.
- Only one deploy per environment runs at a time. A queued deploy waits; it is not cancelled.
- Each deploy runs a smoke check that the running app reports the expected version.
- Each GitHub Environment shows the deployed app URL.

### Promotion checks

- A PR that changes a manifest fails CI if the named version has no GitHub Release with an artifact.
- A PR that changes `environments/prd.yaml` fails CI unless the version is running in `stg`, or is older than the version running in `prd` (a rollback).
- The PR check posts a summary: current version, target version, and the changelog between them.

### Automation

- After a release, a bot opens a PR that sets `environments/stg.yaml` to the new version.
- A manual "Promote to prd" workflow opens a PR that copies the `stg` version into `environments/prd.yaml`.

### Rollback

- Rollback is a manifest PR naming an older version. No special workflow.
- Old release artifacts stay deployable.

### Security

- GitHub Actions authenticates to AWS with OIDC. No long-lived AWS keys.
- Each environment has its own IAM role. The role trusts only deploy jobs running in that GitHub Environment.
- `prd` requires a reviewer approval in the GitHub Environment.
- Manifest changes require CODEOWNERS review.

## Demo scenarios

The demo is done when each of these works end to end.

1. **Change to dev.** A PR changes the Lambda response and adds a changeset. On merge, `dev` shows the change with a version like `1.2.0+main.abc1234`.
2. **Release.** Merging the Version Packages PR creates `demo-app@1.3.0`, a GitHub Release with the artifact, and a PR bumping `stg` to `1.3.0`.
3. **Promote to stg.** Merging the stg PR deploys `1.3.0` to `stg`. The `stg` URL reports `1.3.0`.
4. **Blocked prd promotion.** A PR setting `prd` to a version not running in `stg` fails its check.
5. **Promote to prd.** Running "Promote to prd" opens a PR. On merge and approval, `prd` reports `1.3.0`.
6. **Rollback.** A PR setting `prd` back to `1.2.0` passes checks, deploys, and `prd` reports `1.2.0`.
7. **Missing changeset.** A PR that changes app code without a changeset fails CI.

## Open questions

- One AWS account for all three environments, or one per environment?
- Should `stg` auto-deploy on release, skipping the PR? The PR gives an audit trail; auto-deploy is faster.
- One deployable component, or two, to show a manifest pinning several versions?
- Should the "running in stg" check also accept versions that were in `stg` before but are no longer current?
