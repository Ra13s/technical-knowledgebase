# Knowledgebase contribution rules

This repository stores **reusable engineering techniques**, not interesting technical reading.

## Admission test

Before creating or expanding canonical knowledge, answer:

> Could we reasonably use this in a real development task? What exactly would we do differently?

If there is no concrete answer, do not add it to canonical knowledge.

Good candidates include:

- APIs, annotations and language/framework capabilities
- libraries and tools
- code and configuration patterns
- commands and diagnostic recipes
- implementation and migration recipes
- tests, checks and verification techniques
- coding-agent instructions, harness rules and workflows
- concrete architecture patterns with an implementable recipe
- decision rules that lead to a specific engineering action

Usually reject:

- technology trends and release-news summaries
- broad architectural principles without an implementation recipe
- predictions about how engineering roles are changing
- framework comparisons without a concrete decision or experiment
- facts that are useful to know but do not change what we would build or do

`Interesting` is not a disposition for canonical knowledge. Log it as `skipped-not-actionable` and move on.

## Canonical entry shape

Keep entries short enough to use during development. Include, where applicable:

1. **What it is**
2. **Use when**
3. **How to use it** — preferably code, config, command or prompt
4. **Why it is useful**
5. **Caveats / when not to use**
6. **Version / compatibility**
7. **Sources** — prefer primary documentation
8. **Related entries**

Organize entries by what a developer will search for later, not by newsletter, publication or intake date. One focused capability or recipe per page is usually better than a broad essay.

## Evidence

Claims must remain traceable to sources. Prefer primary documentation for API and framework behavior. Independent experiments and papers are useful for workflow techniques, but state their measured result rather than generalizing beyond the evidence.

A vendor source can establish what a vendor feature does; it is not automatically neutral evidence that the feature is superior.

## Source intake

Before processing a source, check both the legacy `sources/processed.md`, dated ledgers under `sources/processed/`, and open intake PRs. Do not process the same source twice.

For newsletters, use the newsletter as discovery and follow worthwhile linked articles or primary documentation. Extract reusable things, not the newsletter theme.

Use dispositions such as:

- `incorporated`
- `evidence-for-existing-entry`
- `skipped-not-actionable`
- `skipped-duplicate`
- `skipped-low-confidence`
- `experiment-candidate`

The source log is for deduplication; it is not the knowledgebase.

## Intake notes

Each intake run gets a human-readable note under `intakes/`. The note is a primary artifact, not an administrative ledger.

Begin with `## Synthesis`: compact editorial prose explaining the run's strongest idea, how findings connect to existing knowledge, what shared pattern emerges across independent sources, what we should now do differently, and where evidence remains weak or tradeoffs matter. A reader months later should understand why the run mattered without opening every source article.

After the synthesis, include as applicable:

- `## New things we can use now`
- `## Existing KB entries improved`
- `## Valuable agent-system designs discovered`
- `## Rejected / deferred`
- `## Experiments worth trying`
- `## Newly discovered sources worth revisiting`
- `## Source log`

Zero canonical additions is a valid result. Never add filler to make a run look productive, and do not manufacture a grand synthesis when the sources do not support one.
