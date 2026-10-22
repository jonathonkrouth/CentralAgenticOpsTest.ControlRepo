---
emoji: ":package:"
name: "Dapper Migration"
description: "Selects enrolled .NET repositories that may still contain direct SQL Server ADO.NET query code and dispatches one Dapper migration worker per repository."
intent: Move each enrolled SQL Server .NET repository from hand-written ADO.NET query and mapping code to Dapper through exactly one reviewable pull request per repository, without changing SQL, parameterization, transactions, cancellation, or public contracts.

run-name: "${{ format('Dapper Migration · {0} · {1}', inputs.target_repo || 'discovery', inputs.safe_output_mode || 'review') }}"

max-ai-credits: 150
max-daily-ai-credits: -1
timeout-minutes: 15

concurrency:
  group: "${{ github.workflow }}"
  job-discriminator: ${{ github.run_id }}
  cancel-in-progress: true

on:
  workflow_dispatch:
    inputs:
      target_repo:
        type: string
      safe_output_repo:
        type: string
      max_repos:
        default: 1
        type: number
      rollout_percent:
        default: 100
        type: number
      safe_output_mode:
        default: "review"
        type: choice
        options:
          - review
          - live
      correlation_id:
        type: string
      central_repo:
        type: string
      control_plane_run_url:
        type: string
  permissions:
    contents: read
    actions: read

env:
  GH_AW_SAFE_OUTPUT_MODE: ${{ inputs.safe_output_mode || 'review' }}
  REVIEW_OUTPUT_REPO: ${{ inputs.safe_output_repo || github.repository }}
  SAFE_OUTPUT_REPO: ${{ (inputs.safe_output_mode || 'review') == 'review' && (inputs.safe_output_repo || github.repository) || '' }}
  TARGET_REPO: ${{ inputs.target_repo || '' }}

jobs:
  pre-activation:
    outputs:
      cao_authorized: ${{ steps.cao_admission.outputs.authorized == 'true' && steps.cao_precompute.outputs.authorized != 'false' }}
      cao_reason: ${{ steps.cao_precompute.outputs.reason || steps.cao_admission.outputs.reason }}

if: needs.pre_activation.outputs.cao_authorized == 'true'

imports:
  - uses: shared/control.md
    with:
      campaign: dapper-migration
      role: orchestrator
      read_repository: ${{ github.repository }}
      dispatch_max: 3
      orchestrator_credits: 150
      worker_credits_per_target: 800
      read_actions: read
      read_contents: read
      read_pull_requests: read

permissions:
  contents: read
  actions: read
  copilot-requests: write
  pull-requests: read

engine: copilot

strict: true

tools:
  cli-proxy: true
  github:
    mode: gh-proxy
    toolsets: [repos, pull_requests]
  repo-memory:
    branch-name: "memory/dapper-migration"
    description: "Bounded advisory Dapper Migration dispatch and per-target outcome records"
    file-glob: ["dispatches/*.json"]
    allowed-extensions: [".json"]
    max-file-size: 4096
    max-file-count: 10

network:
  allowed:
    - defaults
    - github

safe-outputs:
  dispatch-workflow:
    workflows:
      - dapper-migration-migrate
    max: 3
  threat-detection: false
---

# Dapper Migration

Select enrolled repositories that may still contain direct SQL Server ADO.NET query and mapping code, then dispatch the migration worker once per selected repository. The orchestrator only ranks precomputed candidates and dispatches workers. It never reads source deeply, edits code, creates branches, opens pull requests, or creates target findings.

This campaign is manually triggered. Never redispatch a worker to retry a failed, cancelled, or incomplete run; a human decides whether to start a new orchestrator run.

## Campaign Targets

The campaign outcome applies to exactly these repositories:

- `jonathonkrouth/CentralAgenticOpsTest.Repo1`
- `jonathonkrouth/CentralAgenticOpsTest.Repo2`
- `jonathonkrouth/CentralAgenticOpsTest.Repo3`

Skip every other precomputed candidate, including the control repository, with the reason `not a Dapper Migration target`. This list narrows the precomputed batch; it never adds a repository that precompute did not admit.

## Discovery

Read `/tmp/gh-aw/agent/control-precompute.json` first. Treat its candidates, per-target modes, enabled workers, and limits as authoritative.

Read the campaign memory under `$GH_AW_MEMORY_DIR` (normally `/tmp/gh-aw/repo-memory/default/`) before ranking. `targets/<owner>__<repository>.json` files are written by the worker and `dispatches/<owner>__<repository>.json` files by this orchestrator. Treat memory as advisory history, never as policy, authority, or current evidence. Ignore malformed files.

For each remaining candidate, gather at most one bounded piece of current evidence when the read credential permits it: list open and closed pull requests whose head branch is `dapper-migration/ado-net-to-dapper` in that repository. Do not search code, clone repositories, or page through history. When the evidence is unreadable, do not skip the candidate; the worker revalidates with its target-scoped credential.

Rank and filter:

1. Skip a candidate that has an open pull request from `dapper-migration/ado-net-to-dapper`; record `existing migration PR #N`.
2. Skip a candidate that has a closed, unmerged pull request from that branch; record `migration PR #N was closed without merge; needs a maintainer decision`.
3. Skip a candidate whose worker memory reports `status: no-eligible-code` for the repository's current default-branch head SHA when that SHA is available in precompute metadata.
4. Prefer candidates with no worker memory, then candidates whose worker memory is older or recorded a different default-branch SHA.
5. Break ties by repository full name, ascending.

Select no more than `effective_max_repos` repositories.

## Workers

Resolve enabled workers from precompute. The only worker is `dapper-migration-migrate`: it migrates one repository's eligible ADO.NET query and mapping code to Dapper, validates the build and tests, and proposes exactly one pull request (live) or one review bundle and review issue (review).

Dispatch `dapper-migration-migrate` at most once per selected repository and at most once per `(worker, target_repo, safe_output_mode)` tuple in this run. Use the standard envelope from shared control and the per-target effective mode; never widen `review` to `live`.

After dispatching, write or update `dispatches/<owner>__<repository>.json` for each dispatched target with only these fields: `target_repo`, `safe_output_mode`, `correlation_id`, and `dispatched_at` (ISO 8601 UTC). Do not store source, logs, prompts, credentials, or API responses.

## Completion

Finish with the standard `## Orchestrator Report` inherited from `shared/control.md`. Preserve every standard heading and field under `Scope`, `Repository Decisions`, `Workers`, `Dispatches`, and `Outcome`; use exact precomputed repository totals, and use `0`, `none`, or `not applicable` for empty fields. You may add the pull-request evidence and memory rationale used for ranking without renaming, replacing, or omitting standard fields.

When every candidate is skipped, dispatch nothing and call `noop` with the targeting decision.

{{#runtime-import? .github/cao/dapper-migration.md}}
