# Lock GitHub Actions dependencies with a reviewed transitive lockfile

## What it is

GitHub Actions **workflow dependency lockfiles** (GA on GitHub.com on **2026-10-28**) let a workflow keep readable `uses: vendor/action@vN` references while executing the *exact commit* recorded in `.github/workflows/actions.lock`. The file covers direct and transitive action dependencies and records each upstream repository's numeric owner/repository IDs, protecting against unnoticed tag movement and repository-name takeover.

This provides an enforceable, reviewable **CI executable dependency boundary**, not a verdict that the pinned code is safe. This is a concrete complement to [agent harness supply-chain gates](../ai/agent-harness-supply-chain-gates.md): it constrains the code executed *around* coding agents (build, tests, PR publisher) rather than only their MCP/skill configuration.

## Use when

- GitHub Actions uses third-party or GitHub-hosted actions through mutable tags/branches.
- An automated coding-agent PR process builds or publishes artifacts using those actions.
- Maintaining hundreds of manual full-SHA pins and transitive dependency inventories is impractical.
- You need dependency updates to appear as explicit reviewable changes, including Dependabot PRs.

## Implement

From a clean Git checkout after reviewing the current `uses` references:

~~~bash
gh extension install github/gh-actions-lock
gh actions-lock --help
gh actions-lock                                  # onboard all workflows
# or: gh actions-lock .github/workflows/ci.yml  # pilot one workflow
git diff -- .github/workflows/
git add .github/workflows/actions.lock .github/workflows/
git commit -m "Lock GitHub Actions dependencies"
~~~

The extension writes `.github/workflows/actions.lock`; **do not hand-edit it**. The lockfile records `workflows` (what each onboarded workflow uses) and `dependencies` (original ref, resolved commit, owner ID and repository ID).

Add a **non-mutating check** on workflow and composite-action changes:

~~~bash
gh actions-lock --verify
~~~

Run this in an environment with a pinned/trusted copy of the extension. Trigger the check when `.github/workflows/**` or `**/action.yml` / `**/action.yaml` changes. Make it a required check after validating that the workflow is always scheduled, even for path-filtered PRs; otherwise use an always-present aggregate required job. Dependabot updates the matching lockfile automatically for workflows already onboarded to action version updates, but does not onboard previously unlocked workflows.

### Local actions and reusable workflows

- `uses: ./.github/actions/build` refers to checked-out workspace content and cannot be independently pinned by the extension. If that local composite action itself invokes actions, use `gh actions-lock --migrate-local-actions` to convert to pinned `$/` references; inspect its changes carefully. `--ignore-local-actions` is a conscious partial-coverage exception that can cause a later job-step failure.
- `jobs.<job_id>.uses` calling a **reusable workflow** is **not** pinned by the caller's lockfile. Pin that reusable workflow to a full commit SHA and have the called workflow's **own repository** maintain its own lockfile.
- A lockfile does **not** pin container images, scripts downloaded in `run:` commands, runner images, inputs, or secrets. Keep digest/version pinning, allowlists and reviews for these separate supply-chain surfaces.

### Operational guardrails

1. Review every resolved dependency and its source repository **before** accepting the first lockfile. A clean lockfile can faithfully pin malicious code.
2. Test a tag-move simulation in a throwaway repository: after locking, a tag update must not silently change the executed commit.
3. Verify that all workflows in a policy scope are onboarded, including workflows with no `uses` statements. GitHub's active **Require lockfile** policy will block targeted un-onboarded workflows.
4. On GitHub Enterprise Cloud, pilot that policy in **Evaluate** mode before switching to Active; branch protection and review remain separate. GitHub's GA announcement also lists Free and Team availability for the lockfile feature, while enterprise-wide policy management has narrower eligibility.
5. Do not ask an agent to silently regenerate and commit a lockfile from untrusted PR instructions. Treat the lockfile as a sensitive CI dependency change requiring independent review.

## Failure modes

- **False sense of security:** an attacker can compromise an already-pinned commit, its runtime downloads or its external services. Locking preserves identity and selected code, not the safety of its behavior.
- **Unreviewed update:** automatic Dependabot lock regeneration is convenient, but both the action version and transitive changes need review.
- **Policy lockout:** enabling enforcement before complete adoption can stop CI, including a workflow with zero action dependencies.
- **External reusable workflow gap:** the caller's lockfile provides no transitive guarantee across `jobs.uses` unless the called repository has its own lockfile.
- **Repository ref changes:** a locked commit must remain reachable from the ref recorded for it; GitHub rejects a run if reachability validation fails, requiring an explicit lock update.
- **Stale tooling:** install/upgrade the extension from a trusted source and pin its version in controlled CI; do not rely on a floating command installer in a privileged workflow.

## Validation experiment

On a low-risk repository with a representative workflow, compare traditional `uses: ...@vN` against lockfile-onboarded execution after deliberately moving a private test action's tag. Record the effective executed SHA, failure mode for missing transitive entries, behavior of `--verify`, Dependabot update diffs, and time required to audit dependency changes. Test a reusable-workflow call separately: it is a distinct authority boundary.

## Sources

- GitHub Changelog (2026-10-28), GA announcement: https://github.blog/changelog/2026-10-28-github-actions-workflow-lockfiles-are-generally-available/
- Official concepts and scope: https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles
- Official onboarding procedure: https://docs.github.com/en/actions/how-tos/secure-your-work/lock-workflow-dependencies
- Official policy rollout: https://docs.github.com/en/actions/how-tos/administer/enforce-workflow-lockfiles
- Independent coverage (2026-11-06): https://www.infoq.com/news/2026/11/github-actions-lockfiles-ga/

## Related

- [Gate coding-agent harness configuration as a supply chain](../ai/agent-harness-supply-chain-gates.md)
- [One aggregate required check for conditional GitHub Actions CI](github-actions-aggregate-required-check.md)
- [Compile repo agents into constrained GitHub Actions](../ai/compiled-agentic-workflows-safe-outputs.md)
