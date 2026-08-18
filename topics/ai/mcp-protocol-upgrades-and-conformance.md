# MCP protocol upgrades and conformance testing

## Use when

Use this recipe when implementing an MCP client/server or moving an existing implementation to a newer MCP specification while older clients may still exist.

## Compatibility-first migration recipe

Keep protocol-version adaptation at the transport/routing boundary and keep tool/domain code version-agnostic.

For the MCP `2026-07-28` transition, audit code for assumptions about:

- `initialize` / `initialized`;
- `Mcp-Session-Id`;
- sticky routing or connection-local state;
- server-to-client requests tied to a persistent transport.

The 2026-07-28 core is stateless: requests carry `MCP-Protocol-Version`, `Mcp-Method`, and when applicable `Mcp-Name`. Capability discovery moves to `server/discover`. Application state that must survive calls should be represented explicitly by an application-level identifier/handle rather than hidden in a protocol session.

A safe migration shape is:

```text
HTTP request
    |
    +-- old protocol --> existing MCP adapter ----+
    |                                             |
    +-- new protocol --> version adapter ---------+--> same tool/domain code
```

Do not fork business/tool behavior per protocol version unless the semantics genuinely differ. Make the adapter temporary and removable once the underlying Java MCP stack natively supports the desired contract.

## Test both protocol paths

During a compatibility window, keep tests for both the old and new contracts. Useful assertions include:

- legacy initialization still succeeds where promised;
- stateless discovery does not create a session;
- legacy session headers are rejected on the stateless path;
- `tools/list` exposes the expected tools and cache metadata;
- tool calls reach the same domain implementation on both paths.

## Run the official MCP conformance framework

For a server already running locally:

```bash
npx @modelcontextprotocol/conformance server \
  --url http://localhost:3000/mcp
```

Run one scenario while diagnosing a failure:

```bash
npx @modelcontextprotocol/conformance server \
  --url http://localhost:3000/mcp \
  --scenario server-initialize
```

See the scenarios the installed runner actually supports:

```bash
npx @modelcontextprotocol/conformance list
```

For a client implementation:

```bash
npx @modelcontextprotocol/conformance client \
  --command "./run-my-mcp-client" \
  --suite core
```

Put the conformance command behind a build/CI target and fail the build on conformance failures.

## When the runner lags a new protocol feature

Do not fake conformance. Keep small project-specific HTTP/integration checks for the not-yet-covered behavior and run them alongside the official suite. Delete those checks once the official conformance suite covers the same contract.

This is especially useful around specification transitions because the conformance repository distinguishes active, draft, pending and back-compat scenarios and can evolve independently of your Java framework/SDK.

## Caveats

- Pin the conformance package/version in reproducible CI rather than relying indefinitely on whatever `npx` resolves as latest.
- Protocol support in a Java framework can lag the MCP specification. An adapter can bridge that window, but it should not become a second permanent MCP implementation.
- The exact scenarios and release/draft labels in the conformance suite change; use `conformance list` and the runner's version/applicability options instead of hard-coding assumptions.

## Sources

- MCP 2026-07-28 release: https://blog.modelcontextprotocol.io/posts/2026-07-28/
- Official MCP conformance framework: https://github.com/modelcontextprotocol/conformance
- Java compatibility-first migration example: https://inside.java/2026/08/12/java-mcp-migration/
