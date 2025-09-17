  I see what’s happening 👀

Even after you killed the redis processes (`695616`, `695617`), **they restarted immediately with new PIDs** (`697532`, `697533`). That means **supervisord is managing redis** and respawns them as soon as they die.

That’s why you keep seeing redis on **12001** and **13001**.

---

### ✅ Fix: Stop all supervisor-managed processes first

Run:

```bash
sudo supervisorctl stop all
```

Check again:

```bash
sudo lsof -i :12001
sudo lsof -i :13001
```

Both should be empty now.

---

### 🚀 Restart bench cleanly

Once ports are free:

```bash
cd ~/frappe-bench-v15
bench start
```

(or, if you want everything managed by supervisor again)

```bash
bench restart
```

---

⚠️ Also, don’t use `ping 34.28.138.99:8001` — ping doesn’t take ports. Instead test your site with:

```bash
curl -I http://127.0.0.1:8001
```

or from outside:

```bash
curl -I https://hr.deepgrid.in
```

```bash
worker.1      | /bin/sh: 1: cannot create logs/worker.error.log: Permission denied
sudo chown -R gcp_user:gcp_user ~/frappe-bench-v15/logs
sudo chown -R gcp_user:gcp_user ~/frappe-bench-v15
```
---

### ✅ Database details
```bash
mysql -u root -p
root
show databases;
describe `tabProject Employee Assignment`;
```
