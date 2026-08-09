# Modularity for AI-assisted development

## Current understanding

Code structure affects not only human maintainability but also the amount of context an AI coding agent must inspect to make a correct change. Large, poorly bounded components increase context retrieval, token usage and opportunities for unrelated assumptions.

## External evidence

A controlled refactoring experiment published on Martin Fowler's site repeatedly asked an agent to implement the same feature while a 17k-line data-access component was progressively refactored. Input-token consumption fell from roughly 160k to 27k (about 83%). The largest gain came from decomposing the large component into narrower modules; total code volume itself did not need to fall dramatically.

This is evidence that architectural boundaries have a direct machine-consumption cost in addition to their established human-maintenance cost.

## Our position

**Design for narrow change surfaces.** Modules should make it possible for both humans and agents to understand and modify a capability without loading a large unrelated portion of the system.

AI is not a reason to tolerate worse code structure because "the model can read it." Agent economics strengthen the case for modularity.

Useful characteristics include:

- high cohesion
- explicit interfaces
- low coupling
- locality of behavior and tests
- clear module ownership
- discoverable architecture documentation

## Confidence and limitations

**Confidence: medium-high.** The measured result is compelling but comes from a particular repository/task/tool setup. Exact token reductions should not be generalized as universal.

## Practical implications

When evaluating refactoring ROI for an agent-heavy codebase, include:

- reduced context/token cost
- faster agent iteration
- fewer irrelevant files considered
- easier parallel task decomposition
- reduced collision between concurrent agents

These can make refactoring economically valuable even when runtime behavior is unchanged.

## Experiments

Select one oversized component from a real repository. Establish a repeatable agent task and record context/tokens, elapsed time, changed files and review defects before and after modularization.

## Open questions

- Which structural metrics correlate best with agent context cost?
- Can repository tooling estimate "agent change surface" before refactoring?
- How much benefit comes from physical module boundaries versus documentation and semantic indexes?

## Sources

- https://martinfowler.com/articles/exploring-gen-ai/refactoring-economic-benefit.html
