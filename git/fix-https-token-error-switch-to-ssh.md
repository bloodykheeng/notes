# Fix `fatal: unable to access` HTTPS Token Error → Switch to SSH (Windows)

## The Problem

You cloned/pushed over **HTTPS** and Windows cached a Personal Access Token (PAT)
that has the **wrong scopes** (e.g. read-only, or missing `write_repository`).
Git keeps reusing that bad cached token, so every push fails with:

```
fatal: unable to access 'https://gitlab.com/dbuwembo/niceweb.git/': ...
```

Re-entering the token doesn't help because Windows Credential Manager keeps
handing back the **old** one. The clean fix is to **drop the cached credential**
and **switch the repo to SSH** so you stop dealing with tokens entirely.

> These commands are for **Windows PowerShell** (note `Get-Content`, `clip`).

---

## Step 1 — Remove the Bad Cached Credential

This tells Git's credential helper to forget the stored token for GitLab:

```powershell
"url=https://gitlab.com" | git credential reject
```

> For GitHub, swap the host: `"url=https://github.com" | git credential reject`.
> You can also clear it via **Control Panel → Credential Manager → Windows
> Credentials** and delete the `git:https://gitlab.com` entry.

---

## Step 2 — Switch the Repository Remote to SSH

Point the remote at the **SSH** URL instead of HTTPS:

```powershell
git remote set-url origin git@gitlab.com:dbuwembo/niceweb.git
```

Confirm it changed:

```powershell
git remote -v
# origin  git@gitlab.com:dbuwembo/niceweb.git (fetch)
# origin  git@gitlab.com:dbuwembo/niceweb.git (push)
```

> The SSH URL format is `git@HOST:USER/REPO.git` — note the **colon** after the
> host (not a slash like HTTPS).

---

## Step 3 — Check If You Already Have an SSH Key

```powershell
Get-Content ~/.ssh/id_ed25519.pub
```

- **Key prints out** (starts with `ssh-ed25519 ...`) → you already have one,
  skip Step 4.
- **Error / file not found** → generate one in Step 4.

---

## Step 4 — Generate an SSH Key (only if you don't have one)

```powershell
ssh-keygen -t ed25519 -C "bloodykheeng hp pavilion laptop"
```

Press `Enter` to accept the default path. Leave the passphrase empty (or set one
you'll remember). The `-C` comment is just a label so you can recognize the key
later — use something identifying the machine, e.g. the laptop name.

---

## Step 5 — Copy the Public Key

Copy it straight to the clipboard:

```powershell
Get-Content ~/.ssh/id_ed25519.pub | clip
```

Or print it and copy manually:

```powershell
Get-Content C:\Users\bk\.ssh\id_ed25519.pub
```

> Copy the **`.pub`** file only — that's the **public** key. Never share or paste
> the private key (`id_ed25519`, no extension).

---

## Step 6 — Add the Key to GitLab

1. GitLab → **profile picture → Edit profile → SSH Keys** (or
   `gitlab.com/-/user_settings/ssh_keys`).
2. Click **Add new key**.
3. Paste the public key.
4. Set **Usage type** to `Authentication & Signing` and an expiry if you want.
5. Click **Add key**.

> GitHub equivalent: **Settings → SSH and GPG keys → New SSH key**.

---

## Step 7 — Test the SSH Connection

```powershell
ssh -T git@gitlab.com
```

Expected:

```
Welcome to GitLab, @dbuwembo!
```

> ⚠️ Test against the **same host** where you added the key. Key on GitLab →
> test `git@gitlab.com`; key on GitHub → test `git@github.com`. Mixing them up
> gives `Permission denied (publickey)`.

Now `git push` / `git pull` work over SSH with no token prompts.

---

## Reusing This Laptop on Other Projects

Once the key exists on this machine **and** is registered on GitLab, you never
repeat Steps 3–7. For any other repo still on HTTPS, you only switch its remote:

```powershell
git remote set-url origin git@gitlab.com:dbuwembo/niceapi.git
```

That single command moves `niceapi` (and any other repo) onto SSH. The same key
authenticates all of them.

---

## Quick Reference

| Task | Command (PowerShell) |
|---|---|
| Drop bad cached token | `"url=https://gitlab.com" \| git credential reject` |
| Switch remote to SSH | `git remote set-url origin git@gitlab.com:USER/REPO.git` |
| Check current remote | `git remote -v` |
| Check for existing key | `Get-Content ~/.ssh/id_ed25519.pub` |
| Generate a key | `ssh-keygen -t ed25519 -C "machine label"` |
| Copy public key to clipboard | `Get-Content ~/.ssh/id_ed25519.pub \| clip` |
| Test connection | `ssh -T git@gitlab.com` |
