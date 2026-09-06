# Run this knowledgebase system with ChatGPT

This repository is designed so another engineer can clone/fork it, connect GitHub to ChatGPT, and use the same intake workflow without rebuilding the process from scratch.

The key design choice is that the **governing prompt lives in Git** at [`prompts/knowledgebase-intake.md`](../prompts/knowledgebase-intake.md). ChatGPT can read that file at the beginning of every run, so improvements to the intake policy are versioned with the knowledgebase instead of being trapped inside one chat or scheduled task.

## 1. Clone or fork the repository

Fork this repository, or copy its structure into your own repository.

Then edit [`prompts/knowledgebase-intake.md`](../prompts/knowledgebase-intake.md):

- replace `<OWNER>/<REPO>` with your repository;
- change the first-class technology/domain areas;
- tune the source radar for your interests;
- change the merge policy if you do not want automatic merging.

Keep the actionable admission test, source deduplication, evidence rules, intake synthesis, and final-diff verification unless you intentionally want a different system.

## 2. Connect GitHub to ChatGPT

In ChatGPT, open the Plugin/App directory or Settings and connect GitHub. Authorize the GitHub account and grant ChatGPT access to the repository you want the intake system to maintain.

OpenAI's current GitHub connection documentation:

- https://help.openai.com/en/articles/11145903

Repository access is permission-scoped. If you want ChatGPT to create branches, commits, pull requests, or merges, the connected GitHub integration must have the corresponding permissions. Depending on your account/workspace policy, write actions may require explicit approval.

## 3. Test one manual run first

Start a normal ChatGPT conversation and give it this small bootstrap instruction:

```text
Use the connected GitHub repository <OWNER>/<REPO>.
Read prompts/knowledgebase-intake.md from the repository and treat it as the governing instructions for this run.
Execute one complete knowledgebase intake now.
```

The expected result is:

1. ChatGPT inspects existing canonical knowledge, source logs, recent commits, and open intake PRs.
2. It performs broad web discovery.
3. It adds only items that pass the actionable admission test.
4. It creates the dated source ledger and editorial intake synthesis.
5. It creates a fresh branch and ready-for-review PR.
6. It checks the final diff, CI/status checks, and mergeability.
7. If safe and permitted, it merges; otherwise it leaves the PR open with the blocker documented.

Do the first run manually because it exposes repository-permission or approval problems before you put the workflow on a schedule.

## 4. Make it recurring with ChatGPT Scheduled Tasks

ChatGPT supports recurring Scheduled Tasks and can use supported connected apps such as GitHub. Open the **Scheduled** page in ChatGPT, create a recurring task, and choose the cadence you want.

OpenAI's current Scheduled Tasks documentation:

- https://help.openai.com/en/articles/10291617

For this repository, a useful schedule is **every 4 days**. Use this as the scheduled-task instruction:

```text
Maintain the engineering knowledgebase in the connected GitHub repository <OWNER>/<REPO>.
At the beginning of every run, read prompts/knowledgebase-intake.md from the repository and treat the current contents of that file as the governing instructions.
Execute the complete intake workflow described there, including discovery, source deduplication, canonical updates, the intake synthesis, branch/PR creation, final-diff verification, CI/mergeability checks, and safe merge when allowed.
```

Keeping the scheduled-task instruction short is intentional. The durable policy stays in Git, where it can be reviewed and improved. The task only points to the current version.

### Why keep the prompt in Git?

A scheduled task created inside a ChatGPT Project cannot rely on Project-uploaded files being available when the task runs. A prompt stored in the connected GitHub repository avoids that problem and also gives you normal version history, review, blame, and rollback.

## 5. Review the first few scheduled runs

For the first runs, inspect:

- whether discovery is broad rather than repeatedly sampling the same sources;
- whether canonical additions really answer "what would we do differently?";
- whether rejected sources are still logged for deduplication;
- whether the synthesis explains the engineering insight instead of only listing file changes;
- whether GitHub writes are limited to intended KB files;
- whether automatic merge happens only after required checks and approvals are satisfied.

If the output drifts, change [`prompts/knowledgebase-intake.md`](../prompts/knowledgebase-intake.md), not the scheduled task. The next run will pick up the new policy.

## 6. Suggested repository shape

```text
README.md
CONTRIBUTING.md
prompts/
  knowledgebase-intake.md
intakes/
  YYYY-MM-DD-....md
sources/
  processed.md
  processed/
    YYYY-MM-DD.md
topics/
  ai/
  java/
  platform/
  spring/
  testing/
  ...
```

The exact taxonomy can differ. What matters is separating:

- **canonical reusable knowledge** from
- **run-level synthesis** from
- **the source deduplication ledger**.

## Manual vs scheduled usage

Use a manual ChatGPT run when you want to steer discovery interactively or experiment with the prompt. Use Scheduled Tasks when the workflow and permissions are stable enough to run repeatedly.

The resulting system is intentionally boring in the right places: Git stores policy and durable state; ChatGPT performs discovery and synthesis; deterministic repository/CI checks govern publication. That is much easier to reason about than hiding the whole thing inside one immortal mega-chat.
