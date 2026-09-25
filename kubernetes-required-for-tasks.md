# A real AX Task needs Kubernetes; a local runner experiment does not

AX schedules Tasks on Agent Substrate actors in a Kubernetes cluster. `ax apply`, `ax ssh`, and suspend/resume therefore need an AX control plane and Substrate running there; installing only the CLI on a Mac does not provide them. [AX prerequisites](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/README.md#L59-L91) · [Substrate's Kubernetes role](https://github.com/agent-substrate/substrate/blob/22ba860e0f32e20713165b96abd81f6f96da620c/README.md#L5-L14).

Without Kubernetes, the CLI's `help` and `version` commands work, and the runner accepts local Task/Workspace YAML files for a limited execution probe. That exercises neither the sandbox nor the AX API. [CLI dispatch](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/cmd/ax/main.go#L100-L127) · [Runner local mode](https://github.com/google/ax/blob/e09ed1bc5463ad4b5ca88f755a6e1e2005b3c7b7/docs/runner.md#L153-L165).

The [Mac guide](getting-started-on-mac.md) details a local probe; the [remote k3s card](remote-k3s.md) assesses using an existing cluster instead of local kind.
