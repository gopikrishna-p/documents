In Git, you can **remove an existing remote origin** and **add a new one** with just a couple of commands.

### 🔹 Step 1: Check current remotes

```bash
git remote -v
```

This shows the current remote(s) configured for your repo. Usually, you'll see something like:

```
origin  git@github.com:old-user/old-repo.git (fetch)
origin  git@github.com:old-user/old-repo.git (push)
```

---

### 🔹 Step 2: Remove existing origin

```bash
git remote remove origin
```

(or the older form)

```bash
git remote rm origin
```

---

### 🔹 Step 3: Add new origin

```bash
git remote add origin git@github.com:new-user/new-repo.git
```

or (if using HTTPS)

```bash
git remote add origin https://github.com/new-user/new-repo.git
```

---

### 🔹 Step 4: Push to the new origin

If you already have commits in your local repo, you’ll likely want to push them:

```bash
git push -u origin main
```

(or `master`, depending on your default branch).

---

✅ That’s it! Now your repo is connected to the new origin.

Do you want me to also show you a **shortcut command** that replaces the old origin directly without removing it first?
