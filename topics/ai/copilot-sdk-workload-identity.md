# Give unattended Copilot SDK agents a workload identity

## What it is

Use **organization-owned, short-lived credentials** for autonomous Copilot SDK execution rather than borrowing an engineer's logged-in Copilot session or personal access token. GitHub introduced server-to-server authentication for organization-installed GitHub Apps on **2026-10-27** (public preview). GitHub Actions in an organization-owned repository can instead use the built-in `GITHUB_TOKEN` with a dedicated Copilot permission.

This is an **authentication and cost-attribution primitive**, not permission to push code, merge PRs or call arbitrary external tools. Those side effects need separately scoped repository/service credentials and their own authorization gates.

## Use when

- running a persistent internal coding agent, CI-based code reviewer or noninteractive Copilot SDK worker;
- agent jobs must continue when an individual engineer leaves or signs out;
- an organization needs model allowlists, centralized spending controls, suspension and app-attributed usage;
- comparing the cost of running Copilot SDK from GitHub Actions versus an external workflow engine.

## Select the identity path

| Execution environment | Credential | Minimum product-specific controls |
|---|---|---|
| GitHub Actions in an organization-owned repository | Built-in `GITHUB_TOKEN` | `copilot-requests: write`, organization policy allowing billed Copilot CLI usage |
| External service or CI | GitHub App **organization installation** token | Organization Copilot Requests permission **Read-only**, org enabling installation auth, short-lived installation token |
| Interactive engineer workflow | User login / user auth | User Copilot entitlements and normal session approval |
| Personal-account repo | No equivalent org installation billing path assumed | Check actual support; the public-preview org-installation path is **not** a general personal-account solution |

**Security nuance:** Copilot Requests permission says *Read-only*, but requests **consume paid AI credits**. Do not interpret the permission label as cost-free. GitHub's current organization admin docs state that personal and enterprise installations cannot use this particular org-Copilot authentication path.

## GitHub Actions example

For an **organization-owned** repo whose organization enables the required Copilot CLI policy:

```yaml
name: copilot-investigation
on: [workflow_dispatch]
permissions:
  contents: read
  copilot-requests: write
jobs:
  investigate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Execute pinned agent harness
        run: ./scripts/run-copilot-agent.sh
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Pin actions/scripts and apply a separate permissions policy if the worker also needs to submit changes. The token above is for **making Copilot requests**; `contents: read` deliberately does not grant repository writes. An application with side effects should use an isolated GitHub App or workflow step for reviewed PR publication.

## External worker example (current Copilot SDK docs)

1. Create a GitHub App with **organization Copilot Requests: Read-only**.
2. Install it directly on the **organization** to bill. Obtain its installation ID.
3. Mint a new installation access token via the GitHub App API with a permission request like:

```json
{
  "permissions": {
    "organization_copilot_requests": "read"
  }
}
```

4. Pass that token explicitly to the SDK, disabling user-credential fallback:

```ts
import { CopilotClient, RuntimeConnection } from "@github/copilot-sdk";

const token = process.env.INSTALLATION_TOKEN;
if (!token) throw new Error("installation token missing");

const client = new CopilotClient({
  connection: RuntimeConnection.forStdio(),
  gitHubToken: token,
  useLoggedInUser: false
});
```

Tokens expire after **one hour**. Mint a replacement before expiry; GitHub documents restarting the SDK client or runtime to apply it. Keep the token out of model-visible prompts, trace payloads and the mutable agent filesystem. The SDK's `useLoggedInUser: false` disables ambient token discovery; passing a token only as `COPILOT_GITHUB_TOKEN` in the hosting process will not work for this explicit child-client configuration. For an **existing runtime URI** use the runtime process's `COPILOT_GITHUB_TOKEN` instead, as documented.

**Docs are evolving:** during the October preview some older/cached GitHub documentation describes *repository-level* `copilot_requests: write` and a different token-injection path. The current organization-level docs specify `organization_copilot_requests: read` and explicit SDK token options. Pin an SDK version, check the current docs for it, and smoke-test before rollout rather than mixing the two versions.

## Operational contract

```text
scheduler -> task/run identity -> credential broker
  -> fresh org installation token -> Copilot SDK agent
  -> bounded tools / sandbox
  -> separate repo-write publisher with approved permissions
  -> exact-head checks + PR protection
  -> cost/usage + audit records keyed to app and run
```

Keep a per-run record of `app_client_id`, installation/org, model, token issuance window, task ID, cost/usage and actual downstream GitHub effects. In the AI usage CSV, `app_client_id` attributes organization-installed GitHub App usage; it is not necessarily populated for users or Actions. The organization owns the budget and model policies; **per-app spending limits are not available** in this preview. If the budget must truly stop requests, turn on the organization's **Stop usage when budget limit is reached** setting. Suspend/uninstall the app to revoke access.

## Failure modes and limits

- **Public preview**, product-policy and SDK compatibility can change.
- A `401` may mean an expired token; a `403` may mean an incorrect app permission, org policy or disabled installation auth. Do not retry indefinitely with an engineer's personal credentials.
- The organization's Copilot model policy can deny a requested model regardless of token validity.
- An app installation is not an approval for side effects or broad repository access. Scope the publisher separately.
- App-attributed billing is not a substitute for task-level cost and outcome accounting.
- The same organization-owned identity can create a **shared cost blast radius** if quotas and per-run budgets are omitted.

## Prototype experiment

Create a small organization-level test App and one fake read-only issue triage worker. Test normal SDK inference, token expiry/reissue, disabled app, denied model, exhausted org budget, missing credential, and lack of repository write access. Correlate the run with the organization usage CSV and verify which data fields exist. Compare the Actions `GITHUB_TOKEN` path with an external orchestrator: steps needed, identity governance, latency, cost and failure recovery.

## Sources

- GitHub changelog, **2026-10-27**: https://github.blog/changelog/2026-10-27-github-copilot-supports-server-to-server-authentication/
- GitHub current SDK setup: https://docs.github.com/en/copilot/how-tos/copilot-sdk/auth/server-to-server-tokens
- GitHub organization control and cost caveats: https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/manage-app-copilot-access

## Related

- [Identity-scoped MCP gateway](identity-scoped-mcp-gateway.md)
- [Bind agent approvals to the exact effect](enforcement-bound-agent-approvals.md)
- [Durable agent tool effects and fencing](durable-agent-tool-effects-and-fencing.md)
- [Agent fleet unit economics](agent-fleet-unit-economics.md)
