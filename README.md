# ML-Workflow-PoC

## Prerequisites
- [k3d](https://k3d.io/stable/)
- [argocli](https://argo-workflows.readthedocs.io/en/latest/walk-through/argo-cli/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [argo workflow](https://argo-workflows.readthedocs.io/en/latest/)

## Steps
1. Create a cluster with two agent nodes 
```bash
k3d cluster create mycluster --agents 2 -p "8081:80@loadbalancer"
```
* check nodes if you need
    ```bash
    kubectl get nodes
    ```
  
* Label node to simulate hardwares
    ```bash
    kubectl label nodes k3d-mycluster-agent-0 hardware=gpu
    kubectl label nodes k3d-mycluster-agent-1 hardware=cpu
    kubectl get nodes --show-labels
    ```


2. Install Argo Workflows
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo argo/argo-workflows --namespace argo --create-namespace \
  --set workflow.serviceAccount.create=true \
  --set 'workflow.serviceAccount.name=argo-workflow' \
  --set 'server.authModes[0]=server' \
  --version 0.45.20
```

3. Install Argocli
```bash
# Detect OS
ARGO_OS="darwin"
if [[ uname -s != "Darwin" ]]; then
  ARGO_OS="linux"
fi

# Download the binary
curl -sLO "https://github.com/argoproj/argo-workflows/releases/download/v3.6.10/argo-$ARGO_OS-amd64.gz"

# Unzip
gunzip "argo-$ARGO_OS-amd64.gz"

# Make binary executable
chmod +x "argo-$ARGO_OS-amd64"

# Move binary to path
mv "./argo-$ARGO_OS-amd64" /usr/local/bin/argo

# Test installation
argo version
```

4. submit argo CLI
```bash
argo submit https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml --watch --serviceaccount argo-workflow
argo submit -n argo --watch --serviceaccount argo-workflow ./workflows/ci-cd-pipeline-dag.yaml
argo submit -n argo --watch --serviceaccount argo-workflow ./workflows/model-training-steps.yaml 
```

5. check running nodes
```bash
kubectl get pods -n argo -o wide
```
