# Keep AI-built prototypes out of production using separate build variants

## What it is

Compile unfinished agent-generated features into a **sandbox application variant**, not the production release. Both variants share core behavior, design tokens and stable components; only the sandbox entry point imports experimental modules. This is stronger than a feature flag because prototype code must be absent from the production artifact.

Meta's October 2026 Forum engineering case study describes separate production and sandbox builds using a plugin/socket architecture. Its agents also inspect the live simulator view hierarchy to avoid acting on the wrong screen and attach recorded test/visual evidence to human-reviewed changes.

## Use when

Agents or designers need rapid UI experiments after users already depend on the app, or when an unfinished feature should not be capable of executing in the production build.

## Implementation shape

```text
shared domain core + design system
     |               |
production entry   sandbox entry
stable plugins     stable + experimental plugins
     |               |
production build   prototype build
```

For example, separate frontend entry modules:

```ts
// production.ts
import { stablePlugins } from "./stable";
bootstrap(stablePlugins);

// sandbox.ts
import { stablePlugins } from "./stable";
import { experimentalPlugins } from "./experiments";
bootstrap([...stablePlugins, ...experimentalPlugins]);
```

Define distinct bundler build targets or mobile source sets. Do not rely only on a runtime flag that still imports prototype modules.

## Verification gates

1. **Dependency graph:** assert the production entry point cannot reach files under the experiments directory, including through shared exports.
2. **Packaged artifact:** inspect the actual production bundle/APK to verify prototype modules and assets are absent.
3. **Shared behavior:** run stable API/domain tests against both variants.
4. **Real UI:** use semantic inspection of screen/view hierarchy plus screenshot, navigation and accessibility checks on the sandbox, and again after promotion to production.
5. **Promotion:** require an engineer to accept the prototype, move approved components into stable registration and rebuild/validate production. Attach build, test and recorded UI results to the PR.

A build flavor does not isolate network, databases or credentials by itself. Use separate sandbox accounts/backend permissions and review any debug control interface.

## Evidence and failure modes

Meta reports an eight-week path to core Groups feature parity, an agentic verification loop, and repeatable stress scenarios that found and reduced a memory problem. These are one team's outcomes, not causal proof of the effectiveness of separate builds. Compile exclusion can also fail if build tooling copies all assets or eagerly bundles transitively imported modules; inspect output instead of trusting configuration.

## Experiment

Create separate sandbox/prod entries for one UI feature. Intentionally add a prototype plugin. CI must confirm it appears only in the sandbox artifact; then run a recorded user path and promote the code via a review PR. Compare with a feature-flag-only release workflow.

## Sources

- https://engineering.fb.com/2026/10/15/ml-applications/how-meta-built-forum-ai-native-engineering/

## Related

- [Build agent-addressable application feedback loops](agent-addressable-application-feedback-loop.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
