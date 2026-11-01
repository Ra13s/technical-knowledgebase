# Compile repository agents into constrained GitHub Actions with staged outputs

## What it is

GitHub Agentic Workflows (`gh-aw`) separates an untrusted agent's interpretation of repository evidence from **privileged GitHub writes**. A Markdown workflow with YAML frontmatter is compiled into an auditable Actions `.lock.yml`; the agent reads repository state and emits structured *intent* through a Safe Outputs MCP server. A separate trusted stage applies only allowlisted, rate-limited and vetted operations using scoped credentials. The model is not the final authorization boundary.

The strongest end-to-end case study is Microsoft Aspire's product-to-documentation propagation: a merged change in `microsoft/aspire` starts a workflow that resolves the destination branch *deterministically*, reads the diff and linked issues, drafts MDX docs in `microsoft/aspire.dev`, and requests a review from the original feature SME. In a May 3–June 2, 2026 window, the team reported 396 runs, 82 docs PRs, all 82 ultimately merged, and 44.8 hours median docs-PR merge time. This is one team's observational result under mandatory human review, **not** a guarantee for general agents or auto-merging KB content.

## Use when

- automating issue/PR intake, docs synchronization, KB updates, dependency analysis or routine maintenance;
- an agent must read untrusted issue/PR/web text but propose repository writes;
- cross-repository publication requires credentials narrower than an organization-wide PAT;
- security review requires auditable operation schemas, path restrictions, output caps and checks outside the LLM;
- using a GitHub-hosted schedule rather than an external connector whose write permission is unreliable.

## Concrete architecture

~~~text
GitHub event/schedule
  -> deterministic trigger, eligibility, target branch and stale-source checks
  -> generated, pinned Actions workflow (.lock.yml)
      -> untrusted agent in sandbox, read-only repository tools
      -> model proposes a bounded structured safe-output action
  -> independent trusted Safe Outputs job
      -> validate operation type, count, destination, base branch, changed files
      -> inspect output / threat detection / secret filtering
      -> mint scoped short-lived GitHub App credential
      -> create draft PR, never silently elevate access
  -> required CI + reviewers + branch rules
  -> authoritative merged-state verification and dedup ledger
~~~

The three separate defenses in GitHub's published design are (1) a runner/container/egress trust boundary, including API proxy and MCP gateway so an agent does not see raw credentials; (2) a compiler/configuration boundary that pins tools and allowed connectivity; (3) a plan/output boundary that buffers, validates and rate-limits writes. **No single layer is sufficient.**

## Adaptable workflow recipe

1. Install and pin `gh` and `github/gh-aw`, then author `.github/workflows/technical-kb-intake.md` with an unambiguous schedule/trigger, an explicit read-only toolset and a small `safe-outputs` contract.
2. Resolve target branch, source identity and whether input is already processed in deterministic code *before* prompting the model. Bind discovered findings to URLs and versions; do not let model text choose an arbitrary target repository or branch.
3. Configure the only allowed write as a PR on the intended repo; apply `max: 1`, `allowed-files`, `draft` according to review policy, `protected-files` and `allowed-base-branches`. Apply `title-prefix` and a branch prefix for traceability. An illustrative **fragment**, not a full deployable workflow:

~~~yaml
tools:
  github:
    toolsets: [repos, issues, pull_requests]
safe-outputs:
  create-pull-request:
    max: 1
    draft: true
    title-prefix: "[kb] "
    base-branch: main
    allowed-base-branches: [main]
    allowed-files:
      - "topics/**"
      - "intakes/**"
      - "briefs/**"
      - "sources/processed/**"
      - "README.md"
    protected-files: blocked
~~~

4. Keep the *agent stage* on `contents: read`. Configure scoped GitHub App / safe-output credential **in the trusted output job** for the actual write; do not expose the token to untrusted model tools. The repo/installation rights must permit the operation; cross-repo requires scoped installation access to both repos.
5. Compile with `gh aw compile`, review and commit **both** the source Markdown and generated `.lock.yml`. Keep generated changes subject to code review. Use `staged: true` for a dry-run that previews without applying writes when first integrating.
6. Enable mandatory CI and human approval where appropriate; a Safe Outputs `REQUEST_CHANGES` warning is not a substitute for branch protection.
7. Store a run/source key (trigger SHA, date, article URL digest) and compare with already merged/open PRs. On output failure, inspect the actual PR/ref before retry; create an issue or a reported blocker rather than blindly posting duplicates.

**Authentication caveat:** `GITHUB_TOKEN` for Copilot inference billing is documented for **organization-owned repositories with the applicable Copilot policy** (`copilot-requests: write`). This KB is a **personal** repository: do not assume that route works here. Personal-repo workflows or third-party AI engines need a correctly scoped provider credential, with no tokens embedded in Markdown or agent logs.

## Evidence, caveats and failure modes

- The Aspire implementation is unusually concrete: the cross-repo app is limited to two repositories; the safe-output handler fixes title/label/base allowlist and uses `protected-files: blocked`; SME review is mandatory; branch selection is computed from the product PR and linked issue milestones.
- It also illustrates the tradeoff: all 82 docs PRs merged *after human review*, but 396 source PRs produced only 82 docs PRs. The workflow is a classifier and a draft author, not an autonomous replacement for release reviews.
- A prompt-injected agent may still produce damaging output, bypass restricted downstream code if you give it extra write tools, or leak through channels the sandbox did not mediate. Audit actual permissions, egress, third-party actions and output paths, including symlink/path traversal.
- Security defaults, safe-output fields and pre-release compiler behavior evolve frequently. Pin a version and consult current `gh-aw` reference before copying the fragment.
- An accepted PR is not the same as merged knowledge. The final authoritative acceptance gate belongs to GitHub CI/branch rules and an observed merge.

## Prototype experiment

Deploy a separate minimal GitHub Actions agent workflow on a test repository. Feed (a) valid source evidence, (b) a duplicate, (c) hostile issue text requesting an unrelated file edit, and (d) a stale base branch. Assert the agent lacks write credentials; the safe-output stage refuses protected/out-of-scope files; at most one PR is created; repeated run checks existing PRs; CI and human-review gates remain in control. Record output rejection reasons, tool calls, cost and accepted changes. Evaluate against the current connector-based publish path without changing the canonical KB workflow first.

## Sources

- https://github.blog/ai-and-ml/generative-ai/under-the-hood-security-architecture-of-github-agentic-workflows/
- https://github.blog/ai-and-ml/github-copilot/automating-cross-repo-documentation-with-github-agentic-workflows/
- https://github.github.com/gh-aw/reference/safe-outputs/
- https://github.github.com/gh-aw/reference/threat-detection/
- https://docs.github.com/en/copilot/how-tos/github-agentic-workflows/creating-github-agentic-workflows
- https://github.blog/changelog/2026-06-11-agentic-workflows-no-longer-need-a-personal-access-token/

## Related

- [Narrow-contract background coding agent](narrow-contract-background-coding-agent.md)
- [Enforcement-bound agent approvals](enforcement-bound-agent-approvals.md)
- [Environment-first agent containment](environment-first-agent-containment.md)
- [Idempotent side effects](idempotent-agent-tool-side-effects.md)
