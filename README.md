# cascade-example-monorepo

A worked example of a **monorepo** managed by
[cascade](https://github.com/stablekernel/cascade). One repository holds two
deliverables that version, promote, hotfix, and roll back independently, each
described as a **component** in a single manifest.

## What This Repository Shows

| Component | Path | Tag line | Ladder |
|-----------|------|----------|--------|
| `api` | `services/api` | `api-` | `dev` -> `staging` -> `prod` |
| `web` | `apps/web` | `web-` | `dev` -> `staging` -> `prod` |

Both components share the trunk, the CLI pin, and the environment ladder as
top-level defaults. Each owns its own subtree, its own tag namespace, and its own
version line. A commit under `services/api/**` advances the `api-` line and leaves
`web-` untouched, and the reverse holds for `apps/web/**`.

Cascade records each component's state under its own `state.components.<name>`
subtree, generates one workflow per component per lane, and derives a
per-component concurrency group so promoting `api` never queues behind or cancels
a `web` operation.

## Layout

| Path | Purpose |
|------|---------|
| `.github/manifest.yaml` | Pipeline definition: shared defaults plus the two-component block. |
| `.github/workflows/orchestrate-api.yaml`, `orchestrate-web.yaml` | Generated. Build and cut a `dev` prerelease per component on trunk commits under its path. |
| `.github/workflows/promote-api.yaml`, `promote-web.yaml` | Generated. Promote each component along its own ladder and publish its release. |
| `.github/workflows/cascade-hotfix-api.yaml`, `cascade-hotfix-web.yaml` | Generated. Per-component hotfix. |
| `.github/workflows/cascade-rollback-api.yaml`, `cascade-rollback-web.yaml` | Generated. Per-component rollback. |
| `.github/workflows/build-api.yaml`, `deploy-api.yaml`, `build-web.yaml`, `deploy-web.yaml` | Local build and deploy callbacks, one pair per component. |
| `.github/workflows/scenario-suite.yaml` | End-to-end driver that proves independent per-component advancement. |
| `services/api/`, `apps/web/` | Placeholder component sources so path-scoped versioning has a subtree to key on. |

## Getting Started

### Prerequisites

- A `CASCADE_STATE_TOKEN` repository secret with `contents: write` and
  `actions: write` scope, so cascade can commit state back to trunk and dispatch
  the promote and hotfix workflows.

### Triggering Pipelines

Push a change under `services/api/**` to start the `api` orchestrate run, cut an
`api-` prerelease, and deploy to `dev`. When `dev` is verified, run **Actions ->
promote-api** to advance `api` along its ladder. The same holds for `web` under
`apps/web/**` and **promote-web**. Each component moves on its own schedule.

## What the Test Driver Validates

`.github/workflows/scenario-suite.yaml` runs weekly (and on demand) and asserts:

| Job | Validates |
|-----|-----------|
| `reset` | Manifest state resets cleanly. |
| `advance-api` | An `api`-only change cuts and promotes the `api-` line to `prod`, while `web`'s state subtree and `web-` tag namespace stay empty. |
| `advance-web` | A later `web`-only change advances the `web-` line while `api`'s already-published `prod` state and `api-` tags survive untouched. |
