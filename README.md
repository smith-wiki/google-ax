# What is Google's AX, and how do you use it?

[AX](https://github.com/google/ax) is a declarative orchestration runtime for isolated agent workloads. Start with [what it does](what-is-ax.md); follow the questions below to see how this research develops.

## Questions and answers

1. **2026-09-25 — What is Google's AX, and how does it work?** AX orchestrates sandboxed tasks on Agent Substrate; the actual agent is the task's command, not AX itself. [AX is an orchestrator](ax-is-an-orchestrator.md).
2. **2026-09-25 — Can I try AX on a bare Mac without Kubernetes?** Yes for its CLI and runner; running a real AX Task requires a cluster. [Kubernetes is needed for real Tasks](kubernetes-required-for-tasks.md).
3. **2026-09-25 — Can my existing remote k3s cluster replace local kind?** Potentially: no local kind is needed, but k3s must meet Agent Substrate's API, worker, storage, and image requirements first. [Remote k3s card](remote-k3s.md).
4. **2026-09-25 — How would an agent team be described and solve a simple task in AX?** AX declares separate agent Tasks, not a team workflow; a coordinator compares their reports. [AX does not model agent teams](ax-does-not-model-agent-teams.md); [two-agent conversion example](two-agents-check-a-conversion.md).
5. **2026-09-25 — Then what problem does AX actually solve?** It operates many isolated, stateful agent sandboxes with repeatable setup and lifecycle control; it does not plan agent teamwork. [AX solves agent operations](ax-solves-agent-operations.md).
6. **2026-09-25 — What is Agent Substrate underneath AX?** A Kubernetes-backed runtime that maps logical sandboxed actors onto warm worker pods and manages their lifecycle and routing. [Agent Substrate explained](what-is-agent-substrate.md).
7. **2026-09-25 — Do I need a separate agent brain, and how does it talk to AX?** Your agent program supplies the brain, whether inside or outside AX; a trusted client calls AX's typed gRPC task API, while your application handles results. [Brain-to-AX interface](brain-to-ax-interface.md).
8. **2026-09-25 — What would three agents editing a repository actually do through AX/Substrate?** Three isolated Tasks edit separate files in separate clones; a trusted coordinator receives their patches and merges them. [Three-agent example](three-agents-edit-one-repo.md); [worked manifest](three-agents-edit-one-repo-guide.md).
9. **2026-09-25 — Are AX Tasks deterministic code or agent operations?** Either: each Task runs the command you supply, which may be an ordinary script, an LLM-powered agent, or both in sequence. [A Task runs a command](ax-task-runs-a-command.md).

The cluster-backed paths are source-grounded plans, not reports of a working deployment. AX [warns that its concepts and protocols may change before a stable release](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md).
