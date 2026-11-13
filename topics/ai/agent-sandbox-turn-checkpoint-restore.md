# Recover agent filesystem and process state at turn boundaries

## What it is

Checkpoint the **actual execution sandbox**, not just the model transcript. A coding agent can install dependencies, start services, mutate local files and hold live process state. Restoring only conversation messages does not reconstruct that OS state.

The open-source **Crab** experimental runtime provides a concrete implementation: coordinate at agent turns, inspect recovery-relevant effects via eBPF, select no/filesystem/process/full checkpoint, and overlap snapshots with LLM wait time. Its engine uses `runc`, CRIU and ZFS to create consistent savepoints.

## Use when

- an agent task spans many turns, package installations, long builds or local services;
- work is hosted on interruptible/preemptible workers or needs rollback/branching;
- restart-from-scratch and full-every-turn snapshots are costly;
- an experiment needs to separate conversation recovery from OS/workspace recovery.

**Not needed** for a single stateless task that can be cheaply rerun from a pinned Git checkout.

## Two-plane recovery design

~~~text
Durable workflow/control plane:
  run_id, turn_id, instruction transcript, external tool claims/results,
  authorization, fencing token, sandbox checkpoint ID

Sandbox data plane:
  filesystem snapshot F_n, process dump P_n

end of agent turn
  -> inspect net effects of tools/OS
  -> choose NO_CHANGE | FS | PROCESS | FULL
  -> create coherent checkpoint while awaiting LLM
  -> durably record {turn_id, F_n, P_n, transcript offset}
  -> only then permit next turn to consume model response

crash -> claim/lease check -> restore matching F/P
     -> reconstruct transcript at checkpoint boundary
     -> reconcile external effects before retry/publish
~~~

A successful checkpoint is a *recoverable local savepoint*, not a distributed transaction. Never interpret it as undoing a GitHub push, database write or payment. Keep the existing [tool-effects and fencing](durable-agent-tool-effects-and-fencing.md) ledger for those operations.

## Reproduce on a disposable Linux host

Crab v0 is a technical preview requiring **Ubuntu 24.04/26.04 x86-64**, root-owned single-user daemon and `runc` + CRIU + OpenZFS. Use an isolated VM with no production workloads or secrets.

~~~bash
git clone https://github.com/open-agent-infra/crab.git
cd crab
sudo ./scripts/install-ubuntu.sh
sudo ./scripts/smoke-rollback.sh
~~~

The installer creates a dedicated sparse-file ZFS pool named `crab`; inspect the script and host configuration before running it. The smoke test needs no model credential and verifies state corruption followed by rollback. For explicit savepoints:

~~~bash
sudo crab daemon start --config /etc/crab/config.yaml
SBX=$(sudo crab sandbox run --detach ubuntu:22.04)
sudo crab sandbox exec "$SBX" -- sh -lc 'echo before > /root/state.txt'
CKPT=$(sudo crab checkpoint create "$SBX")
sudo crab sandbox exec "$SBX" -- sh -lc 'echo after > /root/state.txt'
sudo crab restore "$SBX" "$CKPT"
sudo crab sandbox exec "$SBX" -- cat /root/state.txt  # before
sudo crab sandbox rm "$SBX"
sudo crab daemon stop
~~~

To use it from an agent, adopt the documented `Agent` integration/profile or SDK. Keep the agent's checkpoint token and transcript offset together; do not race a new model turn ahead of an incomplete snapshot. The experimental design delays delivery of an LLM response until the matching savepoint is durable.

## Boundary and failure modes

- **Host bind mounts are excluded.** In Crab v0, `--work-dir` maps a host directory to `/work`; restoring the sandbox does *not* restore that host checkout. Copy/clone into the ZFS-backed sandbox root if rollback of the repository is required.
- External network/API changes are untouched. Recover unknown external results by querying the authority; never blindly repeat.
- A root-owned, single-user daemon is not multi-tenant isolation. Do not deploy it as a shared untrusted sandbox service without separate containment design.
- Daemon restart rehydration and rootless/multi-user support are not implemented in v0. Test which failures are actually recoverable.
- A net-zero temporary-file turn might still have effects outside observed scope; monitoring coverage must be measured, not assumed complete.
- Checkpointing processes can interact with sockets, network sessions and nondeterministic external services. Prefer reproducible local fixtures in the pilot.

## Evidence and experiment

The Crab April 30, 2026 paper reports 100% recovery correctness versus 8% for chat-only restoration on its benchmark, up to 87% checkpoint traffic reduction and within 1.9% of fault-free runtime. These are the authors' controlled results on selected tasks/hosts, **not** production multi-tenant reliability guarantees.

Prototype with 10 real long-running coding-agent tasks: inject worker kills after (a) a read-only turn, (b) writing a source file, (c) starting a service and (d) a confirmed or ambiguous external API call. Compare chat-only recovery, full snapshots and adaptive checkpoints. Require recovered build/test state, no duplicate external effects, same working-tree SHA, recovery latency, storage I/O, and zero cross-tenant leaks. Treat daemon death as an explicit unsupported recovery case until validated.

## Sources

- https://arxiv.org/abs/2604.28138
- https://github.com/open-agent-infra/crab

## Related

- [Durable tool-effect claims and fencing](durable-agent-tool-effects-and-fencing.md)
- [Environment-first containment](environment-first-agent-containment.md)
- [Deterministic outer loop](deterministic-outer-loop-agent-platform.md)
