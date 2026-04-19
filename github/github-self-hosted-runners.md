# GitHub Actions Self-Hosted Runner on Kubernetes (ARC v2)

Complete setup guide for running GitHub Actions self-hosted runners as Kubernetes pods with Docker-in-Docker (DinD) support. This enables your CI/CD pipelines to access private/internal resources without exposing them to the public internet.

---

## Prerequisites

- A running Kubernetes cluster with `kubectl` configured
- `helm` v3 installed
- A GitHub repository or organization
- Admin access to the GitHub repo/org for runner registration

---

## Step 1: Install cert-manager

### What is it?
cert-manager is a Kubernetes-native certificate management controller. It automates the creation and renewal of TLS certificates.

### Why do we need it?
ARC's controller uses webhooks that require TLS certificates. Without cert-manager, the ARC controller installation will fail with errors like:
```
no matches for kind "Certificate" in version "cert-manager.io/v1"
```

### Install

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

### Verify it's working

Wait for all deployments to be ready:

```bash
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
```

Check that all three pods (webhook, cainjector, controller) are running:

```bash
kubectl get pods -n cert-manager
```

Expected output — all pods should show `Running`:
```
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-xxxxxxxxx-xxxxx              1/1     Running   0          2m
cert-manager-cainjector-xxxxxxxxx-xxxxx   1/1     Running   0          2m
cert-manager-webhook-xxxxxxxxx-xxxxx      1/1     Running   0          2m
```

### Uninstall (if needed)

```bash
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## Step 2: Install ARC v2 Controller

### What is it?
Actions Runner Controller (ARC) is the official Kubernetes operator from GitHub that manages self-hosted runners. The controller watches for workflow jobs and automatically creates/destroys runner pods on demand.

### Why do we need it?
The controller is the brain of the system. It communicates with GitHub's API, listens for incoming workflow jobs, and orchestrates runner pods in your cluster. Without it, there's nothing to manage the runner lifecycle.

### Install

```bash
helm install arc --namespace arc-systems --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

### Verify it's working

Check the controller pod is running:

```bash
kubectl get pods -n arc-systems
```

Expected output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
arc-gha-rs-controller-xxxxxxxxx-xxxxx   1/1     Running   0          1m
```

Check controller logs for any errors:

```bash
kubectl logs -n arc-systems -l app.kubernetes.io/part-of=gha-rs-controller --tail=50
```

If the label selector returns nothing, get the pod name and query directly:

```bash
kubectl get pods -n arc-systems
kubectl logs -n arc-systems <controller-pod-name> --tail=50
```

### Uninstall (if needed)

```bash
helm uninstall arc -n arc-systems
```

---

## Step 3: Create a GitHub Personal Access Token (Classic)

### What is it?
A Personal Access Token (PAT) is an authentication credential that allows ARC to communicate with GitHub's API on your behalf — specifically to register and manage self-hosted runners.

### Why does it need to be a Classic PAT?
Fine-grained PATs have known compatibility issues with ARC v2. They often result in `403 Forbidden` errors like:
```
Resource not accessible by personal access token
```
Classic PATs with the correct scopes work reliably.

### Create the token

1. Go to **https://github.com/settings/tokens**
2. Click **Generate new token** → **Classic**
3. Give it a descriptive name (e.g., `arc-k8s-runner`)
4. Select these scopes:
   - **`repo`** — full control of private repositories
   - **`admin:org`** — needed for org-level runner registration
   - **`manage_runners:org`** — needed to register/deregister runners
5. Click **Generate token** and copy it immediately (you won't see it again)

### Verify

Test the token works by calling the GitHub API:

```bash
curl -H "Authorization: token YOUR_PAT_HERE" https://api.github.com/user
```

You should see your GitHub user profile JSON. If you get a `401`, the token is invalid.

### Revoke (if needed)

Go to **https://github.com/settings/tokens** and delete the token.

---

## Step 4: Create the Kubernetes Secret for GitHub Auth

### What is it?
A Kubernetes Secret that stores your GitHub PAT securely inside the cluster. The runner pods will use this secret to authenticate with GitHub when registering as self-hosted runners.

### Why do we need it?
ARC needs credentials to talk to GitHub's API. Storing the PAT in a Kubernetes Secret keeps it encrypted at rest and avoids hardcoding credentials in Helm values.

### Install

```bash
kubectl create namespace arc-runners
kubectl create secret generic github-auth --namespace arc-runners --from-literal=github_token=YOUR_PAT_HERE
```

Replace `YOUR_PAT_HERE` with the token from Step 3.

### Verify it's working

```bash
kubectl get secret github-auth -n arc-runners
```

Expected output:
```
NAME          TYPE     DATA   AGE
github-auth   Opaque   1      1m
```

### Uninstall / Update (if needed)

To delete and recreate with a new token:

```bash
kubectl delete secret github-auth -n arc-runners
kubectl create secret generic github-auth --namespace arc-runners --from-literal=github_token=YOUR_NEW_PAT
```

After updating the secret, restart the controller to pick up the change:

```bash
kubectl rollout restart deployment arc-gha-rs-controller -n arc-systems
```

---

## Step 5: Install the Runner Scale Set with Docker-in-Docker

### What is it?
The Runner Scale Set is the actual pool of runner pods that execute your GitHub Actions workflows. It defines how many runners can exist, what image they use, and what capabilities they have.

### Why Docker-in-Docker (DinD)?
By default, runner pods don't have Docker installed. If your workflows build Docker images (e.g., `docker build`, `docker push`), they will fail with:
```
failed to connect to the docker API at unix:///var/run/docker.sock
```
The `containerMode.type="dind"` flag adds a Docker daemon as a sidecar container to each runner pod, providing full Docker support.

### Install

**For an organization:**

```bash
helm install my-runners --namespace arc-runners oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set --set githubConfigUrl="https://github.com/<org>" --set githubConfigSecret=github-auth --set containerMode.type="dind"
```

Replace `<org>` with your GitHub org name (e.g., `hymdall`).

**For a specific repository:**

```bash
helm install my-runners --namespace arc-runners oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set --set githubConfigUrl="https://github.com/<org>/<repo>" --set githubConfigSecret=github-auth --set containerMode.type="dind"
```

Replace `<org>/<repo>` with your repo path (e.g., `hymdall/payment-frontend`).

### Verify it's working

**Check runner pods:**

```bash
kubectl get pods -n arc-runners
```

You should see a listener pod like `my-runners-*-listener` in `Running` state.

**Check the runner scale set:**

```bash
kubectl get autoscalingrunnersets -n arc-runners
```

Expected output with populated fields:
```
NAME         MINIMUM RUNNERS   MAXIMUM RUNNERS   CURRENT RUNNERS   STATE   ...
my-runners   ...               ...               ...               ...
```

**Check on GitHub:**

- For org: Go to **https://github.com/organizations/<org>/settings/actions/runners**
- For repo: Go to your repo → **Settings** → **Actions** → **Runners**

Your self-hosted runner should appear in the list.

**Check controller logs for errors:**

```bash
kubectl get pods -n arc-systems
kubectl logs -n arc-systems <controller-pod-name> --tail=100
```

Look for `ERROR` lines. Common issues:
- `403 Forbidden` → PAT permissions issue (see Step 3)
- Connection errors → cluster can't reach `api.github.com` (check network/firewall)

### Uninstall (if needed)

```bash
helm uninstall my-runners -n arc-runners
```

---

## Step 6: Test with a Workflow

### What is this?
A simple GitHub Actions workflow to confirm the self-hosted runner is working, has Docker access, and can reach your internal services.

### Why?
Before using the runner in production workflows, you should verify that the full chain works: GitHub triggers a job → ARC spins up a runner pod → pod executes the job → pod has Docker and internal network access.

### Create the test workflow

Create `.github/workflows/test-runner.yaml` in your repository:

```yaml
name: Test Self-Hosted Runner
on: workflow_dispatch

jobs:
  test:
    runs-on: my-runners
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Verify Docker is available
        run: docker version

      - name: Test Docker build
        run: |
          echo "FROM alpine:latest" > /tmp/Dockerfile.test
          echo "RUN echo hello" >> /tmp/Dockerfile.test
          docker build -f /tmp/Dockerfile.test /tmp

      - name: Test internal network access
        run: curl -s --max-time 5 http://<internal-ip>:<port> || echo "Cannot reach internal service"
```

Replace `<internal-ip>:<port>` with an actual internal service address.

### Run the test

1. Go to your repo on GitHub
2. Click the **Actions** tab
3. Select **Test Self-Hosted Runner** from the left sidebar
4. Click **Run workflow**

### Verify

All steps should pass with green checkmarks. If the workflow is stuck on "Queued", the runner is not properly registered — go back and check Step 5.

---

## Step 7: Use in Production Workflows

### Example: Build and Push Docker Image

```yaml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: my-runners
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build Docker Image
        run: docker build -t your-registry/your-image:${{ github.sha }} .

      - name: Push Docker Image
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login your-registry -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
          docker push your-registry/your-image:${{ github.sha }}

      - name: Deploy to internal service
        run: |
          curl -X POST http://<internal-ip>:<port>/deploy \
            -H "Content-Type: application/json" \
            -d '{"image": "your-registry/your-image:${{ github.sha }}"}'
```

### Key points
- Always use `runs-on: my-runners` to target your self-hosted runners
- The runner has full access to your internal/private network since it runs inside your K8s cluster
- Docker builds and pushes work because of the DinD sidecar
- Store registry credentials in **GitHub Secrets** (repo → Settings → Secrets and variables → Actions)

---

## Quick Reference: Useful Commands

```bash
# --- cert-manager ---
kubectl get pods -n cert-manager

# --- ARC Controller ---
kubectl get pods -n arc-systems
kubectl logs -n arc-systems <controller-pod-name> --tail=100

# --- Runners ---
kubectl get pods -n arc-runners
kubectl get autoscalingrunnersets -n arc-runners
kubectl get events -n arc-runners --sort-by='.lastTimestamp'

# --- Helm releases ---
helm list -n arc-systems -a
helm list -n arc-runners -a

# --- Full cleanup (remove everything) ---
helm uninstall my-runners -n arc-runners
helm uninstall arc -n arc-systems
kubectl delete secret github-auth -n arc-runners
kubectl delete namespace arc-runners
kubectl delete namespace arc-systems
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## Troubleshooting Quick Reference

| Error | Cause | Fix |
|-------|-------|-----|
| `no matches for kind "Certificate"` | cert-manager not installed | Run Step 1 |
| `no matches for kind "AutoscalingRunnerSet"` | Wrong ARC version installed | Uninstall and install ARC v2 (Step 2) |
| `cannot reuse a name that is still in use` | Old Helm release exists | `helm uninstall my-runners -n arc-runners` then reinstall |
| `403 Forbidden: Resource not accessible by personal access token` | Wrong PAT type or missing scopes | Create a Classic PAT with correct scopes (Step 3) and update secret (Step 4) |
| `docker.sock: no such file or directory` | DinD not enabled | Reinstall runner scale set with `--set containerMode.type="dind"` (Step 5) |
| No pods in `arc-runners` namespace | Runner scale set not installed or auth failure | Check controller logs and verify Steps 4-5 |
| Workflow stuck on "Queued" | Runner not registered with GitHub | Check GitHub Settings → Actions → Runners and controller logs |