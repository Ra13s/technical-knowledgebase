# Git 2.56: stage only resolved conflicts, not unrelated edits

## What it is

Git 2.56 introduces `git add --resolved [<pathspec>...]`. It stages only paths currently *unmerged in the index* that no longer contain conflict markers. If a selected regular file still contains markers, the operation fails **without staging any of the selected paths**. It intentionally does not scoop up unrelated modifications the way `git add -u` or `git add -A` might.

This is a concrete safer primitive for a developer or an automated merge-conflict repair agent. It is a staging constraint, **not** a proof the conflict was resolved correctly.

## Use when

- an agent or developer manually resolves a merge/rebase/cherry-pick conflict in a dirty working tree;
- the repository contains unrelated unstaged changes that must not accidentally enter a conflict-resolution commit;
- staging only the explicitly selected conflicted paths is preferable to broad `git add`;
- building a narrow, auditable conflict-repair tool contract.

## Recipe

Require Git **2.56+** for the new option. After a merge stops with conflicts:

~~~sh
git --version
git status --short
# Edit each conflict; inspect the intended result.
git diff --check
git add --resolved             # all currently unmerged paths, all-or-none marker check
# Or: git add --resolved -- src/one-file.java
git diff --cached --check
git diff --cached --stat
git diff --cached             # REVIEW THE ACTUAL STAGED CONTENT
# Run targeted tests and then continue merge/rebase as appropriate.
~~~

For a coding agent, prefer a tool contract like:

~~~text
read conflict paths (git ls-files -u)
  -> edit allowlisted conflicted paths only
  -> git add --resolved -- <allowlisted paths>
  -> git diff --cached --check
  -> inspect staged diff + run affected tests
  -> human/CI approval gate before publishing
~~~

Never use a successful `git add --resolved` as an auto-merge approval by itself. Text with no conflict markers can still be semantically incorrect; binary changes and deletions have no textual markers and are stageable. The all-or-none rule applies to the chosen pathspec; unrelated already-staged content is not removed, so inspect the whole staged diff.

## Related Git 2.56 housekeeping

`git branch --delete-merged` can clean up *local* branches that Git considers merged into their upstream, with `--dry-run` available to inspect the candidates. Test this carefully in repositories that use squash merges: semantic inclusion of changes does not always equal commit-graph ancestry, and the command is not a substitute for verifying that the intended PR was merged. This is a convenience, not a recommended blind scheduled cleanup command.

## Evidence and limitations

Git's 2.56 release notes and current `git-add` documentation define the exact marker-check and staging semantics. GitHub's release write-up includes an example where a conflict is staged and an unrelated local edit is left unstaged. Unlike optimistic CI success or natural-language confidence, the option provides a deterministic *narrow selection* rule at the Git index boundary; human review and tests remain necessary for correctness.

`--resolved` cannot be combined with `-u` or `-A`. Guard the Git version and add the option conditionally for mixed runner fleets.

## Experiment

Create a dirty integration-test repository with one conflicted Java file and one unrelated changed file. Exercise (a) unresolved markers, (b) fixed markers, (c) a binary or deletion conflict, and (d) a pathspec. Verify exact index contents after every run, including the all-or-none failure case. Put that behavior behind a conflict-repair integration test before letting a coding agent stage merge resolutions unattended.

## Sources

- https://github.blog/open-source/git/highlights-from-git-2-56/
- https://git-scm.com/docs/git-add
- https://git-scm.com/docs/git-branch

## Related

- [Evidence-gated coding-agent edits](../ai/evidence-gated-coding-agent-edits.md)
- [Deterministic outer loop for coding-agent platforms](../ai/deterministic-outer-loop-agent-platform.md)
