# 🧭 Frappe/ERPNext Installation Guide for Ubuntu 22.04 LTS

## Overview

This comprehensive guide will walk you through the installation of **Frappe Framework** and **ERPNext version 13** on **Ubuntu 22.04 LTS**.
The installation includes all necessary dependencies and configurations for a fully functional development environment.

---

## 💻 System Requirements

### Hardware Requirements

* **RAM:** Minimum 4GB (8GB recommended)
* **Storage:** Minimum 20GB free space
* **CPU:** 2+ cores recommended

### Software Requirements

| Tool         | Version                         | Purpose                          |
| ------------ | ------------------------------- | -------------------------------- |
| Python       | 3.6+ (Ubuntu 22.04 uses 3.10.x) | Backend language                 |
| Node.js      | 14.15.0                         | Frontend build tools             |
| Redis        | 5+                              | Caching, real-time updates       |
| MariaDB      | 10.3.x                          | Database                         |
| Yarn         | 1.12+                           | JS dependency manager            |
| Pip          | 20+                             | Python package manager           |
| wkhtmltopdf  | 0.12.5 (qt-patched version)     | PDF generation                   |
| NGINX + Cron | Optional                        | Reverse proxy and scheduled jobs |

---

## ⚙️ Installation Steps

### **Step 1: Update System and Install Git**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git -y
```

---

### **Step 2: Install Python Development Tools**

```bash
sudo apt install pkg-config python3-dev python3-pip python3-setuptools -y
```

---

### **Step 3: Install Virtual Environment**

```bash
sudo apt install virtualenv -y
```

---

### **Step 4: Verify Python Version and Install venv**

```bash
python3 -V
```

If Python 3.10 is installed, run:

```bash
sudo apt install python3.10-venv -y
```

---

### **Step 5: Install and Configure MariaDB**

#### Install MariaDB

```bash
sudo apt install software-properties-common mariadb-server -y
```

#### Secure MariaDB Installation

```bash
sudo mysql_secure_installation
```

**Follow the prompts:**

* Enter current password for root: *(Leave blank or enter SSH root password)*
* Switch to unix_socket authentication [Y/n]: Y
* Change the root password? [Y/n]: Y *(set a new password)*
* Remove anonymous users? [Y/n]: Y
* Disallow root login remotely? [Y/n]: N *(set to N if remote access is needed)*
* Remove test database and access to it? [Y/n]: Y
* Reload privilege tables now? [Y/n]: Y

#### Configure MariaDB for Frappe

Edit the MariaDB configuration file:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add the following configuration:

```ini
[server]
user = mysql
pid-file = /run/mysqld/mysqld.pid
socket = /run/mysqld/mysqld.sock
basedir = /usr
datadir = /var/lib/mysql
tmpdir = /tmp
lc-messages-dir = /usr/share/mysql
bind-address = 127.0.0.1
query_cache_size = 16M
log_error = /var/log/mysql/error.log

[mysqld]
innodb-file-format = barracuda
innodb-file-per-table = 1
innodb-large-prefix = 1
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

Restart MariaDB:

```bash
sudo service mysql restart
```

---

### **Step 6: Install Redis**

```bash
sudo apt install redis-server -y
```

Verify Redis is running:

```bash
redis-cli ping
```

Expected output: `PONG`

---

### **Step 7: Install Node.js 14.x using NVM**

#### Install NVM (Node Version Manager)

```bash
sudo apt install curl -y
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
```

Verify installation:

```bash
nvm --version
```

If `nvm` command not found:

```bash
source ~/.nvm/nvm.sh
```

#### Install Node.js 18.x

```bash
nvm install 18
nvm use 18
nvm alias default 18
node -v
npm -v
```

Alternative Node.js installation (via NodeSource):

```bash
sudo apt update
sudo apt install curl -y
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

---

### **Step 8: Install Yarn**

```bash
sudo npm install -g yarn
yarn --version
```

---

### **Step 9: Install wkhtmltopdf**

```bash
sudo apt install xvfb libfontconfig wkhtmltopdf -y
```

> **Note:** For production, use `wkhtmltopdf 0.12.5` with qt patch.

---

### **Step 10: Install MySQL/MariaDB Development Libraries**

```bash
sudo apt install libmysqlclient-dev default-libmysqlclient-dev
sudo apt update
sudo apt install libmariadb-dev libmariadb-dev-compat
```

---

### **Step 11: Install Frappe Bench**

```bash
sudo -H pip3 install frappe-bench
bench --version
```

---

### **Step 12: Initialize Frappe Bench**

```bash
bench init frappe-bench --frappe-branch version-14
```

> Note: frappe and erpnext versions must match.
> Example:

```bash
bench init frappe-bench --frappe-branch version-13
cd frappe-bench
```

---

### **Step 13: Create a New Site**

```bash
bench new-site your-site-name.local
```

If you face MySQL access error:

```bash
sudo mysql
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('root');
FLUSH PRIVILEGES;
```

---

### **Step 14: Set Default Site**

```bash
bench use your-site-name.local
npm install @tiptap/core@^2.0.0 --save
```

---

### **Step 15: Install ERPNext and HRMS**

```bash
bench get-app erpnext --branch version-14
bench get-app hrms
bench --site your-site-name.local install-app erpnext
bench --site your-site-name.local install-app hrms
```

---

### **Step 16: Start Development Server**

```bash
bench start
```

ERPNext is available at:
👉 [http://localhost:8000](http://localhost:8000)

---

## 🧑‍💻 Post-Installation Configuration

### Create a User (Optional)

```bash
sudo adduser frappe
sudo usermod -aG sudo frappe
su - frappe
```

### Production Setup (Optional)

```bash
sudo bench setup production frappe
```

This will:

* Configure NGINX as reverse proxy
* Setup systemd services
* Configure SSL (optional)
* Enable auto startup

---

## ⚠️ Troubleshooting

### Common Issues and Fixes

1. **Permission Errors**

   ```bash
   sudo chown -R $USER:$USER /path/to/frappe-bench
   ```
2. **Port Already in Use**

   ```bash
   bench set-config -g default_site your-site-name.local
   bench set-config -g serve_default_site 1
   ```
3. **MariaDB Connection Issues**

   ```bash
   sudo systemctl status mariadb
   ```
4. **Redis Connection Issues**

   ```bash
   sudo systemctl status redis
   redis-cli ping
   ```

---

## 🔧 Useful Commands

### Bench Commands

```bash
bench start
bench stop
bench restart
bench update
bench --site your-site-name.local backup
bench --site your-site-name.local restore /path/to/backup
bench new-app app-name
bench --site your-site-name.local install-app app-name
bench --site all list-apps
bench --site your-site-name.local set-maintenance-mode on
bench --site your-site-name.local set-maintenance-mode off
```

### Database Commands

```bash
bench --site your-site-name.local mariadb
bench --site your-site-name.local db-console
```

---

## 🔐 Security Considerations (Production)

1. Change default passwords
2. Enable firewall

   ```bash
   sudo ufw enable
   sudo ufw allow ssh
   sudo ufw allow http
   sudo ufw allow https
   ```
3. Regular backups

   ```bash
   bench --site your-site-name.local backup --with-files
   ```
4. SSL setup

   ```bash
   sudo bench setup lets-encrypt your-site-name.local
   ```

---

## 🔄 Maintenance

### Regular Updates

```bash
pip3 install --upgrade frappe-bench
bench update
bench update --app erpnext
```

### Log Files

* **Frappe logs:** `~/frappe-bench/logs/`
* **MariaDB logs:** `/var/log/mysql/error.log`
* **NGINX logs:** `/var/log/nginx/`

---

## ✅ Conclusion

You now have a **fully functional Frappe/ERPNext** installation on Ubuntu 22.04 LTS.
The system is ready for **development or production use**.

For more details:
👉 [Official Frappe Documentation](https://frappeframework.com/docs)

---

## 🆘 Support

If you face issues:

1. Check troubleshooting section
2. Review log files
3. Refer official docs
4. Visit [Frappe Community Forum](https://discuss.frappe.io/)

---

### 📄 Document Info

* **Version:** 1.0
* **Last Updated:** June 2025
* **Compatible with:** Ubuntu 22.04 LTS, Frappe v13, ERPNext v13

---

## 🔁 Reinstall Site (Recommended for New Setups)

```bash
bench drop-site gopi.local --root-password your_mysql_root_password
bench new-site gopi.local
bench --site gopi.local install-app erpnext
bench --site gopi.local backup
bench --site gopi.local install-app hrms
```

---

## 🧹 Site Management Commands

```bash
bench drop-site deepgrid.local
bench drop-site deepgrid.local --force
bench --site deepgrid.local reinstall
bench --site deepgrid.local uninstall-app hrms
bench --site deepgrid.local uninstall-app hrms --force
bench --site deepgrid.local uninstall-app hrms --delete
bench remove-app hrms
rm -rf apps/hrms
```

---

## 🏗️ Example New Setup

```bash
# Step 1: Create a new bench
bench init frappe-bench --frappe-branch version-14
cd frappe-bench

# Step 2: Create a site
bench new-site yoursite.local

# Step 3: Get compatible ERPNext and HRMS apps
bench get-app --branch version-14 erpnext https://github.com/frappe/erpnext
bench get-app --branch version-14 hrms https://github.com/frappe/hrms

# Step 4: Install apps on the site
bench --site yoursite.local install-app erpnext
bench --site yoursite.local install-app hrms

# Step 5: Start the server
bench start
```

---

## 🧰 Redis Spawn Error Debug (BACKOFF)

If Redis fails with “spawn error → BACKOFF,” follow these steps:

### ✅ Step-by-Step Debug for Redis

#### Step 1 — Check Port Availability

```bash
sudo lsof -i :11002 -i :13002 -i :13001
sudo kill -9 <PID>
```

#### Step 2 — Clean up stale PID files

```bash
rm -f ~/frappe-bench-v15/config/redis_*.pid
rm -f /tmp/redis-*.pid
```

#### Step 3 — Test Redis manually

```bash
redis-server config/redis_queue.conf
```

If it runs fine, stop it with **Ctrl+C**.

#### Step 4 — Check Redis Logs

```bash
tail -n 30 logs/redis-queue.log
tail -n 30 logs/redis-cache.log
```

#### Step 5 — Restart Redis via Supervisor

```bash
sudo supervisorctl start "frappe-bench-v15-redis:*"
sudo supervisorctl status | grep redis
```

#### Step 6 — Regenerate Redis Config

```bash
bench setup redis
bench setup procfile
```

---

✅ **Once Redis runs correctly**, all other services (SocketIO, APIs, etc.) will start smoothly.

---

Would you like me to convert this Markdown into a **well-formatted PDF guide** (with headings, code blocks, and page styling) for documentation or sharing purposes?
