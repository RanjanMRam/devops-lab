# Day 04 – Mon 23 Nov 2026: Git workflows, Kubernetes security and DevSecOps

## Part 1: Git workflows

### Branching strategies

| Strategy | How it works | Good for |
|---|---|---|
| GitFlow | Long-lived `main` and `develop`, plus `feature/*`, `release/*`, `hotfix/*` | Planned, versioned releases |
| Trunk-based | Small changes merged into `main` at least daily, feature flags hide unfinished work | CI/CD and DevOps teams |
| GitHub flow | `main` plus short feature branches merged by pull request | Simple continuous deployment |

Interview answer: "For CI/CD I prefer trunk-based development with short-lived branches and pull requests: small merges, fast CI feedback, fewer conflicts."

### Commands I practised

```bash
git switch -c feature/git-practice            # new branch
git merge --no-ff feature/git-practice -m "Merge"  # merge with a merge commit
git log --oneline --graph -5                  # see history as a graph

# Conflict: edit the same line on two branches, merge, then
cat notes/git-practice.txt                    # <<<<<<< ======= >>>>>>> markers
nano notes/git-practice.txt                   # keep what you want, delete markers
git add notes/git-practice.txt
git commit -m "Resolve merge conflict"

git rebase main                               # replay my commits on top of latest main
git cherry-pick <commit-id>                   # copy one commit to this branch
git revert HEAD --no-edit                     # undo with a new commit (safe on shared branches)
git reset --soft HEAD~1                       # remove last commit, keep changes (local only)
git stash && git stash pop                    # park work, bring it back
git reflog                                    # recover "lost" commits
```

### Merge vs rebase

Before, both branches have a new commit:
```
                 C  <- feature
   A --- B -----<
                 D  <- main
```
- **Merge** joins them with a merge commit M. True history, with a fork.
- **Rebase** replays C on top of D as a new commit C' (new ID). Straight line, no merge commit:
```
   A --- B --- D --- C'  <- feature
```

| | Merge | Rebase |
|---|---|---|
| History | True record, forks | Straight line |
| Commit IDs | Unchanged | Rewritten |
| Shared branches | Safe | Only on my own unpushed branches |

Golden rule: **never rebase a branch others have pulled.**

Interview answer: "Merge preserves exact history with a merge commit. Rebase gives a linear history but new commit IDs. I rebase my own feature branch onto main before a pull request, but never shared branches."

### Cherry-pick conflict

- Cherry-pick copies one commit's change, not the commits before it.
- My "Hotfix line 3" depended on "line 2", which main did not have, so it conflicted.
- Fix: edit the file, remove markers, `git add`, then `git cherry-pick --continue`.
- Other options: `--abort` (go back), `--skip` (drop this commit). Same pattern for `git rebase --continue / --abort`.

### Pull requests and branch protection

- Push a branch, open a PR on GitHub, review Files changed, squash and merge, delete the branch.
- Branch protection (require PR, approvals, status checks) would block Jenkins pushing to main.
- Fix in real life: a bot account with bypass, or CI opens an automatic PR.

## Part 2: Kubernetes security (namespace `sec-lab`)

### RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: viewer
  namespace: sec-lab
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: sec-lab
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: viewer-reads-pods
  namespace: sec-lab
subjects:
  - kind: ServiceAccount
    name: viewer
    namespace: sec-lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
```bash
SA=system:serviceaccount:sec-lab:viewer
kubectl auth can-i list pods -n sec-lab --as=$SA      # yes
kubectl auth can-i delete pods -n sec-lab --as=$SA    # no
kubectl auth can-i list pods -n demo --as=$SA         # no (Role is namespace-scoped)
```
- Role = one namespace, ClusterRole = cluster-wide
- RoleBinding / ClusterRoleBinding attach them to users, groups or service accounts
- Principle: least privilege

### NetworkPolicy

- `default-deny-ingress` with `podSelector: {}` blocks all incoming traffic in the namespace
- `allow-from-approved` lets only pods labelled `access: "true"` reach `app: web` on port 80
```bash
kubectl run test --rm -it --image=busybox -n sec-lab --restart=Never -- wget -qO- -T 3 web   # blocked
kubectl run ok --rm -it --image=busybox -n sec-lab --restart=Never --labels=access=true -- wget -qO- -T 3 web   # works
```
- NetworkPolicies only work if the CNI supports them (Calico, Cilium)
- Interview answer: "Default deny per namespace, then allow only what each service needs: zero trust inside the cluster."

### Secrets

```bash
kubectl create secret generic db-cred -n sec-lab \
  --from-literal=username=admin --from-literal=password='S3cret!'
kubectl get secret db-cred -n sec-lab -o jsonpath='{.data.password}' | base64 -d
```
- base64 is encoding, not encryption
- Secure them with: encryption at rest for etcd, tight RBAC, never plain secrets in Git (Sealed Secrets, SOPS), or HashiCorp Vault / AWS Secrets Manager with External Secrets Operator

### securityContext

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: web
      image: nginxinc/nginx-unprivileged:alpine
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```
```bash
kubectl exec secure-pod -n sec-lab -- id                 # uid=101, not root
kubectl exec secure-pod -n sec-lab -- touch /etc/hack    # Read-only file system (the test passes)
kubectl exec secure-pod -n sec-lab -- touch /tmp/ok      # works: the emptyDir is writable
```
- `kubectl exec <pod> -n <ns> -- <command>` runs a command inside the container; `--` separates kubectl options from the container command
- `runAsNonRoot`: refuses to start as root
- `readOnlyRootFilesystem`: attacker cannot save tools or change config; `/tmp` emptyDir is the one writable place
- `capabilities drop ALL`: removes root powers
- `allowPrivilegeEscalation: false`: cannot gain privileges later
- `seccompProfile RuntimeDefault`: limits system calls
- Plain `nginx:alpine` with `runAsNonRoot` fails with CreateContainerConfigError because the image runs as root

### Pod Security Admission (PSA)

```bash
kubectl create namespace restricted-lab
kubectl label namespace restricted-lab pod-security.kubernetes.io/enforce=restricted
kubectl run bad --image=nginx -n restricted-lab     # rejected: violates PodSecurity "restricted"
sed 's/namespace: sec-lab/namespace: restricted-lab/' secure-pod.yaml | kubectl apply -f -   # accepted
```
- securityContext is voluntary per pod. PSA makes security compulsory for a whole namespace: the API server rejects insecure pods before they are created.
- `sed 's/old/new/' file | kubectl apply -f -` edits the YAML on the fly and pipes it to kubectl (`-` = read from the pipe)
- Levels: privileged (anything), baseline (blocks the most dangerous), restricted (strict best practice)
- Modes: enforce (reject), warn (allow with warning), audit (allow, log it)
- Roll out with warn and audit first, then enforce
- PSA replaced PodSecurityPolicy (removed in Kubernetes 1.25); custom rules need Kyverno or OPA Gatekeeper

## Part 3: DevSecOps concepts

| Practice | Checks | When | Tools |
|---|---|---|---|
| SAST | Source code for insecure patterns | Every commit or PR | SonarQube, Semgrep, Checkmarx |
| SCA | Open-source dependencies with known CVEs | Every build | Snyk, OWASP Dependency-Check, Black Duck |
| Secrets scanning | Passwords and keys in Git | Pre-commit, PRs | gitleaks, GitGuardian |
| IaC scanning | Terraform and Kubernetes YAML misconfigurations | PRs | Checkov, tfsec, kube-linter |
| Image scanning | Packages in container images | After build, before push | Trivy (mine), Grype |
| DAST | The running app, attacked from outside | Test or staging | OWASP ZAP, Burp Suite |
| Runtime security | Suspicious behaviour in containers | Production | Falco |
| Supply chain | Image signing, SBOM | Build and deploy | cosign, Syft |

**Vault vs Kubernetes Secrets:** Kubernetes Secrets live in etcd, base64 only unless encryption at rest is on, no rotation or audit. Vault gives central storage, policies, audit logs, rotation and dynamic secrets.

## Interview questions I can now answer

1. How would you add security to a CI/CD pipeline? Shift left and defence in depth: secrets scanning and SAST on PRs, SCA and image scanning in the build, IaC scanning, DAST in staging, image signing, then RBAC, NetworkPolicies and Pod Security at runtime.
2. SAST vs DAST: SAST reads the code without running it, DAST attacks the running app.
3. How do you handle secrets in Kubernetes and GitOps?
4. How do you restrict what a pod can do? securityContext per pod, PSA per namespace, RBAC, NetworkPolicies.
5. Merge vs rebase, and when to cherry-pick?
