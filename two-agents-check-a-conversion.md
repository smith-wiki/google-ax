# Two AX tasks can draft and independently check a tiny answer

**Problem:** prepare a short FAQ explaining how to convert Celsius to Fahrenheit. One agent writes a draft; a second agent independently checks the formulas and two examples. A person compares the reports and publishes only a consistent answer. This is deliberately a *parallel* two-agent exercise: the checker does not read the writer's draft.

```
              brief: Celsius ↔ Fahrenheit
                     /           \
              draft agent     check agent
                     \           /
                person compares reports
```

AX describes the two isolated executions, **not** this diagram as a dependency graph. `Task.spec.command` launches the actual agent program; AX prepares its sandbox and exposes its lifecycle. Here both tasks use the Antigravity helper already present in the example runner image as a **demonstration command**, asking it to write a report in its workspace. The helper was written for workspace *setup*, so this is a small teaching example rather than a recommended production agent harness. For production, package a purpose-built agent command in an image that retains `/usr/local/bin/ax-task-runner` ([runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L9-L26), [image contents](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/Dockerfile.task-runner#L15-L31), [helper implementation](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax-task-runner/antigravity_bootstrap.py#L74-L125)).

## Illustrative manifest

```yaml
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: conversion-draft
  atespace: default
spec:
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:3a0dea6ad8b55278685db58aca6e37dc4ba04056831d45bef3aaeafdca43cac6"
  command:
    - python3
    - /usr/local/bin/antigravity_bootstrap.py
    - --goal
    - "Prepare /workspace/draft.md: a short Celsius-to-Fahrenheit FAQ with both conversion formulas and worked examples for 20 C and -40 C. Write the actual answer to that file."
  debug: true
---
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: conversion-check
  atespace: default
spec:
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:3a0dea6ad8b55278685db58aca6e37dc4ba04056831d45bef3aaeafdca43cac6"
  command:
    - python3
    - /usr/local/bin/antigravity_bootstrap.py
    - --goal
    - "Prepare /workspace/check.md: independently verify Celsius-to-Fahrenheit and inverse formulas, calculate 20 C and -40 C in F, and list likely conversion mistakes. Write the checks to that file."
  debug: true
```

No `Workspace` is needed for this fixed brief: the default runner has an empty `/workspace` for each Task. These are **separate** task sandboxes, not two agents sharing a directory ([simple upstream example](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/examples/simple.yaml#L15-L33), [runner volume and command contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L9-L26)). The helper requires `GEMINI_API_KEY`; the AX controller can inject it from its configured Kubernetes Secret in the task's atespace. **Never put the key in `spec.env` or the Markdown file** ([helper](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax-task-runner/antigravity_bootstrap.py#L104-L125), [secret lookup and injection](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/internal/controller/reconciler.go#L144-L155)). This also assumes a compatible AX/Agent Substrate cluster and reachable image on the worker architecture; check [remote k3s prerequisites](remote-k3s.md).

## How the result would be collected

On a prepared cluster, save the manifest as `conversion-team.yaml` and, using a trusted AX client, run:

```bash
ax apply -f conversion-team.yaml
ax watch task conversion-draft
ax watch task conversion-check
ax ssh conversion-draft -- cat /workspace/draft.md
ax ssh conversion-check -- cat /workspace/check.md
```

The two `ax ssh` reads must happen **after the report files exist**; a missing file or inconsistent answer means the human coordinator must investigate rather than publish. `debug: true` is required for `ax ssh` and permits powerful guest access, so it is appropriate only for this inspection demo ([CLI commands](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax/main.go#L168-L194), [guest access](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L41-L47)). `ax watch` reports sandbox readiness, **not** that the agent completed successfully: the runner logs child exit but keeps the sandbox alive and AX does not receive that exit code ([runner behavior](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/runner/runner.go#L167-L199), [runner contract](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L41-L43)).

A correct **expected** answer, not a claimed agent output, is `F = C × 9/5 + 32`, `C = (F − 32) × 5/9`; `20 °C = 68 °F` and `−40 °C = −40 °F`. If the checker instead needs to inspect the writer's *actual* draft, a coordinator must explicitly transfer that report to a new task or other shared artifact store before launching the reviewer. AX does not declare such a handoff or `after: conversion-draft` edge; see [what AX does not model](ax-does-not-model-agent-teams.md).

**Verification boundary:** this manifest is source-grounded and illustrative; no agent, model API, AX control plane, or remote cluster was run for this example. Whether this prompt causes the setup helper to write both files is unverified. It demonstrates the resource shape and human handoff, not an observed successful run.
