# Resolve repository instructions by target path

## What it is

A coding-agent context pattern where repository instructions are selected from the path the agent is actually working on, composed from broad to specific scope, bounded by explicit context limits, and re-resolved when tool activity moves to another part of the repository.

This is useful for monorepos where one global `AGENTS.md`/instruction file is either too vague or too large, while sibling modules need different build, style, testing or domain rules.

## Use when

Use this when:

- a repository has nested `AGENTS.md` or equivalent instruction files;
- different modules/services have different constraints;
- an agent navigates across directories during a tool loop;
- you need instruction context without injecting the whole repository handbook into every request.

## Resolution rule

For target file `a/b/c/File.java`, resolve only instructions whose directory is an ancestor of the target:

```text
repo/AGENTS.md
      |
      v
repo/a/AGENTS.md
      |
      v
repo/a/b/AGENTS.md
      |
      v
repo/a/b/c/File.java
```

A concrete precedence model demonstrated by `spring-ai-agents-md` is:

1. an instruction file applies to its directory and descendants;
2. applicable ancestor guidance accumulates;
3. closer guidance wins on conflicts;
4. explicit user instructions override file-based instructions;
5. sibling-directory instructions do not apply.

The proposed AGENTS.md clarification that informed this behavior was still a proposal at publication time, so treat the accumulation details as a tested harness policy, not as an immutable cross-tool standard.

## Resolve again after filesystem movement

Do not resolve scoped instructions only once at session start.

If the agent reads or operates on a new path, propagate the active target path through the tool result/context and re-run instruction resolution on the next model pass:

```text
model -> readFile(module-b/...)
                |
                v
       active target path changes
                |
                v
       resolve applicable instructions
                |
                v
         next model invocation
```

Otherwise an agent can carry module-A constraints into module B, or miss narrower constraints discovered later.

## Bound instruction composition

Repository-controlled instruction text is untrusted input to the context budget. Put explicit ceilings on traversal and composition.

One Spring implementation uses defaults of:

```text
ancestor directories inspected: 32
instruction documents composed: 16
single document size:           64 KB
total composed instruction:    256 KB
```

The exact values are workload-specific; the reusable rule is to have limits and make limit hits observable.

Prefer skipping an oversized document whole over silently truncating it in the middle of an instruction. If a limit changes the context the model receives, emit a visible event/result rather than quietly degrading behavior.

## Observe resolution without leaking instructions

Useful low-cardinality telemetry includes:

```text
document.state      = present | empty
document.count      = zero | one | multiple
resolution.outcome  = complete | depth-limit | document-limit | size-limit
context.size        = <numeric distribution>
```

Do not copy repository instructions, secrets, full paths or arbitrary content into metric tags. Debugging should answer “what resolution happened?” without creating a second sensitive-data store.

## Keep safety outside the instruction layer

Scoped instructions tell the model how to work; they are not an authorization boundary.

The Repository Steward example pairs instruction resolution with deterministic application rules such as:

- reject absolute paths and `..` escapes;
- reject symlinks/protected directories;
- bound reads/search/depth/result counts;
- let the model propose a patch but not approve/apply it;
- bind a proposal to a digest of the reviewed file and reject the write if the file changed.

That separation is important:

```text
instructions -> model behavior
application/sandbox -> enforced capability
```

## Spring AI implementation shape

The `spring-ai-agents-md` example exposes target-path resolution through a `ChatClient` advisor:

```java
chatClient.prompt()
    .advisors(AgentsMdAdvisorParams.target(
        Path.of("src/main/java/com/example/Example.java")))
    .user(request)
    .call();
```

Filesystem tools can propagate the path they accessed so the advisor refreshes applicable instructions during the recursive tool loop.

This exact library can be useful in a Spring AI harness, but the path-scoping pattern is independent of Spring.

## Why it is useful

It turns repository context into a deterministic retrieval problem instead of a prompt-size problem. An agent receives the instructions relevant to the code it is touching, module-local rules can override repository defaults, and the system can explain when instruction discovery was incomplete.

It also reduces a common failure mode in monorepos: a model technically has “the instructions”, but they belong to the wrong part of the tree.

## Caveats

- Verify the exact precedence semantics of the agent/client you integrate with; not every tool necessarily implements nested instruction files identically.
- More nested instruction files can become a maintenance problem. Keep broad rules high in the tree and specialize only where behavior actually differs.
- A context limit is a safety/operability mechanism, not a license to make instruction files enormous.
- Repository instructions remain potentially hostile input. Do not let them bypass tool permissions, sandboxing or side-effect approval.
- Path resolution should use normalized/canonical paths and account for symlinks before using a path as a trust boundary.

## Prototype experiment

Create a small monorepo fixture with root, backend and frontend instruction files. Test that:

1. backend work receives root + backend guidance but not frontend guidance;
2. closest conflicting guidance wins;
3. moving from backend to frontend during a tool loop changes the active instruction set;
4. oversized/deep instruction trees produce an explicit limit event;
5. metrics show resolution state without instruction contents.

## Sources

- DaShaun Carter, *Spring AI + AGENTS.md Repository Steward* (2026-08-31): https://dashaun.com/posts/spring-ai-meets-agents-md-repository-steward/
- AGENTS.md project: https://agents.md/

## Related

- [Layered scoped agent memory with stateless reasoning sessions](layered-scoped-agent-memory.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Environment-first containment for coding agents](environment-first-agent-containment.md)
