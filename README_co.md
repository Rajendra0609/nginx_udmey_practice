# Nginx Reverse Proxy Setup — Production Guide

> Proxies `https://simba` → `http://10.244.24.67:8080`  
> TLS termination at Nginx | HTTP/2 | Rate limiting | Security hardened

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Install Nginx](#install-nginx)
3. [SSL Certificate Setup](#ssl-certificate-setup)
4. [Deploy nginx.conf](#deploy-nginxconf)
5. [DNS Setup — simba](#dns-setup--simba)
6. [Access From Other Machines](#access-from-other-machines)
7. [Verify & Test](#verify--test)
8. [Managing Nginx](#managing-nginx)
9. [Troubleshooting](#troubleshooting)
10. [Production Checklist](#production-checklist)

---

## Prerequisites

- Ubuntu 20.04 / 22.04 / 24.04 (or any Debian-based Linux)
- Your app already running at `http://10.244.24.67:8080`
- `sudo` / root access on the Nginx server
- Nginx server IP: `10.244.24.67`

---

## Install Nginx

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y

# Verify installation
nginx -v

# Enable nginx to start on boot
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## SSL Certificate Setup

Choose **one** of the following options.

### Option A — Self-Signed Certificate (LAN / Internal use)

Use this when you don't have a public domain name.

```bash
# Create the ssl directory
sudo mkdir -p /etc/nginx/ssl

# Generate a 4096-bit self-signed cert valid for 10 years
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
  -keyout /etc/nginx/ssl/simba.key \
  -out    /etc/nginx/ssl/simba.crt \
  -subj "/CN=simba"

# Lock down permissions
sudo chmod 600 /etc/nginx/ssl/simba.key
sudo chmod 644 /etc/nginx/ssl/simba.crt
```

> **Note:** Browsers will show a security warning for self-signed certs. Click **Advanced → Proceed** for internal use. This is normal and safe on a trusted LAN.

---

### Option B — Let's Encrypt (Public Domain)

Use this when you have a real public domain (e.g. `simba.yourdomain.com`).

```bash
sudo apt install certbot python3-certbot-nginx -y

# Replace with your actual domain
sudo certbot --nginx -d simba.yourdomain.com

# Auto-renewal (already set up by certbot, verify it)
sudo systemctl status certbot.timer
```

Then update `nginx.conf` — replace the certificate paths:
```nginx
ssl_certificate     /etc/letsencrypt/live/simba.yourdomain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/simba.yourdomain.com/privkey.pem;
```

---

## Deploy nginx.conf

```bash
# Backup the default nginx config
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# Copy your production config
sudo cp nginx.conf /etc/nginx/nginx.conf

# Test the config for syntax errors — ALWAYS do this before reloading
sudo nginx -t

# Expected output:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Apply the config
sudo systemctl reload nginx
```

---

## DNS Setup — `simba`

`localhost` and `127.0.0.1` only work on the **same machine**. To reach your server as `simba` from other machines, follow the steps below.

---

### Option A — Edit `/etc/hosts` on Each Machine (Simple)

Do this on **every machine** that needs to access `https://simba`.

**Linux / macOS:**
```bash
sudo nano /etc/hosts
```

**Windows** (run Notepad as Administrator, open this file):
```
C:\Windows\System32\drivers\etc\hosts
```

Add this line at the bottom — use the **Nginx server's real IP**, not `127.0.0.1`:
```
10.244.24.67    simba
```

Save and test:
```bash
ping simba
# Should resolve to 10.244.24.67
```

> ⚠️ **Why not `127.0.0.1`?**  
> `127.0.0.1` is the loopback address — it always means *"this machine itself"*.  
> If you add `127.0.0.1 simba` on another machine, it will try to connect to **that machine's** port 443, not your Nginx server.  
> Always use the server's actual network IP: `10.244.24.67`.

---

### Option B — LAN DNS with dnsmasq (Recommended for Production)

Install `dnsmasq` **on the Nginx server** so all LAN machines resolve `simba` automatically — no need to edit `/etc/hosts` on each machine.

```bash
sudo apt install dnsmasq -y

# Add simba DNS record
echo "address=/simba/10.244.24.67" | sudo tee -a /etc/dnsmasq.conf

# Restart dnsmasq
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq

# Open DNS port in firewall
sudo ufw allow 53/udp
sudo ufw allow 53/tcp
```

Then on each client machine, set the **DNS server** to `10.244.24.67` in their network settings (System Settings → Network → DNS).

---

## Access From Other Machines

| Scenario | URL to use |
|---|---|
| Same machine (loopback) | `https://127.0.0.1` or `https://localhost` |
| Any machine after `/etc/hosts` edit | `https://simba` |
| Any machine using direct IP | `https://10.244.24.67` |
| Any machine with dnsmasq DNS | `https://simba` (automatic) |

---

## Verify & Test

### 1. Check Nginx is running
```bash
sudo systemctl status nginx
```

### 2. Test HTTP → HTTPS redirect
```bash
curl -I http://simba
# Should return: 301 Moved Permanently → https://simba
```

### 3. Test HTTPS proxy
```bash
# -k flag skips cert verification for self-signed certs
curl -k https://simba
curl -k https://10.244.24.67

# Or with verbose TLS info
curl -kv https://simba 2>&1 | grep -E "SSL|subject|issuer|HTTP"
```

### 4. Test health check endpoint
```bash
curl -k https://simba/healthz
# Should return: OK
```

### 5. Check open ports
```bash
sudo ss -tlnp | grep nginx
# Should show: 0.0.0.0:80 and 0.0.0.0:443
```

### 6. Check firewall
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
sudo ufw status
```

---

## Managing Nginx

```bash
# Start
sudo systemctl start nginx

# Stop
sudo systemctl stop nginx

# Reload config (zero downtime — use this for config changes)
sudo systemctl reload nginx

# Full restart
sudo systemctl restart nginx

# View real-time access logs
sudo tail -f /var/log/nginx/access.log

# View real-time error logs
sudo tail -f /var/log/nginx/error.log

# Test config before applying
sudo nginx -t
```

---

## Troubleshooting

### Cannot connect to `https://simba` from another machine
- Confirm `/etc/hosts` has `10.244.24.67 simba` (not `127.0.0.1`)
- Check firewall: `sudo ufw status` — ports 80 and 443 must be allowed
- Ping the server: `ping 10.244.24.67`

### 502 Bad Gateway
- Your app on port 8080 is not running
- Check it: `curl http://10.244.24.67:8080`
- Restart your app and try again

### SSL certificate warning in browser
- Expected with self-signed certs — click **Advanced → Proceed**
- For no warnings, use Let's Encrypt with a real domain (Option B)

### Nginx fails to start — port already in use
```bash
sudo lsof -i :80
sudo lsof -i :443
# Kill the conflicting process or stop Apache if installed
sudo systemctl stop apache2
```

### `nginx -t` config test fails
```bash
sudo nginx -t 2>&1
# Read the error line carefully — usually a missing semicolon or wrong file path
```

---

## Production Checklist

Before going live, verify all of the following:

- [ ] Nginx installed and enabled on boot (`systemctl enable nginx`)
- [ ] SSL certificate in place (`/etc/nginx/ssl/simba.crt` and `.key`)
- [ ] `nginx -t` passes with no errors
- [ ] HTTP → HTTPS redirect works (`curl -I http://simba` returns 301)
- [ ] App accessible at `https://simba` from another machine
- [ ] Firewall allows ports 80 and 443 (`ufw status`)
- [ ] `server_tokens off` in config (hides Nginx version)
- [ ] HSTS header active (check with `curl -kI https://simba | grep Strict`)
- [ ] Error logs monitored (`/var/log/nginx/error.log`)
- [ ] `/etc/hosts` or dnsmasq configured on all client machines
- [ ] Self-signed cert replaced with Let's Encrypt for public-facing domains

---

## File Structure

```
/etc/nginx/
├── nginx.conf              ← main config (this repo)
└── ssl/
    ├── simba.crt           ← SSL certificate
    └── simba.key           ← SSL private key

/var/log/nginx/
├── access.log              ← all requests
└── error.log               ← errors and warnings
```

---

## Quick Reference

| Item | Value |
|---|---|
| App backend | `http://10.244.24.67:8080` |
| Nginx server IP | `10.244.24.67` |
| DNS name | `simba` |
| Public URL | `https://simba` |
| HTTP port | `80` (redirects to HTTPS) |
| HTTPS port | `443` |
| SSL cert path | `/etc/nginx/ssl/simba.crt` |
| SSL key path | `/etc/nginx/ssl/simba.key` |
| Access log | `/var/log/nginx/access.log` |
| Error log | `/var/log/nginx/error.log` |
