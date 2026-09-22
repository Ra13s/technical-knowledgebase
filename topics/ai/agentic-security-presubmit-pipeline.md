# Gate agentic security findings with localized threat models and proof

## What it is

A security-review pipeline that treats an LLM finding as a hypothesis, not a vulnerability. Scan each change with narrowly relevant threat context, then require independent structural or sandbox evidence before interrupting a developer or generating a patch.

Google describes this shape in its production pre-submit security system and open-source Mantis harness: localized threat models focus the scanner, specialized triage verifies whether the vulnerable path is reachable, reproduction grounds difficult findings, and a fix agent uses the resulting proof to propose a patch for human review.

## Use when

Use this when AI security review has enough volume that false positives, generic threat context or expensive repository-wide scans become a bottleneck.

It is especially useful for:

- pre-submit review of security-sensitive code changes;
- large repositories where full-context scanning is noisy or expensive;
- vulnerability research where findings can be reproduced in a disposable environment;
- automated patching where a fix should be tied to evidence that the original vulnerability existed.

## Pipeline

```text
code change
  -> localized threat model
       -> live code metadata
       -> dependency / call graph
       -> historical vulnerabilities
       -> domain security rules
  -> lightweight scanner
  -> candidate finding
  -> independent triage
       -> AST / structural checks
       -> call-graph reachability
       -> indexed safety rules
  -> sandbox reproduction when needed
  -> calibrated / deduplicated finding
  -> patch agent
  -> adversarial re-test of the fix
  -> human review / publication gate
```

Keep slower post-submit or nightly scanning as a second layer for vulnerabilities that only become visible across multiple changes.

## Implementation rules

### 1. Scan the change, not the whole universe

Prefer pre-submit analysis of the current diff plus the minimum surrounding context needed to evaluate it. Large one-off scans force the model to rediscover architecture and threat context repeatedly.

Construct a local context bundle from authoritative sources:

```yaml
security_context:
  changed_symbols: [...]
  callers_and_callees: [...]
  trust_boundaries: [...]
  historical_vulnerabilities: [...]
  domain_rules: [...]
```

Do not use a static threat-model document as the only source if live code structure can tell you which dependencies and entry points actually matter.

### 2. Treat scan output as a hypothesis

The first model should be allowed to over-generate moderately. The next stage has a different objective: **prove or disprove the path**.

Useful deterministic or semi-deterministic evidence includes:

- AST structure;
- call-graph reachability;
- taint/data-flow information where available;
- known framework semantics;
- pre-indexed security invariants;
- concrete caller/entry-point evidence.

Example result contract:

```json
{
  "finding": "attacker-controlled URL reaches server-side fetch",
  "entry_point": "WebhookController.create",
  "sink": "HttpClient.send",
  "reachable": true,
  "evidence": ["Controller.java:42", "Fetcher.java:91"],
  "confidence": "structurally-verified"
}
```

### 3. Reproduce ambiguous/high-impact findings in a sandbox

When the environment permits it, turn the claim into an executable acceptance criterion: unit test, mock server, exploit request, integration test or other controlled reproducer.

Define the success condition before running generated exploit code.

```text
candidate -> reproducer -> observed unsafe behavior -> accepted finding
```

A failed reproducer does **not** prove the finding false, and a successful reproducer does not automatically establish real-world exploitability. Preserve those distinctions.

### 4. Keep scanner, verifier and fixer roles independent

Avoid letting the same reasoning trace both invent and approve a finding. Use separate prompts/roles and, when worthwhile, fresh context for:

- hypothesis generation;
- triage/critique;
- reproduction;
- patching;
- fix verification.

This reduces confirmation bias and makes each stage measurable.

### 5. Patch from evidence, then attack the patch

Pass the proof/reproducer to the fix agent. After a patch is generated, rerun the original acceptance criterion and add a counter-review that looks for bypasses or incomplete fixes.

```text
proof + code -> patch -> original exploit no longer succeeds
                    -> adversarial bypass search
                    -> normal tests / static checks
                    -> human review
```

### 6. Compress repository context hierarchically

For very large repositories, maintain security-oriented summaries at file/directory/package levels and retrieve only the branches relevant to the changed code. Keep links to primary code so summaries can be checked rather than trusted blindly.

Mantis reports large token savings from hierarchical security summaries, but treat its exact percentage as workload-specific.

### 7. Measure precision at each stage

Track at least:

- candidates per change;
- fraction rejected by triage;
- fraction reproduced;
- human-confirmed true-positive rate;
- time to result;
- accepted patch rate;
- escaped vulnerabilities / false negatives on a benchmark;
- cost per confirmed vulnerability.

Google reports false-positive rates as low as 3% for some localized threat-model configurations and over 92% precision for a specialized triage agent. Those numbers demonstrate the architecture can work at Google scale; they are not portable thresholds.

## Why it is useful

The key design rule is:

> **LLMs propose security hypotheses; independent evidence earns the right to create developer work.**

This keeps the model useful for semantic discovery while moving trust to code structure, reproducible behavior and explicit review gates.

## Caveats

- Structural reachability is stronger than model confidence but weaker than complete exploitability proof.
- Reproduction requires strong sandboxing because the agent may generate exploit code or start services.
- Historical vulnerabilities and generated threat summaries can become stale; live code wins when they disagree.
- Security pipelines can create blind spots if their domain rules encode bad assumptions; periodically benchmark against known vulnerabilities.
- Google's production metrics come from its internal infrastructure and should not be copied as targets.
- The open-source Mantis repository explicitly warns that findings and patches still require security-expert verification.

## Prototype experiment

On 50 historical security-relevant changes containing known true and false findings, compare:

1. one generic security-review prompt;
2. diff + localized threat context;
3. localized scan + structural triage;
4. localized scan + triage + sandbox reproduction for surviving high-severity findings.

Measure precision, recall, comments/findings per change, latency, token cost and human verification time. Only add patch automation after the evidence pipeline has acceptable precision.

## Sources

- Google Cloud, *Changing the game: Using agentic AI to secure infrastructure code* (2026-09-18): https://cloud.google.com/blog/topics/systems/using-ai-agents-to-secure-google-infrastructure/
- Google, *Mantis* open-source harness: https://github.com/google/mantis
- Google Cloud, *Getting started with Mantis, our open-source bug finding-and-fixing harness* (2026-09-03): https://cloud.google.com/blog/products/identity-security/getting-started-with-the-mantis-harness-to-find-and-fix-bugs

## Related

- [Precision-first multi-stage AI code review](precision-first-multi-stage-code-review.md)
- [Contain coding agents at the environment boundary first](environment-first-agent-containment.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
