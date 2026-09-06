# Knowledgebase intake prompt

This is the copyable governing prompt for the engineering knowledgebase intake system used by this repository.

Replace `<OWNER>/<REPO>` with your repository, then customize the priority areas and source radar if desired. The prompt is intentionally detailed: it is the policy and quality bar for an autonomous or scheduled intake run, not merely a search query.

```text
Maintain an ACTIONABLE engineering knowledgebase in the connected GitHub repository <OWNER>/<REPO>. The purpose is to accumulate concrete techniques, APIs, annotations, libraries, configurations, commands, implementation recipes, architecture patterns, coding-agent workflows, prompts/harness techniques, tests, checks, tools, production case studies, and decision rules that can be directly reused while developing software.

DISCOVERY MUST BE OPEN-ENDED
Do not treat any named source, company, technology, pattern, topic list, or example in this prompt as exhaustive. They are seeds, not boundaries. Each run should actively discover relevant material beyond the known watchlist through web search, linked references, citations, conference material, engineering blogs, papers, release notes, documentation, repositories, and newly emerging sources. A valuable source from an unknown company, individual engineer, academic group, open-source project, conference talk, issue/PR, or niche technical blog should be treated on merit. Do not repeatedly search only the same companies or publications because they are named here. Discovery should be broad even when canonical output is deliberately small.

CORE ADMISSION TEST
Before adding canonical knowledge, ask: "Could we reasonably use this in a real development task? What exactly would we do differently?" If there is no concrete answer, do not add it. Actionable includes both code/platform primitives and reusable development-process/agent-system patterns. A concrete API/annotation/configuration/library is actionable, but so is a production-proven coding-agent architecture when it yields an implementable workflow, harness rule, verification gate, isolation strategy, permission model, reviewer topology, eval method, or decision rule. Interesting trends or generic architecture wisdom without an implementable consequence are not enough.

CODING-AGENT ENGINEERING IS A FIRST-CLASS PRIORITY
Actively search for real production or serious experimental coding-agent systems and extract reusable patterns. Interesting territory includes, but is explicitly NOT limited to: autonomous issue-to-PR systems; multi-agent code review; independent reviewers and cross-validation; planner/implementer/reviewer pipelines; worktree/sandbox isolation; deterministic verification gates; runtime/control-plane state checks before edits; stale/no-op detection; human approval placement; credential and permission boundaries; MCP/tool gateways; semantic code navigation; repository memory/context systems; parallel agents and conflict management; agent-generated test validation; CI-integrated agents; task decomposition; recovery/replanning; long-running agents; agent observability; eval harnesses; merge/defect/review-burden/productivity measurement; and mechanisms for safely increasing autonomy.

Production case studies are especially valuable when they expose whole-system designs, implementation details, measurements, failure modes, and lessons. Search broadly across large companies, startups, research groups, OSS maintainers, developer-tool vendors and individual engineers.

ENGINEERING-BLOG RADAR
Use the following as a broad recurring radar, not a whitelist and not a requirement to scan every site on every run. Rotate across it over time and follow links to new sources.

High-priority engineering organizations: Uber Engineering; DoorDash Engineering; LinkedIn Engineering; Netflix TechBlog; Cloudflare Engineering; Stripe Engineering; Airbnb Engineering; Shopify Engineering; Dropbox Tech; Meta Engineering; Google engineering/research/developer blogs; Microsoft Engineering; GitHub Engineering; Anthropic Engineering/Research; OpenAI Engineering/Research; JetBrains; AWS Builders' Library and AWS engineering; Datadog Engineering; Slack Engineering; Canva Engineering.

Additional strong engineering sources to sample by topic: Block/Square; Pinterest; Reddit; Discord; Spotify; Yelp; Lyft; Grab; Booking.com; Expedia; eBay; PayPal; Twilio; Salesforce; Atlassian; Adobe; HubSpot; GitLab; HashiCorp; Grafana Labs; Elastic; Confluent; Cockroach Labs; MongoDB; Snowflake; Databricks; Temporal; Sentry; Sourcegraph; Vercel; Fastly; Tailscale; Pulumi; Snyk; Wiz; Palantir; NVIDIA; Apple ML Research; ByteDance/TikTok; Alibaba; Bloomberg; Jane Street; CloudBees. Treat this list as examples only and discover peers beyond it.

Developer-agent/tooling radar: GitHub/Copilot; JetBrains; Cursor/Anysphere; Cognition/Devin; Sourcegraph; Replit; Augment; Factory; CodeRabbit; Qodo; Continue; Cline; Windsurf/Codeium and other emerging coding-agent, review-agent, IDE-agent and developer-infrastructure teams. Vendor material is useful for discovering concrete primitives and patterns, but do not treat vendor performance claims as neutral evidence without independent support.

OTHER FIRST-CLASS AREAS
Continue broad discovery across Java/JVM/Spring, software/distributed architecture, platform/cloud engineering, developer productivity, practical AI engineering, APIs/protocols, testing, security, data, observability, and meaningful changes in engineering practice. Do not let the coding-agent priority crowd out an unusually valuable actionable discovery elsewhere.

GENERAL SOURCE SEEDS
Useful recurring non-company or ecosystem sources include Baeldung Java Weekly and linked articles, InfoQ, Martin Fowler, Thoughtworks Technology Radar, Simon Willison, Latent Space/AI Engineer, Foojay, Inside Java, Spring, Quarkus, OpenJDK/JEPs, official documentation, release notes, GitHub repositories/issues/PRs, conference talks and research papers. This list is deliberately non-exclusive.

DISCOVERY METHOD
On each run, first inspect the repository's canonical knowledge, all processed-source logs, recent commits, and open intake PRs. Treat sources already recorded in main or an open intake PR as processed. Then perform both:
1. watchlist/radar discovery from a rotating subset of known high-quality sources; and
2. open-web discovery using varied topic/problem searches that can reveal sources we do not yet know.
Follow worthwhile references and primary sources. For newsletters, read the newsletter and important linked subpages rather than summarizing blurbs. When a previously unknown source repeatedly produces strong material, treat it as a new radar candidate; the radar should evolve rather than remain fixed.

CANONICAL KNOWLEDGE
Organize by future retrieval/use, not publication. Prefer focused pages for reusable techniques/capabilities/patterns. For a production coding-agent case study, it is acceptable to retain a concise case-study page when the whole system design is itself reusable; also extract recurring pieces into canonical pattern pages when useful. Avoid article-summary archives.

Canonical entries should include as applicable:
- What it is
- Use when
- Concrete implementation/workflow shape
- Code/config/command/prompt/pseudocode where useful
- Why it is useful
- Failure modes and caveats
- Version/compatibility or environmental assumptions
- Evidence/measurements and what they do NOT prove
- Source links
- Related entries
- A concrete experiment for patterns not yet validated by us

For agent-system patterns, make the implementation sufficiently concrete that we could prototype it: roles, context boundaries, tool/permission boundaries, state transitions, gates, failure handling, and metrics as applicable.

SOURCE LOG
Maintain a lightweight append-only processed-source log with date, source, title, URL, and disposition such as incorporated, evidence-for-existing-entry, skipped-not-actionable, skipped-duplicate, skipped-low-confidence, or experiment-candidate. The log is for deduplication, not the main product. Inspect both the legacy `sources/processed.md` and dated logs under `sources/processed/`.

INTAKE SYNTHESIS — THIS IS A PRIMARY ARTIFACT
Every run must create a useful human-readable intake note under `intakes/`. It must not read like an administrative run ledger. Begin the note with a `## Synthesis` section that explains, in compact editorial prose, what we learned from the run as a whole.

The synthesis should answer questions such as:
- What is the strongest idea or recurring pattern across the sources?
- How do the new findings connect to or change existing KB knowledge?
- If multiple independent systems point in the same direction, what shared architecture/practice emerges?
- What should we now do differently when designing or using software/coding agents?
- Where does evidence disagree, remain weak, or suggest a tradeoff?

Write this at the same quality level as a strong conversational technical summary: explain the "aha", not merely the file changes. Prefer 2-6 substantive paragraphs plus a compact architecture/workflow sketch when that materially clarifies the synthesis. Do not inflate weak runs; if there is no broader synthesis, say that and keep it short.

After `## Synthesis`, include the operational sections:
1. `## New things we can use now` — concise explanation of each canonical addition and why it matters.
2. `## Existing KB entries improved` — only if applicable.
3. `## Valuable agent-system designs discovered` — particularly useful whole-system designs and what can be reused from them.
4. `## Rejected / deferred` — interesting items that did not pass canonical admission and why.
5. `## Experiments worth trying` — concrete experiments in code or our agent-development workflow.
6. `## Newly discovered sources worth revisiting` — if any.
7. `## Source log` — link to the processed-source ledger for the run.

The synthesis is durable knowledge about the intake itself: someone reading it months later should understand why that run mattered without opening every canonical entry or source article. Avoid generic industry commentary and avoid merely repeating the PR body.

QUALITY AND EXPLORATION BAR
A run with 1-3 genuinely useful canonical additions is better than 10 weak ones, and zero is valid. But do not confuse a small output with a narrow search. Explore beyond familiar sources and vary both companies and search directions between runs. Prefer primary documentation and direct engineering writeups; use independent evidence/papers for effectiveness claims. Distinguish demonstrated behavior from inference. Company blog marketing copy without implementation detail should be rejected quickly.

PUBLISH AND MERGE
Make changes on a fresh branch from current main and open a ready-for-review PR titled `KB intake: YYYY-MM-DD`. The PR body should contain a very short version of the synthesis plus the canonical additions; the full editorial synthesis belongs in the intake note. Before merging, re-read the final diff; confirm only intended KB files changed, processed-source state is consistent, and every canonical addition passes the actionable test. Check mergeability and required CI/status checks. If clean and allowed, merge automatically into main, preferably squash merge. If failing/pending required checks, conflict, required human approval, unexpected changes, or material uncertainty prevents safe merge, do not bypass protections; leave the PR open and document the blocker. Always start subsequent runs from latest main and reconcile stale intake PRs without duplicating their sources.
```

## Customize it

At minimum, change:

- `<OWNER>/<REPO>`
- the first-class technology/domain areas you care about
- the engineering/source radar
- the desired merge policy if you do not want automatic merging

Keep the admission test, processed-source deduplication, synthesis requirement, evidence discipline, and final-diff verification unless you have a deliberate reason to change them. Those are the parts that stop the repository from degenerating into an article-summary graveyard.
