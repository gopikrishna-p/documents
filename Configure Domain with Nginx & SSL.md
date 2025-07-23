# Complete Guide to Configure Domain with Nginx & SSL (Let's Encrypt)

## 🎯 Objective
To map a domain (e.g., `admin.adorablealways.com`) to a server's IP address and configure it to:
- Use Nginx as a reverse proxy
- Serve your project running on a specific port (e.g., 8088)
- Enable HTTPS using Let's Encrypt (via Certbot)
- Ensure auto-renewal of the SSL certificate

---

## 1️⃣ DNS Configuration
Before you begin, make sure:
✅ Your domain (e.g., `admin.adorablealways.com`) has an A record pointing to your server IP (e.g., `34.28.138.99`).
✅ DNS propagation may take up to 48 hours.

To check DNS status:
```bash
dig admin.adorablealways.com
```
Look for the A record pointing to your server's IP.

---

## 2️⃣ Install Nginx (if not already installed)
```bash
sudo apt update
sudo apt install nginx
```

---

## 3️⃣ Configure Nginx
Create a config file:
```bash
sudo nano /etc/nginx/sites-available/admin.adorablealways.com
```

Add this basic reverse proxy configuration:
```nginx
server {
    listen 80;
    server_name hr.deepgrid.in;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name bi.deepgrid.in;

    # Redirect all HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name bi.deepgrid.in;

    ssl_certificate /etc/letsencrypt/live/bi.deepgrid.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bi.deepgrid.in/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8010;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 4️⃣ Enable the Nginx Configuration
```bash
sudo ln -s /etc/nginx/sites-available/admin.adorablealways.com /etc/nginx/sites-enabled/
```

---

## 5️⃣ Test and Reload Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

You should now be able to access the app via `http://admin.adorablealways.com`.

---

## 6️⃣ Install SSL Certificate using Certbot
Install Certbot:
```bash
sudo apt install certbot python3-certbot-nginx
```

Run Certbot for SSL setup:
```bash
sudo certbot --nginx -d admin.adorablealways.com
```

🛡 Certbot will:
- Request a certificate from Let's Encrypt
- Automatically update your Nginx config for HTTPS
- Redirect HTTP to HTTPS if selected

---

## 7️⃣ Verify HTTPS Access
Visit:  
🔗 `https://admin.adorablealways.com`  
✔️ It should show a secure lock symbol and serve your application.

---

## 8️⃣ Test SSL Auto-Renewal
```bash
sudo certbot renew --dry-run
```
Certbot will simulate the renewal and verify that no issues exist with automatic renewal.

---

## 9️⃣ Allow HTTP/HTTPS Through Firewall (if applicable)
If using UFW:
```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw status
```

On GCP or AWS, open ports 80 and 443 in your instance's firewall/security group settings.

---

## 🔍 Troubleshooting Tips

| Issue | Command to Check |
|-------|------------------|
| DNS not resolving | `dig admin.adorablealways.com` |
| Nginx error logs | `sudo tail -f /var/log/nginx/error.log` |
| App not reachable | Check if it's running on port 8088 |
| SSL not working | `openssl s_client -connect admin.adorablealways.com:443` |

---

## 🧠 Additional Notes
- Nginx reverse proxy hides port 8088, exposing clean HTTPS URLs.
- Let's Encrypt certificates expire in 90 days, but Certbot will handle renewal via systemd or cron.
- Custom ports (e.g., 8088 directly over HTTPS) require manual SSL binding and are not recommended.

---

## ✅ Final Output
Your application running on port 8088 is now securely available at:  
🔐 `https://admin.adorablealways.com`
```