# Trying AX on an Apple Silicon Mac

> Source snapshot: [`google/ax` at `e09ed1b`](https://github.com/google/ax/tree/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7) and [`agent-substrate/substrate` at `22ba860`](https://github.com/agent-substrate/substrate/tree/22ba860e0f32e20713165b96abd81f6f96da620c). AX is pre-stable, and its module actually pins the older Substrate commit `672533541dbf` ([AX `go.mod`](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/go.mod#L1-L9)). Commands below therefore pin both projects; this page does not assume that AX works against current Substrate `main`.

## Short answer

**Kubernetes is not required to read the manifests, build the `ax` CLI, or run `ax-task-runner` locally. Kubernetes is required for a real `Task`.** AX's server/controller turn a Task into an Agent Substrate actor, and Substrate itself uses Kubernetes for infrastructure and workers ([AX prerequisites](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md#L59-L91), [Substrate architecture](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/README.md#L5-L14)).

| Goal | Kubernetes? | Sensible first step |
|---|---:|---|
| Learn the CLI and manifest shape | No | Build `ax`; run `ax help` and `ax version`. Commands that read/write resources still need an AX server. |
| Exercise runner metadata, readiness, environment, and child-command behavior | No | Run `ax-task-runner` directly with local YAML files. |
| Execute `ax apply`, create a sandbox, or try suspend/resume/`ax ssh` | Yes | Use the pinned Substrate revision in a local kind cluster, after the adaptations below. |

## Track 1: the smallest useful local experiment

AX requires Go 1.27.1 at this revision ([module declaration](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/go.mod#L1-L9)). Install a current Go toolchain, then:

```bash
git clone https://github.com/google/ax.git
cd ax
git checkout e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7

go run ./cmd/ax version
go run ./cmd/ax help
```

These two commands do not contact Kubernetes. Normal resource commands either use `--server`/`AX_SERVER` or derive a Kubernetes context and start a `kubectl port-forward` to `svc/ax-server` ([CLI dispatch](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax/main.go#L95-L127), [tunnel resolution](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/tunnel/tunnel.go#L177-L205)).

A better cluster-free probe runs the real runner. It accepts Task and Workspace files explicitly ([entrypoint flags](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax-task-runner/main.go#L15-L22), [flag parsing](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax-task-runner/main.go#L48-L78)):

```bash
mkdir -p /tmp/ax-local-workspace
cat >/tmp/ax-local-task.yaml <<'YAML'
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: local-probe
spec:
  command: ["/bin/sh", "-c", "printf 'runner reached the command\n' > result.txt"]
  workspaces:
    - name: local
      path: /tmp/ax-local-workspace
YAML

cat >/tmp/ax-local-workspace.yaml <<'YAML'
apiVersion: ax.io/v1alpha1
kind: Workspace
metadata:
  name: local
spec: {}
YAML

go run ./cmd/ax-task-runner \
  --task-file /tmp/ax-local-task.yaml \
  --workspace-file /tmp/ax-local-workspace.yaml \
  --port 8081
```

Leave that process running; the runner deliberately stays alive after its child exits. In another terminal:

```bash
curl -i http://127.0.0.1:8081/readyz
curl -s http://127.0.0.1:8081/metadata/v1alpha1/ax/task
cat /tmp/ax-local-workspace/result.txt
```

The checkpoints are HTTP `200`, the submitted Task YAML, and `runner reached the command`. The runner may warn that it cannot create container-oriented state directory `/ax` on macOS; that does not invalidate this one-shot probe, but it means the maiden-run marker is not persisted. Stop it with Control-C and remove the two YAML files and workspace. This exercises the runner contract—workspace setup, readiness, metadata, and process supervision—not the AX API server, Substrate isolation, snapshots, or the `ax` resource lifecycle ([runner behavior](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/runner/runner.go#L106-L200), [upstream local test](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L153-L165)).

## Track 2: a cautious kind experiment

This path is **source-supported but not verified end to end here**. Treat each checkpoint as a gate.

### 1. Install and check host tools

Install [Homebrew](https://brew.sh/) first if the Mac does not have it. Use a Docker provider that runs Linux containers, such as [Docker Desktop for Apple silicon](https://docs.docker.com/desktop/setup/install/mac-install/). With Homebrew, the basic set is:

```bash
brew install go git kubectl ko
brew install --cask docker
open -a Docker

docker info
docker buildx version
go version
kubectl version --client
ko version
```

Go must be at least 1.27.1. A separate `kind` installation is unnecessary: Substrate runs its pinned kind tool through Go. Its documented kind prerequisites are Go, `kubectl`, and a working Docker daemon ([Substrate quickstart](https://github.com/agent-substrate/substrate/blob/672533541dbf/README.md#L82-L108), [`kind.sh`](https://github.com/agent-substrate/substrate/blob/672533541dbf/hack/kind.sh#L17-L20)). Network access is also needed for Go modules, base images, Python packages, and public gVisor assets.

### 2. Pin the compatible Substrate source and create the cluster

```bash
cd ..  # from the Track 1 ax checkout; keep the source trees side by side
git clone https://github.com/agent-substrate/substrate.git
cd substrate
git checkout 672533541dbf

test "$(git rev-parse --short=12 HEAD)" = 672533541dbf
./hack/create-kind-cluster.sh

kubectl --context kind-kind get nodes
kubectl --context kind-kind get node \
  -o jsonpath='{.items[0].status.nodeInfo.architecture}{"\n"}'
```

On an M-series Mac, expect `arm64`. The script creates a local registry on `localhost:5001`, enables the certificate beta APIs used by Substrate and AX, and configures kind's containerd to reach that registry ([cluster configuration](https://github.com/agent-substrate/substrate/blob/672533541dbf/hack/create-kind-cluster.sh#L53-L126), [registry wiring](https://github.com/agent-substrate/substrate/blob/672533541dbf/hack/create-kind-cluster.sh#L194-L222)). Lack of `/dev/kvm` is explicitly non-fatal for gVisor. Do not pursue the microVM path on an M2: Substrate's Apple Silicon guide requires M3 or later for nested virtualization ([gVisor fallback](https://github.com/agent-substrate/substrate/blob/672533541dbf/hack/create-kind-cluster.sh#L72-L85), [M2 limitation](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/docs/dev/microvm-local.md#L89-L100)).

### 3. Install Substrate and actual worker capacity

```bash
./hack/install-ate-kind.sh --deploy-ate-system
./hack/install-ate-kind.sh --deploy-demo-counter

kubectl --context kind-kind -n ate-system get pods
kubectl --context kind-kind -n ate-system get svc api
kubectl --context kind-kind get sandboxconfig gvisor-default
kubectl --context kind-kind get workerpools -A
kubectl --context kind-kind -n ate-system get job rustfs-bucket-init
```

The kind wrapper deliberately selects `linux/$(go env GOARCH)`, `localhost:5001`, context `kind-kind`, and bucket `ate-snapshots` ([wrapper](https://github.com/agent-substrate/substrate/blob/672533541dbf/hack/install-ate-kind.sh#L22-L43)). Its gVisor config includes both amd64 and arm64 assets ([manifest](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/manifests/ate-install/sandboxconfig-gvisor.yaml#L15-L39)). The core install does not itself provide AX with a worker pool; the counter demo supplies a small generic gVisor pool ([pool manifest](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/demos/counter/counter.yaml.tmpl#L15-L60)). All listed workloads should be Ready before proceeding.

### 4. Close the two AX/Kind integration gaps

First, build an arm64 runner. Do **not** use AX's `make build-task-runner` unchanged: it hard-codes `linux/amd64` ([Makefile](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/Makefile#L45-L58)). From the pinned AX checkout:

```bash
cd ../ax
mkdir -p bin/linux_amd64
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build \
  -trimpath -ldflags='-s -w' \
  -o bin/linux_amd64/ax-task-runner ./cmd/ax-task-runner

docker build --platform linux/arm64 \
  -t localhost:5001/ax-task-runner:mac-arm64 \
  -f Dockerfile.task-runner .
docker push localhost:5001/ax-task-runner:mac-arm64
```

Capture the digest-pinned reference and verify the local image architecture:

{% raw %}
```bash
export TASK_IMAGE="$(
  docker image inspect --format '{{index .RepoDigests 0}}' \
    localhost:5001/ax-task-runner:mac-arm64
)"
test -n "${TASK_IMAGE}"
test "$(
  docker image inspect --format '{{.Architecture}}' \
    localhost:5001/ax-task-runner:mac-arm64
)" = arm64
printf '%s\n' "${TASK_IMAGE}"
```
{% endraw %}

A digest is mandatory: the AX-pinned Substrate revision requires a digest-pinned image reference ([validator](https://github.com/agent-substrate/substrate/blob/672533541dbf/cmd/ateapi/internal/controlapi/actor_template.go)). This also avoids AX's unpinned default image ([constant](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/pkg/apis/v1alpha1/types.go#L30-L40)). Stop here if the image does not build, push, or report arm64; the checked-in remote digest is not proven arm64-compatible.

Second, deploy AX and replace its GCS snapshot default with kind's S3-compatible RustFS bucket:

```bash
kubectl config use-context kind-kind
kubectl apply -f deploy/redis.yaml
KO_DOCKER_REPO=localhost:5001 KO_DEFAULTPLATFORMS=linux/arm64 \
  ko apply -f deploy/ax-controller.yaml
kubectl -n ax-system set env deployment/ax-controller \
  AX_SNAPSHOTS_BUCKET=s3://ate-snapshots/ax/
KO_DOCKER_REPO=localhost:5001 KO_DEFAULTPLATFORMS=linux/arm64 \
  ko apply -f deploy/ax-server.yaml

kubectl -n ax-system rollout status deployment/ax-redis
kubectl -n ax-system rollout status deployment/ax-controller
kubectl -n ax-system rollout status deployment/ax-server
kubectl -n ax-system get pods
```

That override is essential: AX ships a developer-specific `gs://dberkov-gke-dev3/ate-env/` value ([controller manifest](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/deploy/ax-controller.yaml#L67-L84)), whereas kind configures ate-api and atelet for S3/RustFS and creates `ate-snapshots` ([storage overlay](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/manifests/ate-install/kind/kustomization.yaml#L46-L76), [bucket job](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/manifests/ate-install/kind/rustfs.yaml#L93-L133)). `kubectl set env` is a trial-time patch; reapplying the original controller manifest can restore the wrong value.

### 5. Probe one real Task

```bash
go build -o bin/ax ./cmd/ax
./bin/ax --context kind-kind ctx

cat >/tmp/ax-kind-probe.yaml <<YAML
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: kind-probe
spec:
  image: "${TASK_IMAGE}"
  command: ["sh", "-lc", "printf 'task ran\\n' > /workspace/result.txt"]
  debug: true
YAML

./bin/ax --context kind-kind apply -f /tmp/ax-kind-probe.yaml
./bin/ax --context kind-kind watch task kind-probe
./bin/ax --context kind-kind describe task kind-probe
./bin/ax --context kind-kind ssh kind-probe -- cat /workspace/result.txt
```

Success means the last command prints `task ran`; merely seeing Running/Ready is not proof that the child command succeeded, because the runner remains alive after that child exits ([runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L41-L47)). This is a command-execution probe, not an AI-agent demonstration. A real agent still needs an arm64 image containing both `/usr/local/bin/ax-task-runner` and the agent executable, plus its own API credentials and cost controls.

### 6. Teardown

```bash
./bin/ax --context kind-kind delete task kind-probe
./bin/ax tunnel stop
rm -f /tmp/ax-kind-probe.yaml

cd ../substrate
./hack/delete-kind-cluster.sh
```

The teardown deletes the kind cluster and removes only the registry container labeled as created by Substrate ([script](https://github.com/agent-substrate/substrate/blob/672533541dbf/hack/delete-kind-cluster.sh#L19-L48)). It destroys the in-cluster RustFS/PostgreSQL data too.

## Boundaries and cautions

- **Version compatibility is the first gate.** A local source diff from AX's pinned `672533` dependency to Substrate `22ba860` changes the ActorTemplate field/type from `SnapshotsConfig snapshots_config` to `SnapshotConfig snapshot_config`, replaces container `readyz` with `wakeup_probe`, and changes lifecycle/status messages ([older schema](https://github.com/agent-substrate/substrate/blob/672533541dbf/pkg/proto/ateapipb/ateapi.proto), [current schema](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/pkg/proto/ateapipb/ateapi.proto#L751-L983)). AX's roadmap still lists migration to the new Actor API as future work ([roadmap](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/roadmap.md#L15-L23)). Do not substitute current Substrate `main` and assume compatibility.
- **Local kind security is developmental.** The RustFS overlay contains static `rustfsadmin` credentials and even marks secret management as a TODO ([overlay](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/manifests/ate-install/kind/atelet/kustomization.yaml#L45-L58)). `debug: true` enables process and filesystem access used by `ax ssh`; keep this cluster local and disposable.
- **Architecture remains a runtime checkpoint.** Substrate's own builds and gVisor assets have arm64 paths, but this source audit did not prove every pinned third-party image index or AX's Python/Antigravity package on arm64.
- **The kind command probe needs no cloud account or model API key.** Docker Desktop has [licensing conditions for some commercial use](https://docs.docker.com/desktop/setup/install/mac-install/); a later model-backed agent or workspace bootstrap would require provider credentials and could incur API charges. This probe deliberately does neither.

## Verification boundary

The repositories and scripts were inspected locally at the revisions named above. On the inspection machine, Go, `kubectl`, and `ko` were unavailable, so neither the local runner probe nor the kind/AX sequence was executed. The kind section is therefore an audited, conservative experiment plan with explicit stop points—not a claim of a successful Apple Silicon end-to-end deployment.
