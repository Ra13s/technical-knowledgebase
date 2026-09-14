# Build agent-addressable application feedback loops

## What it is

An application architecture and development-harness pattern that gives coding agents a **fast structured control/observation surface** for the running product instead of forcing every validation cycle through slow screenshots, accessibility-tree discovery, manual simulator clicks or human narration.

Shopify's 2026 native Shop migration provides two production examples:

- **Tardis** exposes live native app events, logs and state plus commands that agents can send to the app; it can capture comparable screenshots/event windows from the old and new apps.
- Shopify's broader **agent-addressable architecture** separates business logic from UI so core behavior can run headlessly on desktop and be inspected/navigated through a CLI; the same control surface can remotely drive a simulator when real UI validation is needed.

The reusable rule is: when an agent can edit code in seconds but needs minutes to observe whether the edit worked, the feedback interface has become the bottleneck.

## Use when

Use this when agents repeatedly modify applications whose correctness depends on running UI or interactive state, such as:

- iOS/Android apps;
- desktop applications;
- browser applications with complex client state;
- device/control software;
- UI migrations where behavior and analytics parity matter.

It is especially valuable when multiple agent worktrees can implement in parallel but every task queues behind a human or simulator to validate navigation, state and events.

## Architecture

```text
                    +----------------------+
                    | real UI / simulator  |
                    +----------+-----------+
                               |
agent -> structured CLI/API -> app control layer
                               |
                   +-----------+-----------+
                   |                       |
          inspect state/events        navigate / act
                   |                       |
                   +-----------+-----------+
                               |
                        headless domain core
```

Keep the headless/structured path close to the same production logic used by the UI. It is a control and observation surface, not a second implementation of product behavior.

## Implementation rules

### 1. Separate business behavior from UI mechanics

Put state transitions, domain rules and data operations behind interfaces that can run without rendering a screen.

A useful target is:

```text
UI -> application/domain commands -> state/effects
CLI -> same application/domain commands -> state/effects
```

If the CLI has to reimplement the UI's business rules, the architecture has created a second source of truth.

### 2. Expose typed inspect / navigate / act commands

Prefer semantic commands over pixel coordinates.

For example:

```text
app inspect navigation
app inspect analytics --since checkpoint:checkout-start
app navigate product --id 123
app action add-to-cart --product 123
app wait state --path cart.items --count 1
```

The command names are illustrative. The principle is that an agent should be able to query the same facts a developer would inspect in a debugger without first solving computer vision.

### 3. Keep a route to the real UI

Headless execution is fast, but it cannot prove layout, accessibility, animation or real device integration.

Use the structured control surface to reach a known state quickly, then switch to simulator/device validation when the assertion is visual or platform-specific.

```text
CLI navigate quickly -> known checkpoint -> screenshot/accessibility/perf assertion
```

Shopify reports using a remote control mode so agents can drive the simulator without rediscovering each screen through screenshots/accessibility trees.

### 4. Make behavioral parity machine-comparable

For migrations or dual implementations, capture named checkpoints from both systems and compare stable behavior rather than asking a reviewer to eyeball everything.

Useful parity signals include:

- navigation destination/state;
- analytics event names and counts;
- stable payload fields;
- relationships between events and page/entity context;
- screenshots at named checkpoints;
- accessibility state;
- persisted/session state.

Explicitly ignore known nondeterminism such as timestamps or generated page UUIDs rather than normalizing the entire payload into meaninglessness.

### 5. Keep work checkpoints small and independently provable

Do not ask an agent to rewrite a whole application and validate at the end.

A checkpoint should be small enough that:

- the behavior is testable;
- visual/parity review is understandable;
- a human can review quickly;
- failure has a narrow search space.

Shopify's Helix workflow has agents propose ordered checkpoints, require tests/visual review and adversarial review, then get human approval before moving to the next committed checkpoint.

### 6. Bind approvals to the artifact that was reviewed

If a plan or checkpoint changes after approval, invalidate the old approval.

Shopify's Shop migration tied plan acceptance to a hash of the plan contents. The reusable rule is broader:

```text
approval = hash(reviewed artifact) + approver + scope
```

A materially changed plan is a new artifact and requires a new decision.

### 7. Measure feedback-loop latency explicitly

Instrument:

```text
edit -> build -> launch -> navigate -> observe -> diagnose -> next edit
```

If `navigate + observe` dominates the loop, improving model quality alone may produce almost no throughput gain. Invest in control surfaces, faster builds, state fixtures and structured observability instead.

## Why it is useful

Coding agents make implementation cheaper, which exposes previously hidden costs in the development loop. UI navigation, simulator setup, event inspection and human verification can become the new critical path.

An agent-addressable product turns those operations into tools the model can call directly. That increases autonomy without asking the model to infer application state from pixels when a deterministic source already exists.

It also improves ordinary developer tooling: a semantic CLI and structured live state are useful for debugging, tests, migrations and support, not only agents.

## Failure modes and caveats

- **Second implementation:** a headless path that duplicates product rules can pass while the real UI is broken.
- **Testing the harness instead of the app:** still validate real rendering, accessibility, device APIs and performance.
- **Overpowered debug surface:** production builds must not expose unsafe commands or private state unintentionally; authenticate/compile-gate/debug-scope the interface.
- **Semantic drift:** CLI commands and event schemas need tests/versioning as the app evolves.
- **Visual blind spots:** structured state cannot detect clipping, animation glitches or platform-native feel.
- **Agent-generated architectural drift:** fast porting still needs repository rules, static analysis, performance checks and domain-expert review.

## Prototype experiment

Choose one UI flow that agents modify often:

1. move its state transition/business logic behind a headless callable boundary;
2. expose `inspect`, `navigate` and `act` commands through a local CLI or debug API;
3. add one named visual checkpoint;
4. capture stable analytics/event output at the checkpoint;
5. run an agent task using the current screenshot/manual loop and the structured loop;
6. measure time from edit to verified feedback, human interventions and missed defects;
7. only expand the interface when the structured loop materially reduces validation latency without hiding real-UI failures.

## Sources

- https://shopify.engineering/back-to-native
- https://shopify.engineering/shop-app-migration

## Related

- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
