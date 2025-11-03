You’re hitting the classic **SSH key / access** issue. Fix it fast with one of these paths.

# 0) Make sure the URL & access are right

* Repo is **private** → you must be a member of **The-DARE-Labs** and have access to that repo.
* Double-check the exact name: you tried `statsbridge-experiment-1` (from the screenshot). That looks right, but confirm spelling/case.
* If org SSO is enabled, you must **accept the org invite** and **authorize your SSH key for the org**.

---

# Option A — Use SSH (preferred once set up)

### 1) Do you already have an SSH key?

```bash
ls -al ~/.ssh
# look for id_ed25519 or id_rsa (and their .pub)
```

### 2) If not, create one

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# press enter for default path; set a passphrase if you want
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### 3) Add the **public** key to GitHub

```bash
cat ~/.ssh/id_ed25519.pub
# copy this output → GitHub → Settings → SSH and GPG keys → New SSH key
# If the org uses SSO, click “Enable SSO” for The-DARE-Labs on that key.
```

### 4) Test SSH

```bash
ssh -T git@github.com
# You should see: "Hi <username>! You've successfully authenticated..."
```

> Behind a firewall that blocks port 22? Try SSH over 443:

```bash
ssh -T -p 443 git@ssh.github.com
```

(or configure `~/.ssh/config` with `Host github.com Hostname ssh.github.com Port 443`)

### 5) Clone

```bash
cd ~/Convonomics
git clone git@github.com:The-DARE-Labs/statsbridge-experiment-1.git
```

---

# Option B — Use HTTPS (quickest, no SSH)

Use GitHub username + **PAT** (token) instead of a password.

```bash
cd ~/Convonomics
git clone https://github.com/The-DARE-Labs/statsbridge-experiment-1.git
# When prompted for password, paste a Personal Access Token with "repo" scope.
```

---

# Option C — Use GitHub CLI (nice DX)

```bash
# Install gh if needed, then:
gh auth login
# Choose: GitHub.com → HTTPS → Authenticate via web browser
gh repo clone The-DARE-Labs/statsbridge-experiment-1 ~/Convonomics/statsbridge-experiment-1
```

---

## If it still fails

* Confirm you’ve been **added to the org** (and accepted the invite).
* Ask an admin to verify repo permissions for your user.
* If using SSH: ensure your key is **authorized for the org (SSO)** in GitHub.
* Verify the repo actually exists (name/case) in the org.

Want me to generate a tiny bash script that checks SSH setup and prints next steps automatically?
