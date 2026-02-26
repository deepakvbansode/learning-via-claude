# SSH Config — Managing Multiple Accounts

> **Goal:** Set up `~/.ssh/config` on my local machine to manage my personal GitHub, office GitHub, and a Bitbucket account.

---

## What I Know

- **SSH client reads `~/.ssh/config`** on every connection to decide which key, user, and hostname to use.
- A **`Host` block** is an alias you invent; `HostName` is the real server address — they can differ.
- For **two GitHub accounts**, create two `Host` aliases both pointing to `HostName github.com` but with different `IdentityFile` entries — GitHub identifies you by which public key you registered.
- **Git remote URLs use the `Host` alias** instead of `github.com` — changing just the hostname portion routes SSH to the right config block.
- **`ssh-agent`** holds keys in memory; `~/.ssh/config` is separate — read by the SSH client itself.

---

## Mental Model

> `~/.ssh/config` is like a **kubeconfig for SSH** — named contexts, each with its own credentials, all pointing to potentially the same server. Git remotes reference these context names directly in the URL. GitHub cares only about **which public key you present**, not your username.

---

## Gotchas to Remember

- `User git` is **always `git` for GitHub** — it never changes regardless of account.
- Always **test with `ssh -T`** before touching repos.
- For new clones, **use the alias in the clone URL directly** — not `github.com`.
- For Bitbucket, **same pattern** — just a new `Host` block with `HostName bitbucket.org`.
- **More specific `Host` blocks should come before general ones** in the config file.

---

## Quick Reference

### `~/.ssh/config`

```
Host github-personal
    HostName     github.com
    User         git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github-office
    HostName     github.com
    User         git
    IdentityFile ~/.ssh/id_ed25519_office

Host bitbucket-work
    HostName     bitbucket.org
    User         git
    IdentityFile ~/.ssh/id_ed25519_bitbucket
```

---

### Key Commands

```bash
# Test a connection
ssh -T github-personal

# Update existing repo to use office account
git remote set-url origin git@github-office:org/repo.git

# Clone with specific account
git clone git@github-personal:myuser/repo.git

# See current remote URL
git remote -v

# Add key to ssh-agent (if using passphrase)
ssh-add ~/.ssh/id_ed25519_personal
```
