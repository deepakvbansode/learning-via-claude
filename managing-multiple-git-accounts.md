# Managing Multiple Git Accounts on One Machine

## The Problem

You have two (or more) GitHub accounts — say, a personal one and an office one — on the same laptop. By default, Git and SSH have no concept of "which account am I acting as right now." Every `git push` uses the same SSH key, and every commit carries the same name/email — leading to pushes rejected by the wrong account, or commits attributed to the wrong identity.

---

## How to Solve It

There are **two independent layers** to configure. They solve different problems:

| Layer | Config file | What it controls |
|---|---|---|
| **SSH** | `~/.ssh/config` | *Authentication* — which key (= which account) is used when talking to GitHub |
| **Git** | `~/.gitconfig` | *Authorship* — the name and email that appear in commits |

You need both. Fixing only SSH means you can push correctly but your commits still carry the wrong identity. Fixing only Git config means your name/email is right but pushes still fail or go to the wrong account.

---

## Layer 1 — SSH: Authentication

### Why it works this way

GitHub doesn't identify you by username in the SSH URL — it identifies you by **which public key you present**. When you register an SSH key in GitHub Settings, GitHub maps that key to your account. So if you present the right key, GitHub knows who you are.

The SSH client reads `~/.ssh/config` on every connection to pick the right key. You create **named aliases** (e.g. `github-personal`, `github-office`) that both point to `github.com` but use different key files. Git remote URLs then reference these aliases instead of `github.com` — that's what routes each repo to the correct account.

> Mental model: `~/.ssh/config` is like a **kubeconfig for SSH** — named contexts, each with its own credentials, all pointing to the same server. Git remotes reference the context name directly in the URL.

### Gotchas

- `User git` is **always `git`** for GitHub — it never changes, regardless of account.
- More specific `Host` blocks should come **before** general ones in the config.
- `ssh-agent` (key caching) is separate from `~/.ssh/config` — the config is read by the SSH client itself.

---

## Layer 2 — Git Config: Commit Authorship

### Why it works this way

Git embeds `user.name` and `user.email` into every commit at the time you commit. The global `~/.gitconfig` sets defaults. Git supports **conditional includes** (`includeIf`) that automatically swap in a different config file based on which directory you're working in — so repos under `~/Documents/working/` can silently use your office identity while everything else uses personal.

---

## Step-by-Step Setup

### Step 1 — Generate SSH Keys

```bash
# Personal GitHub
ssh-keygen -t ed25519 -C "your-personal@email.com" -f ~/.ssh/id_ed25519_personal

# Office GitHub
ssh-keygen -t ed25519 -C "your-office@email.com" -f ~/.ssh/id_ed25519_office
```

Add the public keys to each GitHub account:
```bash
cat ~/.ssh/id_ed25519_personal.pub   # → paste into personal GitHub account
cat ~/.ssh/id_ed25519_office.pub     # → paste into office GitHub account
```
GitHub → Settings → SSH and GPG keys → New SSH key → paste.

---

### Step 2 — Configure `~/.ssh/config`

```
Host github-personal
    HostName     github.com
    User         git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github-office
    HostName     github.com
    User         git
    IdentityFile ~/.ssh/id_ed25519_office

# Future: Bitbucket (same pattern, different HostName)
Host bitbucket-work
    HostName     bitbucket.org
    User         git
    IdentityFile ~/.ssh/id_ed25519_bitbucket
```

Test that each alias resolves to the right account:
```bash
ssh -T github-personal   # → Hi <personal-username>!
ssh -T github-office     # → Hi <office-username>!
```

---

### Step 3 — Configure Git Commit Identity

**`~/.gitconfig`** (main file — personal is the default):
```ini
[user]
    name  = Deepak (Personal)
    email = deepak@personal.com

[includeIf "gitdir:~/Documents/working/"]
    path = ~/.gitconfig-office
```

**`~/.gitconfig-office`** (office overrides):
```ini
[user]
    name  = Deepak
    email = deepak@company.com
```

Any repo under `~/Documents/working/` automatically gets the office identity. Add more `includeIf` blocks for additional directories as needed.

---

### Step 4 — Fix Remote URLs

For **existing repos**, update the remote to use the alias:
```bash
# Office repo
git remote set-url origin git@github-office:org/repo.git

# Personal repo
git remote set-url origin git@github-personal:myuser/repo.git
```

For **new clones**, use the alias in the URL directly:
```bash
git clone git@github-office:org/repo.git
git clone git@github-personal:myuser/repo.git
```

---

## Verify Everything Works

```bash
# Confirm SSH is routing to the right account
ssh -T github-personal
ssh -T github-office

# Confirm commit identity inside any repo
git config user.name
git config user.email

# Confirm remote URL uses the alias
git remote -v
```

---

## Quick Reference

| File | Controls |
|---|---|
| `~/.ssh/config` | Which SSH key (= which GitHub account) to use per host alias |
| `~/.ssh/id_ed25519_*` | The private keys themselves |
| `~/.gitconfig` | Default commit name/email (personal) |
| `~/.gitconfig-office` | Office commit name/email (auto-applied by directory) |
| `.git/config` | Per-repo override (use when a repo sits outside normal directories) |
