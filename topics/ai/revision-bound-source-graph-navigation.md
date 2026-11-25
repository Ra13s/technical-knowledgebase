# Give coding agents bounded source-graph navigation before broad search

## What it is

For repository-scale coding tasks, supplement grep/glob with a **revision-aware, locally built source graph**: indexed definitions, call/use relationships, route mappings and change impact. Give the agent compact `path:line` evidence to verify in actual source rather than a free-text summary that may have gone stale.

Entire's open-source **Entire Graph** provides a concrete implementation: tree-sitter parsing, ranked lexical/structural retrieval, callers/callees/type relations, semantic diffs, an explicit coverage/health report and immutable-HEAD versus working-tree query modes. No hosted model call is required to build its source graph. The useful pattern is **retrieve -> verify source -> test**, not “believe a graph and edit.”

## Use when

- a coding agent must locate behavior in an unfamiliar Java/Kotlin or multi-language repository;
- a multi-file migration requires impact analysis beyond text matches;
- planning needs callers, API routes, inheritance, type consumers or changed symbol dependencies;
- the same repository is scanned repeatedly by short-lived agents;
- grep results are huge, retrieval consumes substantial context and the cost is measurable.

## Prototype using Entire Graph

Requirements per the current open-source README: Git >=2.36, Entire CLI >=0.10.0. Review and pin the binaries before installation in controlled environments; do not blindly execute vendor installation scripts in CI.

~~~bash
# After installing a vetted Entire CLI:
entire plugin install graph
entire graph version
entire graph capabilities --json
entire graph setup --repo .           # review generated AGENTS.md, CLAUDE.md, .entire/agent-guide.md
entire graph health --repo . --json   # parser coverage, failures, freshness

# Agent workflow example, adapting symbols and paths to the project:
entire graph query --profile full "where is request authorization enforced?"
entire graph impact --symbol AuthorizationService.check
entire graph diff --repo /absolute/repo/root --base main --head HEAD -- src/
~~~

The setup operation modifies one managed block in the repository's agent instructions. Review the generated guidance before committing. Alternatively use the direct query commands without automatically modifying repo instructions.

A simple tool contract for any source-graph implementation:

~~~text
graph_query(task, ref_or_worktree, max_results, max_bytes)
  -> {revision, parser_coverage, ranked[{path,line,symbol,relation,confidence}]}

agent reads the top source ranges at the SAME revision
  -> checks unindexed/dynamic behavior with grep, tests and runtime inspection
  -> edits only after actual code/contract is confirmed
  -> re-runs targeted tests and fresh impact analysis after editing
~~~

**Cache identity matters:** a committed-tree graph for `HEAD` is not proof about uncommitted edits. Entire Graph queries can use a working-tree view by default, while opt-in Claude first-prompt injection explicitly uses `--head` (committed tree) and can miss uncommitted work. In linked worktrees, use the exact checkout root for `--repo` on range commands; do not accidentally query another agent's tree.

## How to test whether the layer is worthwhile

Do not adopt a semantic graph because its README reports a high benchmark score. The vendor reports LoCoMo long-conversation retrieval of 94.74% and token savings up to 71% in selected scenarios, but LoCoMo is **not** a coding correctness benchmark. Its own paired experiments on first-prompt graph injection found differences too small to be statistically conclusive.

Use a local ablation with real repo questions:

1. Freeze 30 Java/Kotlin code-navigation tasks with known ground-truth paths/invariants; include dynamic wiring, reflection, Spring annotations, generated sources and renamed symbols.
2. Compare grep/ripgrep baseline, source graph as explicit tool, and capped prompt injection **on the same task set and model**.
3. Run each question from a committed SHA and again with meaningful uncommitted worktree modifications.
4. Score top-k true-location recall, incorrect impact links, first-use latency, p95, token cost, actual successful edits and tests, and time spent verifying false leads.
5. Break the parser intentionally or exclude a module; require coverage/warnings to be visible rather than silently treating missing edges as zero callers.

## Compiler-resolved Java transformations (Dukehoff MCP, 2026-11-25)

When the task changes a Java symbol rather than just locating it, consider the open-source Dukehoff MCP server (Apache NetBeans/javac-based, JDK 17+, Maven/Gradle/Ant). Install a vetted pinned release into the sandbox; generate the client registration with `dukehoff-mcp --config codex` (or the supported client name). The project documents tool group selection, e.g. `--config claude --tools core,refactor`. Prefer read-only tools in planning; mutation tools must be constrained to the candidate worktree and require the same write authority as any editor.

- Start with `workspace_status` to confirm project roots/index readiness, then `find_usages` (optionally grouped by caller), `outline`, and `diagnostics`.
- Use `rename` or `change_signature` for overloaded/overridden Java methods; they use the refactoring engine and write files. Inspect the returned diff and remaining string references; reflection, Spring configuration, generated code, serialization names and external consumers still need separate checks.
- For structural migrations, `apply_rule` supports dry runs, path scope and rule packs (junit5, jakarta, modernize). Preview and inspect affected files before applying; finish reported manual edits. Do not treat arbitrary model-authored rewrite rules as trusted executable configuration.
- After mutations, run git diff, compile and real targeted tests. An IDE's diagnostics are not a full build or a security check. Keep tool processes isolated per worktree or verify their exact workspace before every mutation.

The maintainer's September eval on a NetBeans checkout reported 344 seconds to cold-index 158 projects and approximately 13 seconds for one semantic usage query; those are vendor measurements on one setup, not proof of general speedup. Compare 15 paired tasks with same model (overloads, overrides, multi-module rename and reflection/config cases). Record cold/warm latency, changed incorrect references, actual build/tests, review time and missed dynamic uses. This is an extension of the graph experiment, not a replacement for grep or independent verification.

Sources (October 18 release and implementation): https://dukehoff.com/en/blog/dukehoff-mcp-for-java/ ; https://github.com/dukehoff-gmbh/dukehoff-mcp ; https://github.com/dukehoff-gmbh/dukehoff-mcp/blob/main/eval/RESULTS.md

## Failure modes

- Graph completeness varies by language and framework; Java reflection, Spring dependency injection, generated bindings and dynamic dispatch can hide edges. **Missing graph edges do not prove absence of callers.**
- Cached graph for stale ref/worktree yields plausible but wrong locations. Always bind query output to revision and read the current file before editing.
- Parsing many large repos can cost CPU/storage, and prewarming the committed tree may not warm default working-tree queries.
- A source graph is distinct from conversation/organization memory; do not conflate symbol intelligence with durable agent state.
- A tool call can cost an entire extra model turn. A capped early context injection can help or hurt depending on prompt and task class; measure rather than mandate.
- Agent instructions installed by a plugin are executable process configuration from a trust perspective; review under the harness supply-chain gate.

## Sources

- Entire Graph implementation, current README and CLI examples: https://github.com/entireio/entire-graph
- Entire CLI and plugin interface: https://github.com/entireio/cli
- Entire Graph command, coverage and benchmarks docs: https://github.com/entireio/entire-graph/tree/main/docs
- Independent discovery of Entire's agent-centric source tooling (2026-11-08): https://www.infoq.com/news/2026/11/entire-git-forge-coding-agents/

## Related

- [Layered scoped agent memory with stateless reasoning sessions](layered-scoped-agent-memory.md)
- [Path-scoped repository instructions](path-scoped-repository-instructions.md)
- [Ablate coding-agent harness scaffolding when models change](ablate-agent-harness-scaffolding.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
