# One aggregate required check for conditional GitHub Actions CI

## What it is

In a repository where CI jobs run conditionally, make one final `status` job the only required GitHub check. That job always appears and fails when any relevant upstream job failed or was cancelled.

This avoids two common problems:

- a path-filtered workflow is configured as required but never runs for an unrelated change, leaving the PR stuck waiting for a check that will never appear;
- a new CI job is added but someone forgets to add it to branch protection/rulesets.

## Use when

Use this pattern for:

- monorepos with frontend/backend/service-specific CI;
- expensive jobs that should run only for relevant paths;
- repositories using GitHub merge queues;
- CI where the set of checks changes often.

For a tiny repository where every CI job always runs, requiring the jobs directly is simpler.

## 1. Make component workflows reusable

Instead of giving each component workflow its own `pull_request` trigger, expose it through `workflow_call`:

```yaml
# .github/workflows/backend.yml
name: Backend

on:
  workflow_call:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: ./gradlew build
```

Do the same for the other independently runnable CI areas.

## 2. Create one top-level CI workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  merge_group:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: actions/checkout@v6
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            frontend: ['frontend/**']
            backend: ['backend/**']

  frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    uses: ./.github/workflows/frontend.yml

  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    uses: ./.github/workflows/backend.yml

  status:
    if: always()
    needs: [changes, frontend, backend]
    runs-on: ubuntu-latest
    steps:
      - name: Fail when a required upstream job did not succeed
        if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
        run: exit 1
```

In production, pin third-party actions to reviewed immutable commit SHAs according to the repository's supply-chain policy; version tags above are kept readable for the pattern example.

## 3. Require only the aggregate check

In branch protection or a repository ruleset, require the final `CI / status` check instead of each conditional component job.

Then:

- relevant component succeeds -> `status` succeeds;
- irrelevant component is skipped -> `status` still succeeds;
- component fails -> `status` fails;
- component is cancelled -> `status` fails;
- change-detection job fails -> `status` fails.

When adding another conditional workflow later, add it to the top-level caller and to the `status.needs` list. Branch protection does not need another check name.

## Merge queue requirement

If the repository uses GitHub's merge queue, the top-level workflow must also listen to `merge_group`. GitHub explicitly requires this for Actions-based required checks; otherwise the required check is not reported for the merge group and the queued merge fails.

## Why not put `paths` on a required workflow trigger?

GitHub documents that when a required workflow is skipped due to path filtering, its check can remain `Pending`, blocking the PR with “Waiting for status to be reported.” Detect paths inside the always-triggered workflow instead.

## Caveats

- `status` is only as complete as its `needs` list. Add every job whose failure should block merging.
- Decide explicitly whether cancelled jobs should block; this recipe treats cancellation as failure.
- A job that is allowed to fail should not be included as a blocking dependency without adjusting the aggregation rule.
- Reusable workflow permissions cannot be elevated above the caller's `GITHUB_TOKEN` permissions; define caller permissions deliberately.
- If you use a third-party path-filter action, pin and update it like any other CI dependency. A small first-party change detector is another option for high-assurance repositories.

## Sources

- Marc Philipp, *One required check to rule them all* (2026-08-10): https://marcphilipp.de/blog/2026/08/10/one-required-check-to-rule-them-all/
- GitHub: troubleshooting required status checks: https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks
- GitHub: reusable workflows: https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations
- GitHub: `merge_group` event: https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#merge_group
