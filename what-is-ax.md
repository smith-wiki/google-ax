# AX orchestrates sandboxed tasks; your image and command provide the agent

> Source snapshot: [`google/ax` at `e09ed1b` (2026-09-24)](https://github.com/google/ax/tree/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7). AX explicitly warns that its concepts, protocols, and specifications may change before a stable release, and the public API is still `ax.io/v1alpha1` ([README](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md#L6-L13), [schema](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/types.go#L30-L40)).

AX is a Kubernetes-adjacent control plane for declaring, starting, inspecting, suspending, and resuming isolated workloads on [Agent Substrate](https://github.com/agent-substrate/substrate). It is **not** an agent SDK or a complete agent implementation. The project describes a path from `ax apply` through a gRPC API server and Redis queue to controllers, which create and activate Substrate actors; `ax-task-runner` then runs inside each actor ([architecture](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/DESIGN.md#L3-L44)). The README's “billions of tasks” language is the project's stated design target, not a result verified here.

## The three resources

| Resource | Intended role | What the inspected source currently confirms |
|---|---|---|
| **`Task`** | One isolated execution unit: image, command, environment, resource requests/limits, workspace bindings, and lifecycle. | The controller maps a task to a same-named Substrate actor and passes its `Task` and bound `Workspace` specifications to the container ([concept](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md#L5-L20), [reconciler](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/controller/reconciler.go#L94-L169)). The schema contains CPU/memory fields, but the current ActorTemplate builder does not copy them into its container definition; do not assume those limits are enforced at this revision ([schema](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/ax.proto#L70-L111), [builder](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/substrate/client.go#L211-L275)). |
| **`Workspace`** | Reusable declaration of Git inputs, MCP configuration, skills, and an optional natural-language setup goal. Each task receives its own materialized copy; the first binding is the command's working directory. | The default runner currently clones Git repositories, creates the configured skills directory, and optionally invokes bundled Antigravity for a binding `goal` ([concept](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md#L22-L32), [setup source](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/workspace/setup.go#L76-L127)). The setup path does **not** currently resolve skill registries or materialize MCP entries; it only creates the skills path ([source](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/workspace/setup.go#L261-L269)). Treat the broader Workspace description as direction, not all implemented behavior. |
| **`Model`** | Named provider/model/parameters/secret configuration for platform-owned model calls. | It is not the model automatically used by the user's agent. `TaskSpec` has no model reference, while the current task reconciler injects only the fixed `gemini-api-secret` / `GEMINI_API_KEY` credential for goal bootstrap ([Task schema](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/ax.proto#L70-L90), [secret lookup](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/controller/reconciler.go#L352-L368)). A model-backed workspace planner exists, but the repository's production paths do not call it at this snapshot; the roadmap still lists environment curation and harness customization as future work ([planner](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/workspace/planner.go#L37-L111), [roadmap](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/roadmap.md#L26-L40)). |

An **atespace** scopes all three resources; its default name is `default` ([concepts](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md#L1-L7)). It becomes a Substrate atespace, not a Kubernetes CRD namespace stored in etcd: AX stores resources in Redis and uses Redis Streams to feed horizontally scalable controllers ([design](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/DESIGN.md#L3-L34)).

## Runner versus agent

This distinction prevents the most likely false start:

1. The control plane always starts `/usr/local/bin/ax-task-runner` as the container command. A custom image must still provide that executable ([runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L9-L26)).
2. The runner is PID 1. It prepares workspaces, serves health/readiness and metadata, and starts `spec.command` as a supervised child. **That child is the user's agent or other workload** ([runner source](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/runner/runner.go#L106-L187)).
3. With no `spec.command`, the runner starts no user workload; it only keeps the metadata/guest service alive. Both checked-in examples omit `command`, so they demonstrate sandbox lifecycle and `ax ssh`, not an autonomous agent run ([source behavior](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/runner/runner.go#L167-L172), [example](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/examples/simple.yaml#L15-L33)).
4. A Workspace binding's `goal` is only a first-boot **environment setup** request to Antigravity. It requires `GEMINI_API_KEY`; missing credentials, timeout, or bootstrap failure is logged and the task command may still start ([setup source](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/workspace/setup.go#L272-L314)). It is not the task agent's ongoing goal.
5. When the child exits, the runner stays up and only logs its exit status; the controller does not currently receive that status. Consequently, `Running`/`Ready` does not mean “the agent succeeded” ([runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L41-L47)).
6. AX's current suspend/resume contract preserves `/workspace`, then starts a fresh container and process tree; it is not continuation of the child process in memory ([runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L21-L26), [snapshot scope](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/substrate/client.go#L254-L269)).

`spec.debug: true` enables process execution and filesystem guest services used by `ax ssh`; it is off by default because this is privileged inspection access ([sandbox guide](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/sandbox.md#L28-L33)).

Token/timeout budgets, approval policies, automatic idle suspension, stateful branching, task identity, governance, and trajectory/telemetry collection are roadmap items, not capabilities to rely on in this revision ([roadmap](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/roadmap.md#L3-L40)).

## Practical path to a first task

### Prerequisites

The documented path requires a Kubernetes cluster with Agent Substrate already installed, `kubectl`, `ko`, Go, and a registry the cluster can pull from ([quick start](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md#L59-L91)). Building this revision specifically requires Go **1.27.1** ([`go.mod`](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/go.mod#L1-L9)); Docker or Podman is additionally needed to build the task-runner image ([development guide](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/development.md#L3-L17)).

Before deployment, inspect and replace the checked-in `AX_SNAPSHOTS_BUCKET`: the controller manifest currently contains a project-specific `gs://dberkov-gke-dev3/ate-env/` value, and the source has the same fallback ([manifest](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/deploy/ax-controller.yaml#L67-L84), [source](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/substrate/client.go#L206-L227)). Also note that the repository's runner build target is hard-coded to `linux/amd64` ([Makefile](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/Makefile#L45-L58)).

The documented control-plane sequence is:

```bash
kubectl get svc api -n ate-system
go install github.com/google/ax/cmd/ax@latest
make deploy AX_IMAGE_REPO=<registry-your-cluster-can-pull>
```

`make deploy` applies Redis and builds/deploys `ax-controller` and `ax-server` with `ko` into `ax-system` ([Makefile](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/Makefile#L63-L85)). Normal CLI calls use the current kube context and automatically maintain a port-forward to `svc/ax-server`; `--server` or `AX_SERVER` bypasses that ([CLI documentation](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md#L122-L190)).

### Small execution probe

Unlike the upstream smoke examples, this manifest includes a real command. It proves child-command execution, not an AI agent:

```yaml
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: command-probe
  atespace: default
spec:
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:3a0dea6ad8b55278685db58aca6e37dc4ba04056831d45bef3aaeafdca43cac6"
  command: ["sh", "-lc", "echo runner-launched-me > /workspace/result.txt"]
  debug: true
```

```bash
ax apply -f command-probe.yaml
ax watch task command-probe
ax describe task command-probe
ax ssh command-probe -- cat /workspace/result.txt
ax suspend task command-probe
ax resume task command-probe
ax delete task command-probe
```

For a real agent, extend the default runner image with the agent executable and put its invocation in `spec.command`, or embed/replace the runner while honoring its HTTP and lifecycle contract; upstream documents all three options and a custom-runner manifest ([runner customization](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L58-L165)). For Workspace/Model syntax, start with the upstream [manifest guide](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/manifests.md) and current [complete example](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/examples/task.yaml), but cross-check the schema: that guide says “four kinds” and links a missing `examples/multi-workspace.yaml`, while the current source defines only `Task`, `Workspace`, and `Model` ([kinds](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/types.go#L30-L36)).

## Verification boundary

This research inspected the cloned source at the revision above. On this machine, `go`, `kubectl`, and `ko` were unavailable, and no Agent Substrate cluster was exercised; the control plane, manifest, suspend/resume path, and task image were **not run end to end here**. The commands above are a source-grounded procedure to try on a prepared cluster, not a claim of a successful local deployment.
