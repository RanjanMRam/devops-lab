# Day 02 – Fri 20 Nov 2026: Argo CD hands-on

## What I did
- Installed Argo CD in the cluster and logged in to the web interface and command-line tool
- Deployed `demo-app` from GitHub with an Argo CD Application (GitOps)
- Deployed changes through Git commits, then turned on auto-sync, self-heal and prune
- Watched self-heal fix manual changes (drift) and prune remove deleted resources
- Practised rollback and fixed a broken deployment (ImagePullBackOff)

## Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w

# Web interface (keep running in its own terminal tab)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Log in
argocd admin initial-password -n argocd
argocd login localhost:8080 --username admin --password <password> --insecure
```

## The Application

`apps/demo-app.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/RanjanMRam/devops-lab.git
    targetRevision: main
    path: charts/demo-app
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f apps/demo-app.yaml   # needed every time this file changes
argocd app get demo-app               # sync status, health, sync policy, revision
argocd app get demo-app --refresh     # check Git now
argocd app get demo-app --hard-refresh  # fetch Git again, ignoring the cached copy
argocd app diff demo-app              # what will change
argocd app sync demo-app              # manual sync
```

## Deploying a change through Git

```bash
# edit charts/demo-app/values.yaml, e.g. replicaCount: 3
git commit -am "Scale demo-app to 3 replicas" && git push
argocd app get demo-app --refresh
kubectl get pods -n demo -w
```

## How Argo CD detects Git changes

- **Polling:** checks Git every 3 minutes by default. I changed it to 60 seconds:
  ```bash
  kubectl -n argocd patch configmap argocd-cm --type merge \
    -p '{"data":{"timeout.reconciliation":"60s"}}'
  kubectl -n argocd rollout restart statefulset argocd-application-controller
  kubectl -n argocd rollout restart deployment argocd-repo-server
  ```
- **Webhook:** GitHub notifies Argo CD on every push (`https://<argocd-address>/api/webhook`), so it deploys within seconds. Used in production. A local lab needs a tunnel such as ngrok because GitHub cannot reach localhost.
- **Interview answer:** use webhooks for near-instant deployments and keep polling as a fallback.

## Self-heal (drift correction)

```bash
kubectl scale deployment demo-app -n demo --replicas=1   # manual change
kubectl get pods -n demo -w                              # Argo CD scales it back
kubectl delete svc demo-app -n demo
kubectl get svc -n demo                                  # recreated in seconds
```
Drift means the live cluster no longer matches Git. Self-heal puts Git's version back. Argo CD watches the cluster continuously, so this needs no refresh.

## Prune

- Added `charts/demo-app/templates/configmap.yaml` (`demo-config`), pushed, and it was created.
- Deleted the file, pushed, and Argo CD removed it from the cluster.
- `kube-root-ca.crt` is created by Kubernetes in every namespace. Argo CD never prunes it because it does not manage it.

## Rollback

```bash
# GitOps way (preferred): Git stays the single source of truth
git revert HEAD --no-edit && git push

# Argo CD way: the cluster then differs from Git
argocd app history demo-app
argocd app set demo-app --sync-policy none   # rollback needs auto-sync off
argocd app rollback demo-app <history-id>
kubectl apply -f apps/demo-app.yaml           # turn auto-sync back on
```

## Synced but not Healthy

- Set `image.tag: "doesnotexist"` and pushed.
- Result: Sync **Synced**, Health **Degraded/Progressing**, pods in **ImagePullBackOff**.
- Diagnose: `kubectl describe pod <pod> -n demo` and read the Events section.
- Lesson: Synced means the cluster matches Git. Healthy means the app actually works.

## Problems I hit and how I fixed them

- **Pushed auto-sync to GitHub but the app did not auto-deploy:** Argo CD uses the Application stored in the cluster, not the file in Git. Fix: `git pull`, then `kubectl apply -f apps/demo-app.yaml`. The app of apps pattern removes this manual step.
- **Edited files on the GitHub website:** run `git pull` before the next local change, or the push is rejected. Better to edit locally and push.
- **New file not deployed:** check it is in `templates/`, committed and pushed (`git status`), renders (`helm template`), then `argocd app get demo-app --hard-refresh`.
- **argocd commands say "connection refused":** the port-forward stopped. Restart it.

## Interview questions I can now answer

1. What is GitOps, and how is it better than Jenkins running `kubectl apply`?
2. What is the difference between Synced/OutOfSync and Healthy/Degraded? Example: Synced but Degraded because of a bad image tag.
3. What do selfHeal and prune do, and when would you turn them off?
4. How do you roll back in GitOps, and why is a Git revert preferred?
5. Someone changed production by hand with kubectl. What happens with Argo CD?
6. How does Argo CD detect changes in Git (polling vs webhooks)?
7. How do you manage the Argo CD Application definitions themselves (app of apps, ApplicationSets)?
