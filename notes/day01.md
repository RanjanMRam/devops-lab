# Day 01 – Thu 19 Nov 2026: Lab setup and Argo CD concepts

## What I built
- WSL2 Ubuntu on Windows, with Docker Desktop's WSL integration turned on for Ubuntu
- Tools: kubectl, kind, Helm, argocd command-line tool
- A three-node local Kubernetes cluster (1 control plane, 2 workers) with kind
- GitHub repo `devops-lab` with a Helm chart `charts/demo-app`

## Setup commands

```bash
# Check Docker works from Ubuntu
docker version
docker run hello-world

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
sudo install -m 0755 kind /usr/local/bin/kind

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# argocd command-line tool
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd

# Verify
kubectl version --client && kind version && helm version && argocd version --client
```

## kind cluster

`kind-config.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name devops-lab --config kind-config.yaml
kubectl get nodes                     # 3 nodes, all Ready
kubectl get pods -A                   # system pods
kubectl describe node devops-lab-worker   # capacity, allocatable, conditions

# After a laptop restart
docker start devops-lab-control-plane devops-lab-worker devops-lab-worker2
```

## Helm chart

```bash
helm create charts/demo-app           # generates an nginx chart
helm lint charts/demo-app             # check the chart for errors
helm template demo charts/demo-app    # render the YAML without installing

# Manual install (push model) to compare with GitOps later
helm install demo charts/demo-app -n demo --create-namespace
kubectl get pods -n demo
kubectl port-forward svc/demo-demo-app 8080:80 -n demo
helm uninstall demo -n demo
```

## Argo CD concepts

| Concept | Meaning |
|---|---|
| GitOps | Git is the single source of truth for what runs in the cluster. Changes go through a commit or pull request, not `kubectl apply`. |
| Push vs pull | Jenkins pushes changes into the cluster. Argo CD runs inside the cluster and pulls from Git, so CI holds no cluster credentials. |
| Application | Argo CD object that links a Git repo and path to a cluster and namespace. |
| Project | Groups applications and limits which repos and clusters they can use. |
| Desired vs live state | Desired state is what is in Git. Live state is what is actually running in the cluster. |
| Sync status | Synced or OutOfSync: does the live state match Git? |
| Health status | Healthy, Progressing, Degraded or Missing: is the app actually working? |
| Auto-sync | Applies Git changes automatically. |
| Self-heal | Reverts manual changes made directly in the cluster. |
| Prune | Deletes resources that were removed from Git. |
| Rollback | Revert the commit in Git (preferred), or roll back to an earlier sync in Argo CD's history. |

## Problems I hit and how I fixed them

- **`curl: not found` at a `docker-desktop:~#` prompt:** I was inside Docker Desktop's internal Linux system, not Ubuntu. Fix: install Ubuntu (`wsl --install -d Ubuntu`), set it as default (`wsl --set-default Ubuntu`) and turn on WSL integration for Ubuntu in Docker Desktop settings.
- **`apt update` stuck at "Waiting for headers":** network or DNS issue in WSL. Check with `ping 8.8.8.8` and `ping archive.ubuntu.com`. Fix DNS by setting `nameserver 8.8.8.8` in `/etc/resolv.conf` with `generateResolvConf = false` in `/etc/wsl.conf`, then `wsl --shutdown`. Office VPNs can also block WSL.
- **Work in `~`, not `/mnt/c/...`:** the Windows drive is slow from Linux.
