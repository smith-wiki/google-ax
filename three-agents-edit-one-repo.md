# Three agents can edit one repository through three isolated AX Tasks

**Concrete job:** improve the onboarding of this research repository. An explainer edits `ax-is-an-orchestrator.md`, a workflow reviewer edits `ax-does-not-model-agent-teams.md`, and an operations reviewer edits `remote-k3s.md`. Each gets a separate clone of the **same** Git repository from a reusable `Workspace` definition, but none shares a writable directory or Git push credentials with another ([Workspace setup](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md#L22-L32), [AX Task model](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md#L5-L9)).

```text
trusted coordinator (the "brain")
       │ submits 1 Workspace + 3 Tasks
       ▼
AX control plane → Agent Substrate → 3 isolated agent commands
       ▲                                  │
       └── fetches 3 Git patches, checks and merges them ──┘
```

The agent executable inside each Task makes its one file edit and writes a Git patch and completion marker to **its own** `/workspace`. AX/Substrate run and isolate the Tasks, not the multi-agent plan. The trusted coordinator waits for each marker, collects patches through privileged `ax ssh`, checks the base revisions and changed paths, then reviews and merges them into a branch. AX's `Ready` status does not mean an agent finished successfully; it does not transfer files or merge patches for the coordinator ([runner exit behavior](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L41-L47), [AX team boundary](ax-does-not-model-agent-teams.md)).

[The worked example](three-agents-edit-one-repo-guide.md) has the four-resource manifest, exact command-to-patch contract, and coordinator collection steps. It illustrates the role of each layer, **not a verified cluster run**; the existing [remote k3s prerequisites](remote-k3s.md) and an agent runtime still matter. With only three agents, the AX/Substrate stack is probably heavier than running three ordinary processes; its intended advantage is operating many isolated stateful sandboxes ([why AX exists](ax-solves-agent-operations.md)).
