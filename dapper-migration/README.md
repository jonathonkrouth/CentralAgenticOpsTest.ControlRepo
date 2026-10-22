# Dapper Migration

Migrates direct SQL Server ADO.NET query and mapping code to
[Dapper](https://github.com/DapperLib/Dapper) in each enrolled .NET repository.
Each repository gets at most one draft pull request. The migration keeps SQL
text, parameterization, transactions, cancellation, public API contracts, and
behavior unchanged.

The campaign is review-first. Generated policy keeps every target in `review`.

## Targets

- `jonathonkrouth/CentralAgenticOpsTest.Repo1`
- `jonathonkrouth/CentralAgenticOpsTest.Repo2`
- `jonathonkrouth/CentralAgenticOpsTest.Repo3`

The control-plane scope also includes the control repository. The orchestrator
skips it, and the worker returns `noop` for any repository not listed here.

## Why a campaign

Each repository needs its own migration, validation, run history, and pull
request. That calls for one orchestrator that selects targets and one
independently dispatchable worker per repository, rather than a single fleet
run.

## Workflows

| Workflow | Role | Trigger | Writes |
| --- | --- | --- | --- |
| [`dapper-migration`](../.github/workflows/dapper-migration.md) | Orchestrator | `workflow_dispatch` only | `dispatch-workflow` (max 3) and advisory memory |
| [`dapper-migration-migrate`](../.github/workflows/dapper-migration-migrate.md) | Worker `migrate` | Dispatched by the orchestrator, or a manual `debug` run | Live: one draft PR in the target. Review: one review bundle artifact and one review issue in the control repository |

The campaign has no schedule. Neither workflow retries. A failed or incomplete
worker run waits for a human to start a new orchestrator run.

## Behavior

### Orchestrator

- Reads CAO precompute and the `memory/dapper-migration` branch.
- Skips repositories outside the target list, repositories that already have an
  open or closed-unmerged PR from `dapper-migration/ado-net-to-dapper`, and
  repositories that the worker last recorded as having no eligible code at the
  same default-branch SHA.
- Dispatches `dapper-migration-migrate` at most once per target, mode, and run,
  within `max_repos` (default `1`).

### Worker

1. **Duplicate check.** Returns `noop` when the target already has an open
   migration PR, a closed-unmerged migration PR, or a stale
   `dapper-migration/ado-net-to-dapper` branch. In review mode it also returns
   `noop` when an open review issue already covers the target.
2. **Eligibility.** Returns `noop` when no production ADO.NET command or reader
   code remains. Out-of-scope patterns stay unchanged and are listed, including
   `SqlBulkCopy`, TVPs, `SqlDependency`, `SequentialAccess` streaming, EF, and
   generated code.
3. **Migration.** Keeps SQL byte-for-byte. Uses `DynamicParameters` with the
   original `DbType`, size, and direction. Uses `CommandDefinition` with the
   original `CancellationToken`, keeps the same transaction, and uses
   `QueryFirstOrDefaultAsync` for single-row reads. It keeps scalar conversion,
   leaves global Dapper settings alone, and adds no public API changes. It adds
   one `Dapper` package reference and keeps every existing test assertion.
4. **Validation.** Runs `dotnet restore`, `build`, and `test` as a baseline
   before editing and again afterward. Tests that need SQL Server are reported
   as `not run` with the missing prerequisite. A migration-caused failure is
   reverted and reported as incomplete, and no PR is created.
5. **Output.**
   - **Live:** one draft PR from the fixed branch
     `dapper-migration/ado-net-to-dapper`, titled
     `[dapper-migration:migrate] Migrate ADO.NET data access to Dapper` and
     labelled `dapper-migration` and `dapper-migration:migrate`.
   - **Review:** a `publish_review_bundle` artifact containing `changes.patch`
     and validation details, plus one deduplicated issue in the control
     repository.

### Duplicate safeguards

| Layer | Safeguard |
| --- | --- |
| Orchestrator | Singleton concurrency, one dispatch per `(worker, target, mode)`, `dispatch-workflow.max: 3`, no retries, no schedule |
| Worker concurrency | One run per target. `cancel-in-progress: false` queues a second dispatch instead of cancelling an in-flight PR |
| Handler | `create-pull-request`: `max: 1`, `allowed-branches: [dapper-migration/ado-net-to-dapper]`, `preserve-branch-name: true`, `recreate-ref: false` (an existing branch is never recreated), `fallback-as-issue: false`, `allowed-repos` equal to `target-repo` (one repository per run) |
| Handler | `create-issue`: `deduplicate-by-title: true`, stable title, `max: 1`, `expires: 14d` |
| Prompt | Checks for an existing PR, branch, review issue, or body marker `<!-- dapper-migration:migrate target=OWNER/REPO -->` |

### Never

The campaign never merges a PR, enables auto-merge (left at its `false`
default), pushes to a default branch, force-pushes, or writes outside declared
safe outputs. Agent GitHub tools are read-only.

`create-pull-request.expires` is not set because gh-aw v0.92.6 only supports PR
expiry for same-repository PRs.

## Safety capabilities

| Item | Orchestrator | Worker |
| --- | --- | --- |
| Agent permissions | `contents`, `actions`, `pull-requests`: read; `copilot-requests: write` | `contents`, `actions`, `issues`, `pull-requests`: read; `copilot-requests: write` |
| GitHub tools | `gh-proxy` read: `repos`, `pull_requests` | `gh-proxy` read: `repos`, `pull_requests`, `issues` |
| Network | `defaults`, `github` | `defaults`, `github`, `dotnet` (NuGet) |
| Runtime | — | .NET SDK `10.0.x` through `runtimes.dotnet` |
| AI credits per run | 150 | 800 |
| Timeout | 15 min | 45 min |
| Threat detection | `false` (CAO contract for dispatchers) | gh-aw default |
| Target credential | Shared control (`read_repository` = control repo) | Shared control (`read_repository` = `target_repo`) |

The worker allows changes only to `**/*.cs`, `**/*.csproj`,
`**/Directory.Packages.props`, `**/Directory.Build.props`, and
`**/packages.lock.json`, with at most 60 files and 1 MB per patch. Changes to
protected manifests get a `REQUEST_CHANGES` review (`protected-files: request-review`).

## Policy

The `dapper-migration` entry in
[`.github/workflows/cao.json`](../.github/workflows/cao.json) sets campaign mode
`review`, `max-repositories: 3`, and `rollout-percent: 100`. It sets each of the
three targets to `review` and declares the worker
`migrate` → `dapper-migration-migrate`.

### Review-mode proof

In review mode:

- the orchestrator keeps every target in `review`;
- `create-pull-request.target-repo` resolves to the review repository, not the
  target;
- the worker prompt forbids `create_pull_request`;
- the target is only read, and edits stay in the `target/` checkout.

A review run therefore cannot change a target repository.

### Authorizing live PR creation

Live mode needs all of the following. Each can only narrow the others:

1. **CAO policy.** Change each target's mode from `review` to `live` in
   `.github/workflows/cao.json`. Leave the campaign default at `review` and add
   no worker `max-mode`.

   ```diff
            "targets": {
              "jonathonkrouth/CentralAgenticOpsTest.Repo1": {
   -            "mode": "review"
   +            "mode": "live"
              },
              "jonathonkrouth/CentralAgenticOpsTest.Repo2": {
   -            "mode": "review"
   +            "mode": "live"
              },
              "jonathonkrouth/CentralAgenticOpsTest.Repo3": {
   -            "mode": "review"
   +            "mode": "live"
              }
            },
   ```

2. **Credential reach.** In PAT mode,
   `GH_AW_GITHUB_WRITE_PAT_REPOSITORIES` must map each target to a write PAT
   secret with `contents: write` and `pull_requests: write`. Likewise,
   `GH_AW_GITHUB_READ_PAT_REPOSITORIES` must map each target to a read PAT
   secret. `gh cao setup auth token` configured these maps.
3. **Request.** Dispatch the orchestrator with `safe_output_mode=live`. A
   request can only narrow the policy.
4. **Compiled capability.** The worker already declares `create-pull-request`
   targeting `inputs.target_repo` in live mode. No workflow change is needed.

Optional: the CAO operations guide recommends documenting a `target-authority`
entry in each target repository's `.github/workflows/cao.json`. Admission does
not read target files. Adding the entry changes the target repositories, so it
is not applied here:

```json
{
  "$schema": "https://raw.githubusercontent.com/githubnext/gh-cao/main/.github/workflows/shared/cao.schema.json",
  "version": 1,
  "target-authority": {
    "campaigns": {
      "dapper-migration": {
        "authority": "jonathonkrouth/CentralAgenticOpsTest.ControlRepo"
      }
    }
  }
}
```

Promote one target at a time. Prove review mode, then live mode, on the same
single target (`max_repos=1`) before widening.

## Operational value

Deferred. The worker outcome is measurable: eligible ADO.NET call sites
remaining per target, and the migration PR state. However, no deterministic
evaluator or grader has been added yet. Add one with the `add-operational-value`
skill after the worker is stable, and adopt the worker and evaluator together.

## Steering

Optional control-owned guidance can go in `.github/cao/dapper-migration.md`.
Both prompts load it as a trailing optional `runtime-import`. The file is
intentionally not created. Steering cannot widen authority.

## Prerequisites and limitations

- The targets are `net10.0`. The worker installs SDK `10.0.x`. gh-aw's runtime
  dependency cooldown may delay a Dapper version published in the last three
  days.
- The agent cannot start a SQL Server instance or containers. SQL Server
  integration tests are reported as not run. The current target unit tests use
  fake ADO.NET providers and need no database.
- Package-completeness tests are not available in this control repository,
  which has no test harness. Validation relies on `gh aw compile --strict` and
  the `CAO validation` workflow.
