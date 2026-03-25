# Managing Multiple K8s Kubeconfigs

## The Problem

You have multiple Kubernetes clusters — dev, staging, prod, client environments. By default, `kubectl` reads a single `~/.kube/config`, and `kubectl config use-context` mutates that file globally. Switch context in one terminal and every other terminal silently follows. One wrong `kubectl apply` and you're touching prod when you meant dev.

---

## How to Solve It

There are **two independent layers** to configure:

| Layer | Mechanism | What it controls |
|---|---|---|
| **File isolation** | `KUBECONFIG` env var | Which kubeconfig file(s) kubectl reads — scopes context to a shell session |
| **Context switching** | `kubectl config` / `kubectx` | Which cluster+user+namespace is active within that file |

You need both. File isolation without context management means you can't switch clusters. Context management without file isolation means global mutation affects all terminals.

---

## Core Concepts

### kubeconfig Structure

A kubeconfig has three lists: `clusters`, `users`, and `contexts`. A context binds one cluster + one user + one default namespace. `current-context` is the global pointer kubectl uses for every command.

```yaml
clusters:
- name: dgx
  cluster:
    server: https://10.0.0.1:6443
    certificate-authority-data: ...

users:
- name: kubernetes-admin
  user:
    client-certificate-data: ...

contexts:
- name: kubernetes-admin@kubernetes
  context:
    cluster: dgx
    user: kubernetes-admin
    namespace: default        # ← people forget this is here

current-context: kubernetes-admin@kubernetes
```

> Mental model: like a kubeconfig for SSH (`~/.ssh/config`) — named contexts each with their own credentials, all pointing to different servers. The three-way split (cluster/user/context) lets you mix-and-match without duplicating credentials.

---

### KUBECONFIG Env Var

By default kubectl reads `~/.kube/config`. Set `KUBECONFIG` to a colon-separated list of files and kubectl merges them in memory at runtime — no file is modified.

```bash
export KUBECONFIG=~/.kube/config:~/.kube/dgx-config:~/.kube/staging-config
kubectl config get-contexts   # sees all contexts from all three files
```

> Mental model: like `PATH` for binaries — multiple directories merged at runtime, nothing written to disk.

**Key gotcha:** `use-context` always writes to the **first file** in the `KUBECONFIG` list, not the file where the context was originally defined.

---

### kubectl config Commands

```bash
kubectl config get-contexts                           # list all contexts
kubectl config current-context                        # what's active now
kubectl config use-context <name>                     # switch (mutates disk globally)
kubectl config set-context --current --namespace=foo  # change default namespace
kubectl config view --minify | grep namespace         # verify active namespace
```

> **Warning:** `use-context` mutates `~/.kube/config` on disk. All terminals sharing that file are now on the new context — not just the one you typed in.

---

### kubectx / kubens

Ergonomic wrappers around `kubectl config`. Still mutates disk under the hood — convenience only, not safety.

```bash
kubectx              # fuzzy-pick context (with fzf installed)
kubectx prod         # switch to prod context
kubectx -            # back to previous context (like cd -)
kubens payments      # switch default namespace on current context
```

> The real value is `kubectx -` — toggles between two contexts rapidly, same as `cd -` in shell.

---

## The Safe Pattern: One File Per Cluster

Keep each cluster in its own kubeconfig file. Single-context files self-select — no `use-context` needed.

```bash
~/.kube/dgx-config       # only dgx context inside
~/.kube/prod-config      # only prod context inside
~/.kube/staging-config   # only staging context inside
```

Then set `KUBECONFIG` per terminal or per project:

```bash
# ad-hoc in terminal
export KUBECONFIG=~/.kube/dgx-config

# scoped to a single command
KUBECONFIG=~/.kube/prod-config kubectl get pods
```

---

## Step-by-Step Setup

### Step 1 — Split kubeconfigs by cluster

If you have everything in `~/.kube/config`, extract each cluster into its own file:

```bash
# Export a specific context to its own file
kubectl config view --minify --context=kubernetes-admin@kubernetes \
  --flatten > ~/.kube/dgx-config
```

---

### Step 2 — Shell functions for ad-hoc switching

Add to `~/.zshrc` or `~/.bashrc`:

```bash
function kctx-dgx()     { export KUBECONFIG=~/.kube/dgx-config; }
function kctx-prod()    { export KUBECONFIG=~/.kube/prod-config; }
function kctx-staging() { export KUBECONFIG=~/.kube/staging-config; }
```

No `use-context` needed — single-context files auto-select. New terminal = clean state, nothing carries over.

---

### Step 3 — direnv for project-based automation (best for teams)

Install direnv, then add `.envrc` to each project root:

```bash
# ~/projects/dgx-infra/.envrc
export KUBECONFIG=~/.kube/dgx-config
```

`cd` into the project → `KUBECONFIG` activates automatically. `cd` out → reverts. Zero manual switching. Context follows code directory, not memory.

```bash
direnv allow   # run once per project to trust the .envrc
```

---

## Verify Everything Works

```bash
# Confirm which file kubectl is reading
echo $KUBECONFIG

# Confirm active context and namespace
kubectl config current-context
kubectl config view --minify | grep namespace

# List all visible contexts (from merged files)
kubectl config get-contexts
```

---

## Gotchas

1. **`use-context` is global** — mutates the file on disk, affects every terminal using that file. Always use `KUBECONFIG` isolation instead of relying on `use-context` for safety.

2. **Context stores default namespace** — switching context silently changes your default namespace. Always verify with `kubectl config view --minify | grep namespace` after a context switch.

3. **KUBECONFIG merge writes to the first file** — when you have a merged list (`KUBECONFIG=a:b:c`), `use-context` writes to file `a`, even if the context you switched to lives in file `b`. Can corrupt your first file unexpectedly.

4. **Tools like `aws eks update-kubeconfig` and `gcloud container clusters get-credentials` write their own files** — they don't know about your other clusters. Use the `--kubeconfig` flag to redirect their output to a dedicated file rather than letting them append to `~/.kube/config`.

5. **`kubectx` is ergonomics, not safety** — it still calls `use-context` under the hood. Don't confuse convenience with isolation.

---

## Quick Reference

**Core commands**
```bash
kubectl config get-contexts                    # list all contexts
kubectl config current-context                 # what's active
kubectl config use-context <name>              # switch (global, mutates disk)
kubectl config set-context --current \
  --namespace=<ns>                             # change default namespace
kubectl config view --minify | grep namespace  # verify active namespace
```

**Isolation patterns**
```bash
export KUBECONFIG=~/.kube/project-file         # scoped to this shell session
KUBECONFIG=~/.kube/prod kubectl get pods       # scoped to single command
```

**Extract a context to its own file**
```bash
kubectl config view --minify --context=<name> --flatten > ~/.kube/new-file
```

**direnv per-project**
```bash
# .envrc
export KUBECONFIG=~/.kube/project-config
direnv allow
```

| Pattern | Use when |
|---|---|
| Shell function (`kctx-dgx`) | Ad-hoc switching, single developer |
| `KUBECONFIG` inline | One-off command against a specific cluster |
| direnv `.envrc` | Project has a dedicated cluster, team environment |
| Merged `KUBECONFIG` list | Aggregating files from multiple tools (aws, gcloud) |
| `kubectx -` | Rapidly toggling between two clusters |
