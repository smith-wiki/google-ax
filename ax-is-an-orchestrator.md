# AX orchestrates tasks; it does not supply your agent

AX's server stores declared resources in Redis; its controller provisions [Agent Substrate](https://github.com/agent-substrate/substrate) actors, and the in-sandbox runner starts the task's `spec.command` as a child process. If `spec.command` is absent, no user agent starts—the runner stays up to serve metadata. AX is execution infrastructure, not an agent SDK or a ready-made agent. [Architecture](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/DESIGN.md#L3-L44) · [Runner behavior](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/runner/runner.go#L167-L195).

[Why AX exists](ax-solves-agent-operations.md) explains the operational problem this infrastructure addresses. [The architecture guide](what-is-ax.md) expands the three resources and lifecycle. [Kubernetes is required for a real Task](kubernetes-required-for-tasks.md) explains where this infrastructure runs.
