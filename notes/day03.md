# Day 03 – Sat 21 and Sun 22 Nov 2026: CI pipeline, Trivy and app of apps

## What I built
- App of apps: a `root` Argo CD app that manages every Application in `apps/` from Git
- My own app (`app/index.html` + `app/Dockerfile`) on a minimal Alpine base image
- Jenkins running in Docker, with Docker Hub and GitHub credentials
- A Jenkins pipeline: build, Trivy scan, push to Docker Hub, update the image tag in Git
- Argo CD deploys every new tag automatically (full CI/CD + GitOps flow)

## The full flow

```
You push code
   -> Jenkins: build -> Trivy scan -> push to Docker Hub -> commit new tag to Git
                          |
                          +-- critical vulnerability? build fails, nothing is pushed or deployed
   -> GitHub: values.yaml now has tag "N"
   -> Argo CD (polls every 60s): root app -> demo-app -> sync
   -> Kubernetes: rolling update, pods pull ranjanmr1989/demo-app:N
```
Key point: **Jenkins never touches the cluster.** CI ends at Git, Argo CD is the only thing that deploys.

## App of apps

`bootstrap/root-app.yaml` (kept outside `apps/` so the root app does not manage itself):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/RanjanMRam/devops-lab.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
```bash
kubectl apply -f bootstrap/root-app.yaml   # the only manual kubectl apply
argocd app list                            # root and demo-app
```
Test: added a `team: devops` label to `apps/demo-app.yaml`, pushed, and it appeared without `kubectl apply`.

## App and Dockerfile

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```
```bash
docker build -t demo-app:test app/
docker run --rm -d -p 9090:80 --name demo-test demo-app:test
curl localhost:9090
docker stop demo-test
```
- `-t` names the image, `app/` is the build context (where the Dockerfile and copied files live)
- Alpine images are small and have fewer vulnerabilities

## Image names

`docker.io / ranjanmr1989 / demo-app : 7` = registry / namespace / repository / tag
- Namespace must be my Docker Hub username (controls who can push)
- Repository name is my choice: lowercase, letters, numbers, `.`, `_`, `-`
- Docker Hub repo is public so the cluster can pull without an image pull secret
- In companies: private registries (ECR, Harbor) with pull secrets or IAM roles

## Jenkins in Docker

```bash
docker run -d --name jenkins --restart unless-stopped \
  -p 8081:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk17
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y docker.io git"
docker exec -u root jenkins chmod 666 /var/run/docker.sock
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
Credentials (Username with password): `dockerhub` (username + Docker Hub access token), `github` (username + personal access token with repo scope).

## Jenkinsfile (repo root)

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "ranjanmr1989/demo-app"
    TAG   = "${BUILD_NUMBER}"
  }
  stages {
    stage('Build') {
      steps { sh 'docker build -t $IMAGE:$TAG app/' }
    }
    stage('Security scan (Trivy)') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --exit-code 1 --severity CRITICAL --ignore-unfixed $IMAGE:$TAG
        '''
      }
    }
    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub',
            usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
          sh '''
            echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
          '''
        }
      }
    }
    stage('Update manifest (GitOps)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github',
            usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh '''
            git config user.email "jenkins@devops-lab"
            git config user.name "jenkins"
            sed -i "s|^  repository:.*|  repository: $IMAGE|" charts/demo-app/values.yaml
            sed -i "s|^  tag:.*|  tag: \\"$TAG\\"|" charts/demo-app/values.yaml
            git add charts/demo-app/values.yaml
            git commit -m "ci: deploy $IMAGE:$TAG"
            git push https://$GH_USER:$GH_TOKEN@github.com/$GH_USER/devops-lab.git HEAD:main
          '''
        }
      }
    }
  }
}
```

What each part does:
- `sh '...'` runs a shell command in the Jenkins workspace. Single quotes let the shell expand variables, which keeps secrets out of logs.
- Build: `docker build` tagged with the build number.
- Trivy: scans the image, `--exit-code 1` fails the build on CRITICAL vulnerabilities, so later stages never run (security quality gate, shift left).
- Push: `withCredentials` injects the token, masked in logs. `--password-stdin` keeps it out of process lists.
- Update manifest: `sed` writes the new repository and tag into `values.yaml`, then Jenkins commits and pushes to Git.

## How tagging works

- `TAG = BUILD_NUMBER`, so every build gets a unique tag (1, 2, 3...)
- The same tag is built, scanned, pushed and written to `values.yaml`
- Helm template: `image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`
- Argo CD sees the new tag, Kubernetes does a rolling update
- The tag is quoted (`"7"`) so YAML keeps it as text
- Never use `latest`: not traceable, may not trigger a rollout, no clean rollback
- Alternatives: Git commit hash, semantic version, image digest (`@sha256:...`)

## Security gate test

Changed the base image to `FROM nginx:1.16`: the Trivy stage failed with critical CVEs, and Push and Update manifest never ran. Changed back to `nginx:alpine` and it passed.

## Problems I hit and how I fixed them

- **`argocd` said "token is expired":** logins last 24 hours. Fix: `argocd login localhost:8080 --username admin --password <pw> --insecure`
- **Port-forward "broken pipe" messages:** harmless, leave the terminal running.
- **Jenkins "permission denied ... docker.sock":** the permission resets when Docker Desktop restarts. Fix with `~/jenkins-fix.sh`:
  ```bash
  docker start jenkins
  docker exec -u root jenkins chmod 666 /var/run/docker.sock
  docker exec jenkins docker ps > /dev/null && echo "Jenkins can use Docker"
  ```
- **Local push rejected:** Jenkins committed to main in the meantime. Always `git pull` first.

## Interview answers

- **The full pipeline (under a minute):** "Jenkins builds a Docker image tagged with the build number and scans it with Trivy, failing on critical vulnerabilities. It pushes clean images to the registry and updates the image tag in the Helm values in Git. Argo CD watches that repo and deploys the change. CI never has cluster access, every release is a traceable Git commit, and rollback is a git revert."
- **Docker socket in Jenkins:** fine in a lab, but it gives Jenkins root on the host. In production, build on Kubernetes agents with Kaniko or Buildah.
- **App of apps:** "The only manual step is bootstrapping the root app. After that, everything, including the app definitions, comes from Git."
- **Branch protection vs CI pushes:** use a bot account allowed to bypass, or have CI open an automatic pull request (like Argo CD Image Updater).
