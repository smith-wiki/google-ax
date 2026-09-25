# What is Google's AX, and how do you use it?

[AX](https://github.com/google/ax) is a declarative orchestration runtime for isolated agent workloads. Start with [what it does](what-is-ax.md); follow the questions below to see how this research develops.

## Questions and answers

1. **2026-09-25 — What is Google's AX, and how does it work?** AX orchestrates sandboxed tasks on Agent Substrate; the actual agent is the task's command, not AX itself. [AX is an orchestrator](ax-is-an-orchestrator.md).
2. **2026-09-25 — Can I try AX on a bare Mac without Kubernetes?** Yes for its CLI and runner; running a real AX Task requires a cluster. [Kubernetes is needed for real Tasks](kubernetes-required-for-tasks.md).
3. **2026-09-25 — Can my existing remote k3s cluster replace local kind?** Potentially: no local kind is needed, but k3s must meet Agent Substrate's API, worker, storage, and image requirements first. [Remote k3s card](remote-k3s.md).

The cluster-backed paths are source-grounded plans, not reports of a working deployment. AX [warns that its concepts and protocols may change before a stable release](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md).
