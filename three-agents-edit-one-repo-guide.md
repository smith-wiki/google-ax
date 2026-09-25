# Worked example: three agents edit one Git repository through AX

This is an **unexecuted, source-grounded design** for the public [google-ax research repository](https://github.com/smith-wiki/google-ax). The concrete request is: make its onboarding clearer. Three agents work from the same `main` revision but in **different** sandboxes:

| AX Task / agent role | Owns one existing file | Requested change |
|---|---|---|
| `wiki-overview` / explainer | `ax-is-an-orchestrator.md` | Add a concrete paragraph showing where an agent command runs. |
| `wiki-team` / workflow reviewer | `ax-does-not-model-agent-teams.md` | Add an example of a result handoff that AX does not automate. |
| `wiki-k3s` / operations reviewer | `remote-k3s.md` | Add a plain-language actor-versus-worker explanation for k3s readers. |

Here **four programs** are involved if the coordinator is automated: a trusted coordinator/“brain” chooses roles and merges results, and three separate agent commands edit the repository. AX provides the three `Task` sandboxes; Agent Substrate places their actors on its warm worker pool. Neither layer merges their edits. The tasks deliberately need no access to one another and no Git write credentials; the coordinator collects patches and decides what to publish ([AX Task concept](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/concepts.md#L5-L9), [Substrate actors and workers](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/docs/glossary.md#L43-L50), [brain interface](brain-to-ax-interface.md)).

## What the coordinator submits

This single multi-document manifest shows the complete AX side. All three `Task`s bind the **same Workspace resource**, but the runner clones its Git definition *separately in each task* at `/workspace/wiki` (`dir: wiki` is relative to the binding's `/workspace` path). `AGENT_GOAL` is the work instruction; a Workspace binding's `goal` would instead ask an agent to *prepare the environment*, so it is intentionally absent here ([Workspace setup](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/workspace/setup.go#L130-L193), [runner semantics](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L9-L43)).

```yaml
apiVersion: ax.io/v1alpha1
kind: Workspace
metadata:
  name: wiki-repo
  atespace: default
spec:
  git:
    - name: wiki
      repo: https://github.com/smith-wiki/google-ax.git
      branch: main
      dir: wiki
---
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: wiki-overview
  atespace: default
spec:
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:3a0dea6ad8b55278685db58aca6e37dc4ba04056831d45bef3aaeafdca43cac6"
  workspaces: [{name: wiki-repo, path: /workspace}]
  debug: true
  env:
    - name: TARGET_FILE
      value: ax-is-an-orchestrator.md
    - name: AGENT_GOAL
      value: "Edit only ax-is-an-orchestrator.md: add one concrete explanation of where an AX agent command runs. Cite the upstream AX runner. Do not commit or push."
  command:
    - /bin/sh
    - -ec
    - |
      set -u
      python3 /usr/local/bin/antigravity_bootstrap.py --workspace /workspace/wiki --goal "$AGENT_GOAL"
      git -C /workspace/wiki reset -q HEAD
      git -C /workspace/wiki add -- "$TARGET_FILE"
      git -C /workspace/wiki diff --cached --binary HEAD > /workspace/change.patch
      test -s /workspace/change.patch
      git -C /workspace/wiki rev-parse HEAD > /workspace/base.sha
      printf 'ok\n' > /workspace/done
---
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: wiki-team
  atespace: default
spec:
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:3a0dea6ad8b55278685db58aca6e37dc4ba04056831d45bef3aaeafdca43cac6"
  workspaces: [{name: wiki-repo, path: /workspace}]
  debug: true
  env:
    - name: TARGET_FILE
      value: ax-does-not-model-agent-teams.md
    - name: AGENT_GOAL
      value: "Edit only ax-does-not-model-agent-teams.md: add one example of transferring a result between two isolated Tasks using a coordinator. Cite AX sources. Do not commit or push."
  command:
    - /bin/sh
    - -ec
    - |
      set -u
      python3 /usr/local/bin/antigravity_bootstrap.py --workspace /workspace/wiki --goal "$AGENT_GOAL"
      git -C /workspace/wiki reset -q HEAD
      git -C /workspace/wiki add -- "$TARGET_FILE"
      git -C /workspace/wiki diff --cached --binary HEAD > /workspace/change.patch
      test -s /workspace/change.patch
      git -C /workspace/wiki rev-parse HEAD > /workspace/base.sha
      printf 'ok\n' > /workspace/done
---
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: wiki-k3s
  atespace: default
spec:
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:3a0dea6ad8b55278685db58aca6e37dc4ba04056831d45bef3aaeafdca43cac6"
  workspaces: [{name: wiki-repo, path: /workspace}]
  debug: true
  env:
    - name: TARGET_FILE
      value: remote-k3s.md
    - name: AGENT_GOAL
      value: "Edit only remote-k3s.md: add one plain-language example contrasting a logical actor with its warm worker Pod. Cite Agent Substrate sources. Do not commit or push."
  command:
    - /bin/sh
    - -ec
    - |
      set -u
      python3 /usr/local/bin/antigravity_bootstrap.py --workspace /workspace/wiki --goal "$AGENT_GOAL"
      git -C /workspace/wiki reset -q HEAD
      git -C /workspace/wiki add -- "$TARGET_FILE"
      git -C /workspace/wiki diff --cached --binary HEAD > /workspace/change.patch
      test -s /workspace/change.patch
      git -C /workspace/wiki rev-parse HEAD > /workspace/base.sha
      printf 'ok\n' > /workspace/done
```

The image digest comes from AX's [checked-in Task example](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/examples/task.yaml#L15-L38). Its [Dockerfile](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/Dockerfile.task-runner#L15-L31) includes Python, Git, the Antigravity SDK, and `/usr/local/bin/antigravity_bootstrap.py`. The helper's real purpose is **environment setup**; invoking it as `spec.command` to edit docs is an expedient teaching example, *not* the recommended production agent harness. A production image should provide its own agent command while retaining the AX runner contract ([helper prompt and policies](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax-task-runner/antigravity_bootstrap.py#L74-L101), [custom runner images](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L58-L75)). The helper requires `GEMINI_API_KEY`, which the AX controller can inject from its configured Secret; **no Git write credential enters the agents** ([secret lookup](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/controller/reconciler.go#L144-L155)). This is still a credential-bearing sandbox: restrict its network access and use only a repository you trust.

## What happens after `ax apply`

1. A **trusted coordinator**, not AX, selects these three independent roles and submits the manifest with `ax apply -f three-agent-wiki.yaml` (or issues typed `UpdateTask` gRPC calls). The AX server stores resources; its controller creates Substrate actor templates/actors. Substrate assigns the three sandboxes to available worker capacity—**not necessarily three Kubernetes Pods** ([AX architecture](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/DESIGN.md#L3-L44), [Substrate worker model](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/docs/glossary.md#L43-L50)).
2. Each runner prepares its own clone. Each agent edits its **one assigned file**. Its shell command resets any staged files, stages only that file, writes a binary-safe `git diff` to `/workspace/change.patch`, records its base commit in `/workspace/base.sha`, then writes `/workspace/done`. Those paths look the same in every Task but live on different task volumes; each Task's `spec.command` runs as the runner's child ([runner lifecycle](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/runner/runner.go#L119-L195)). If the agent fails or makes no tracked change, the shell stops before `done`. A marker does **not** prove the edit is correct.
3. The coordinator waits for **all three** `done` markers, with a timeout, then retrieves each `base.sha` and `change.patch` with `ax ssh <task> -- cat /workspace/<file>`. `debug: true` is needed for this privileged demo retrieval; the CLI writes guest command output to the client's standard output, so the coordinator can save each patch separately ([CLI guest output](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax/main.go#L1004-L1115)). `ax watch` / `Ready` is **not** the completion signal: the runner stays alive after the child exits, and AX does not receive its exit code ([runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L41-L47)).
4. On its **trusted** Git checkout, the coordinator rejects patches whose base commit differs, checks that each patch touches only its assigned path, applies the three patches, reviews the combined diff, checks links/content, and *only then* pushes a branch or opens a PR. Do not give sandboxed agents the coordinator's kubeconfig or Git write token: the current AX API itself has no built-in gRPC authentication interceptor/TLS ([access boundary](brain-to-ax-interface.md)). AX does not supply this merge/review/result protocol.

For example, after the three markers exist, the trusted coordinator can save the outputs (the redirection is on the **coordinator**, not inside an actor):

```bash
for task in wiki-overview wiki-team wiki-k3s; do
  ax ssh "$task" -- cat /workspace/base.sha > "$task.base"
  ax ssh "$task" -- cat /workspace/change.patch > "$task.patch"
done
```

With a trusted local clone in `integration/`, one mechanical gate looks like this. Inspect each patch's paths and content before applying it; the loop only checks that every agent started at the same commit and that patches apply cleanly.

```bash
base=$(git -C integration rev-parse HEAD)
for task in wiki-overview wiki-team wiki-k3s; do
  test "$base" = "$(cat "$task.base")" || exit 1
  git -C integration apply --stat "../$task.patch"  # compare with the role's allowed file
  git -C integration apply --check "../$task.patch" || exit 1
  git -C integration apply "../$task.patch" || exit 1
done
git -C integration diff --check
```

It then compares each `.base` to its recorded starting commit, checks path ownership and patch applicability, reviews the actual changes, and integrates them in one branch. The submitted manifest is a *fan-out specification of three independent executions*; the fan-in is coordinator code, not an AX DAG. If the three agents instead had to edit each other's output, the coordinator would apply one patch, publish a new revision or provide a result artifact, and only then start the dependent Task ([team boundary](ax-does-not-model-agent-teams.md)).

**Verification boundary:** No AX, Agent Substrate, Gemini API, or remote k3s instance was run for this example. The current k3s cluster's compatibility, access to the image and model API, and the helper's ability to produce useful edits are unverified. The full AX path requires [remote-k3s prerequisites](remote-k3s.md); the code here is a source-grounded demonstration of responsibilities and handoff, not a report of three successful edits.
