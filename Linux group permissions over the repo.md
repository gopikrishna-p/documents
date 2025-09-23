Got it 👍 You want both **`aravindh`** and **`gcp_user`** to have access to the repo without running into these “Permission denied” issues.

The clean way is to use a **Linux group** that both users belong to, and give that group permissions over the repo.

---

## 🔧 Steps to Fix Permissions for Both Users

### 1. Create a Shared Group

```bash
sudo groupadd projectgrp
```

### 2. Add Both Users to the Group

```bash
sudo usermod -aG projectgrp aravindh
sudo usermod -aG projectgrp gcp_user
```

(Log out & log back in, or run `newgrp projectgrp` to apply the new group without reboot.)

---

### 3. Change Repo Ownership to Group

From inside `/home/aravind/Desktop/INVESTMENT-PLATFORM`:

```bash
sudo chown -R :projectgrp .
```

---

### 4. Set Group Permissions (Read/Write/Execute)

```bash
sudo chmod -R 770 .
```

This gives **read/write/execute** to owner and group, no access for others.

---

### 5. Make Group Sticky (so new files also inherit the group)

```bash
sudo find . -type d -exec chmod g+s {} \;
```

Now any new file created in the repo will automatically belong to `projectgrp`.

---

✅ Now both `aravindh` and `gcp_user` can `git pull`, `checkout`, and modify files without permission issues.

---

⚠️ Question for you: do you want **all new files by default** to be writable by both users (like a true shared dev folder), or do you prefer only the current repo to be fixed?
