# ML-Workflow-PoC

A small reference for orchestrating ML and CI/CD pipelines on Kubernetes with [Argo Workflows](https://argo-workflows.readthedocs.io/).

> **Status:** Proof of concept. Built as a reference for teammates evaluating Argo Workflows — not production-ready.

## What this project is

A minimal, self-contained playground that explores three distinct Argo Workflows patterns and demonstrates how to pin individual steps to specific node types (CPU vs GPU) using `nodeSelector`. Everything runs on a local [k3d](https://k3d.io/) cluster, so you can study the patterns end-to-end without any cloud infrastructure or real GPU hardware.

There is no application code in this repo — only Argo `Workflow` manifests under [`workflows/`](./workflows). Each workflow uses public container images so it runs as-is.

## What it demonstrates

- A sequential `steps:` template for an ML training pipeline (data prep → train → eval).
- A linear `dag:` template for a baseline CI/CD pipeline (build → test → scan → deploy).
- A branching `dag:` with `when:` conditionals that fan out into success/failure paths.
- Pinning individual steps to specific nodes via `nodeSelector`, using k3d node labels (`hardware=cpu`, `hardware=gpu`) to simulate heterogeneous hardware.

## Why three separate files

Each workflow file isolates **one** Argo pattern. Rather than cramming every concept into a single mega-workflow, the PoC keeps the patterns side-by-side so a reader can study them independently and copy whichever one fits their use case:

| File | Pattern | One-line summary |
|---|---|---|
| [`workflows/model-training-steps.yaml`](./workflows/model-training-steps.yaml) | Sequential `steps:` | ML training pipeline; the training step is pinned to a GPU node, the rest run on CPU. |
| [`workflows/cicd-pipeline-dag.yaml`](./workflows/cicd-pipeline-dag.yaml) | Linear `dag:` | Baseline CI/CD pipeline (build → test → scan → deploy), all CPU. |
| [`workflows/cicd-pipeline-complex-dag.yaml`](./workflows/cicd-pipeline-complex-dag.yaml) | Branching `dag:` with conditionals | Adds parallel `scan`/`lint` after a successful test and a failure-notification branch when the test fails. |

## Repository layout

```
.
├── README.md
├── CLAUDE.md
└── workflows/
    ├── model-training-steps.yaml         # steps: pattern, CPU + GPU
    ├── cicd-pipeline-dag.yaml            # dag:   linear
    └── cicd-pipeline-complex-dag.yaml    # dag:   branching with when:
```

## Prerequisites

- [k3d](https://k3d.io/stable/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Helm](https://helm.sh/)
- [Argo CLI](https://argo-workflows.readthedocs.io/en/latest/walk-through/argo-cli/)

## Quick start

### 1. Create a k3d cluster with two labeled agent nodes

```bash
k3d cluster create mycluster --agents 2 -p "8081:80@loadbalancer"

# Label the agents to simulate heterogeneous hardware
kubectl label nodes k3d-mycluster-agent-0 hardware=gpu
kubectl label nodes k3d-mycluster-agent-1 hardware=cpu

kubectl get nodes --show-labels
```

> Note: there is no real GPU here. The `hardware=gpu` label is purely a scheduling hint so you can see `nodeSelector` route the training step to a specific node.

### 2. Install Argo Workflows

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo argo/argo-workflows --namespace argo --create-namespace \
  --set workflow.serviceAccount.create=true \
  --set 'workflow.serviceAccount.name=argo-workflow' \
  --set 'server.authModes[0]=server' \
  --version 0.45.20
```

### 3. Submit the workflows

```bash
# ML training pipeline (steps:)
argo submit -n argo --watch --serviceaccount argo-workflow ./workflows/model-training-steps.yaml

# Linear CI/CD DAG
argo submit -n argo --watch --serviceaccount argo-workflow ./workflows/cicd-pipeline-dag.yaml

# Branching CI/CD DAG (default: test passes)
argo submit -n argo --watch --serviceaccount argo-workflow ./workflows/cicd-pipeline-complex-dag.yaml
```

### 4. Verify pod placement

```bash
kubectl get pods -n argo -o wide
```

You should see the `training` pod scheduled onto `k3d-mycluster-agent-0` (the `gpu`-labeled node) and the surrounding steps on `k3d-mycluster-agent-1` (`cpu`).

### 5. Trigger the failure branch in the complex DAG

The complex DAG accepts a `test_mode` parameter. Pass `fail` to make the test step throw, which causes the `when:` conditionals to skip `scan`/`lint`/`deploy` and instead run `failure-notification`:

```bash
argo submit -n argo --watch --serviceaccount argo-workflow \
  ./workflows/cicd-pipeline-complex-dag.yaml \
  -p test_mode=fail
```

## Reading the workflows as a reference

If you'd rather study the YAML than run it, here's where to look:

- **`steps:` vs `dag:` syntax** — compare the `templates[0]` blocks of `model-training-steps.yaml` and `cicd-pipeline-dag.yaml`.
- **`nodeSelector` hardware pinning** — every template body sets `nodeSelector: { hardware: cpu | gpu }`. The training step in `model-training-steps.yaml` is the one that targets `gpu`.
- **Conditional branching** — see the `when: "{{tasks.test.status}} == Succeeded"` and `== Failed` clauses in `cicd-pipeline-complex-dag.yaml`. The `test-task` template in that file shows how the `test_mode=fail` parameter is wired in to force the failure path.
- **Parameters** — both `training-step` (in `model-training-steps.yaml`) and `test-task` (in both DAG files) demonstrate `inputs.parameters` with default values that can be overridden via `argo submit -p`.

## Limitations & honest caveats

- **No application code.** All steps run public container images and print fake output. There's nothing real being built, tested, or trained.
- **GPU is simulated.** The `hardware=gpu` label is just a label; the training step uses a CUDA base image but does no GPU work.
- **No tests or CI for the manifests themselves.** YAML changes are not validated automatically.
- **Pinned versions.** The argo-workflows Helm chart is pinned to `0.45.20`; newer charts may need flag adjustments.
- **Local-only.** Setup assumes k3d on a single machine; nothing here is hardened for shared or remote clusters.

## Next steps

Ideas for anyone adapting this as a starting point:

- Replace the placeholder containers with real build/test/training images.
- Add [artifacts](https://argo-workflows.readthedocs.io/en/latest/walk-through/artifacts/) to pass data between steps.
- Promote shared templates into a `WorkflowTemplate` so multiple workflows can reuse them.
- Wire the DAG into a real trigger (Argo Events, a Git webhook, or `argo cron`).
