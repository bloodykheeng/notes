# GitHub SSH Authentication on Linux

Authenticate with GitHub using an SSH key pair instead of a password. You generate a private/public key pair, upload the public key to GitHub, and use the private key to prove your identity.

Reference: [Create & Add SSH Keys To GitHub For Authentication](https://youtu.be/yVP3sYgd0bY?si=U53_7xk5goJl2xW4)

---

## Step 1 — Generate an SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

| Flag | Meaning |
|---|---|
| `-t ed25519` | **Type** — the encryption algorithm to use. `ed25519` is modern, fast, and secure. Other options are `rsa`, `ecdsa`, but `ed25519` is the recommended choice today. |
| `-C "your@email.com"` | **Comment** — a label embedded in the public key so you can identify it. Can be anything: your email, server name, machine name, or purpose. Has no effect on how authentication works. |

When prompted for a file path, you can press `Enter` to use the default or specify a custom path:

```
Enter file in which to save the key: /home/youruser/.ssh/github_key
```

**Default path on Ubuntu VPS (logged in as root)** — if you just press `Enter`, keys are saved at:

```
/root/.ssh/id_ed25519        ← private key
/root/.ssh/id_ed25519.pub    ← public key
```

If you specify a custom path you get:

```
/home/youruser/.ssh/github_key        ← private key (keep this secret)
/home/youruser/.ssh/github_key.pub    ← public key (upload this to GitHub)
```

### Passphrase prompt

```
Enter passphrase (empty for no passphrase):
```

- **Leave empty** (press `Enter`) — recommended for servers doing automated deployments, CI/CD, or scheduled git pulls. No password means scripts can run without human input.
- **Set a passphrase** — encrypts the private key file on disk. If someone steals the file they still can't use it without the passphrase. **But every time you do `git pull`, `git push`, or `ssh-add`, you will be asked to enter it.** So if you set one, make it something you will always remember.

> On a Ubuntu VPS doing deployments → leave it empty. On a personal laptop → set a passphrase you won't forget.

### Remove a Passphrase You Already Set

If you set a passphrase and it keeps prompting you on every `git pull` / `git push`, remove it:

```bash
ssh-keygen -p -f /root/.ssh/id_ed25519
```

It will ask:
```
Enter old passphrase:        ← type your current passphrase
Enter new passphrase:        ← leave empty, just press Enter
Enter same passphrase again: ← press Enter again
```

Your passphrase is now removed. `git pull` will no longer prompt for it.

---

## Step 2 — Copy the Public Key

**If you used the default path** (just pressed `Enter` during keygen):

```bash
cat /root/.ssh/id_ed25519.pub      # as root
cat ~/.ssh/id_ed25519.pub          # as a normal user
```

**If you used a custom path:**

```bash
cat /home/youruser/.ssh/github_key.pub
```

Copy the entire output — it starts with `ssh-ed25519 ...`

---

## Step 3 — Add the Public Key to GitHub

1. Go to **GitHub → Settings → SSH and GPG keys**
2. Click **New SSH key**
3. Set **Title** to something like `My Server Key`
4. Set **Key type** to `Authentication Key`
5. Paste the contents of `github_key.pub` into the **Key** field
6. Click **Add SSH key**

GitHub now knows: *if someone presents the private key matching this public key, let them in.*

---

## Step 4 — Add Your Private Key to the SSH Agent (Custom Path Only)

> Skip this step if you used the default path (`/root/.ssh/id_ed25519`). SSH picks it up automatically.
> Only needed if you saved your key to a custom path like `/root/.ssh/github_key`.

```bash
# Start the agent
eval "$(ssh-agent -s)"

# Add your custom key
ssh-add /home/youruser/.ssh/github_key
```

### Confirm the key was added

```bash
ssh-add -l
```

You should see a line with your key fingerprint and email.

---

## Step 5 — Test the Connection

> ⚠️ **Test against the host where you actually added the key.**
> GitHub and GitLab are separate services. Adding your key to **GitLab**
> does nothing for **GitHub**, and vice versa. A common mistake is adding
> the key on GitLab but running `ssh -T git@github.com`, which gives:
>
> ```
> git@github.com: Permission denied (publickey).
> ```
>
> The key is fine — you just pointed it at the wrong host. Use the matching command:

```bash
ssh -T git@github.com     # if you added the key to GitHub
ssh -T git@gitlab.com     # if you added the key to GitLab
```

Expected output:

```
Hi youruser! You've successfully authenticated, but GitHub does not provide shell access.
# or, on GitLab:
Welcome to GitLab, @youruser!
```

---

## Step 6 — Get the SSH Link from GitHub / GitLab

Do **not** copy the HTTPS link — you must use the SSH link, otherwise git will still ask for a username and password even after all this setup.

**GitHub:**
Go to your repo → click **Code** → select the **SSH** tab → copy the link.
```
git@github.com:youruser/your-repo.git
```

**GitLab:**
Go to your repo → click **Clone** → copy the link under **Clone with SSH**.
```
git@gitlab.com:youruser/your-repo.git
```

Then clone:

```bash
git clone git@github.com:youruser/your-repo.git
# or
git clone git@gitlab.com:youruser/your-repo.git
```

---

## Already Cloned with HTTPS? Switch to SSH

If you already cloned with HTTPS, git will ignore your SSH key and keep asking for a username and password on every `git pull` or `git push`.

Check your current remote URL:

```bash
git remote -v
```

If it shows `https://...` you need to switch it:

```bash
# For GitHub
git remote set-url origin git@github.com:youruser/your-repo.git

# For GitLab
git remote set-url origin git@gitlab.com:youruser/your-repo.git
```

Confirm the change:

```bash
git remote -v
# origin  git@gitlab.com:youruser/your-repo.git (fetch)
# origin  git@gitlab.com:youruser/your-repo.git (push)
```

Now `git pull` and `git push` will use your SSH key with no password prompt.

---

## Persisting the Key Across Reboots

> ⚠️ **Only needed if you used a CUSTOM key path** (e.g. `/root/.ssh/github_key`).
>
> If you used the **default path** (`/root/.ssh/id_ed25519` — i.e. you just
> pressed `Enter` during keygen), you can **skip this entire section**. SSH
> automatically looks for default key names (`id_ed25519`, `id_rsa`, …) in
> `~/.ssh/` on every connection, including after reboots — there is nothing to
> persist. Just run `ssh -T git@github.com` (or `git@gitlab.com`) and it works.

The SSH agent resets on reboot. To auto-load your **custom-named** key, add this to `~/.bashrc` or `~/.zshrc`:

```bash
eval "$(ssh-agent -s)" > /dev/null 2>&1
ssh-add /home/youruser/.ssh/github_key 2>/dev/null
```

Or use `~/.ssh/config` to always use your custom key for GitHub:

```bash
nano ~/.ssh/config
```

Add:

```
# GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile /root/.ssh/id_ed25519

# GitLab
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile /root/.ssh/id_ed25519
```

With this config in place you never need `ssh-add` manually — Git picks up the key automatically.

---

## Quick Reference

| Task | Command |
|---|---|
| Generate key pair | `ssh-keygen -t ed25519 -C "you@email.com"` |
| View public key | `cat ~/.ssh/github_key.pub` |
| Start SSH agent | `eval "$(ssh-agent -s)"` |
| Add key to agent | `ssh-add ~/.ssh/github_key` |
| List loaded keys | `ssh-add -l` |
| Test GitHub connection | `ssh -T git@github.com` |
| Clone via SSH | `git clone git@github.com:user/repo.git` |
