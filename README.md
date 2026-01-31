# NGINX — From Basics to Advanced (Study + Real‑Time Ops Guide)

Generated from your uploaded config: `nginx_backup[.txt`.

This README is designed for **revision + real-time work**. It starts from **fundamentals** and builds up to **advanced production patterns**, using your config as the running example.

---

## Table of Contents

1. [Quick start: what your config does](#1-quick-start-what-your-config-does)
2. [NGINX fundamentals (Basics)](#2-nginx-fundamentals-basics)
3. [HTTP serving & locations (Intermediate)](#3-http-serving--locations-intermediate)
4. [Performance tuning (Intermediate)](#4-performance-tuning-intermediate)
5. [Security hardening (Intermediate → Advanced)](#5-security-hardening-intermediate--advanced)
6. [TLS/SSL + HTTP/2 (Intermediate → Advanced)](#6-tlsssl--http2-intermediate--advanced)
7. [PHP-FPM + FastCGI (Intermediate)](#7-php-fpm--fastcgi-intermediate)
8. [Caching & Microcaching (Advanced)](#8-caching--microcaching-advanced)
9. [Rate limiting & DDoS controls (Advanced)](#9-rate-limiting--ddos-controls-advanced)
10. [Observability: logging, metrics, debugging (Advanced)](#10-observability-logging-metrics-debugging-advanced)
11. [Deployment patterns (Advanced)](#11-deployment-patterns-advanced)
12. [Troubleshooting playbook](#12-troubleshooting-playbook)
13. [Hands-on labs (practice tasks)](#13-hands-on-labs-practice-tasks)
14. [Interview/Revision flashcards](#14-interviewrevision-flashcards)
15. [Appendix: your original config](#15-appendix-your-original-config)

---

## 1) Quick start: what your config does

### Traffic flow
- **Port 80** redirects all traffic to **HTTPS** using a 301 redirect.
- **Port 443** serves content from `root /home/raja/Rajendra/demo/` with **HTTP/2**, **TLS**, **Basic Auth**, **FastCGI (PHP-FPM)**, **microcaching**, and **rate limiting on `/thumb.png`**.

### Major features present
- Worker tuning: `worker_processes auto;` and `worker_connections 1024;`
- gzip compression enabled
- Client body/header buffers and timeouts configured
- `server_tokens off;` to hide version
- `fastcgi_cache` microcache with cache status header `X-Cache`
- TLS configuration + DH params + HSTS
- Basic authentication using `.htpasswd`
- Security headers: `X-Frame-Options`, `X-XSS-Protection`

---

## 2) NGINX fundamentals (Basics)

### 2.1 NGINX architecture
- NGINX is **event-driven**: one worker can handle many connections.
- Common performance levers:
  - Workers (`worker_processes`)
  - Per-worker connections (`worker_connections`)
  - OS limits (`ulimit -n`, `worker_rlimit_nofile`)

**From your config**
```nginx
user www-data;
worker_processes auto;

events {
  worker_connections 1024;
}
```

### 2.2 Configuration contexts (must know)
- `main` (top-level): global directives like `user`, `worker_processes`, `load_module`
- `events`: connection processing (epoll/kqueue), connection limits
- `http`: HTTP engine settings (gzip, logs, caches)
- `server`: virtual host (listen/hostnames)
- `location`: routing per path/regex

### 2.3 Basic commands (daily use)
```bash
# test config
sudo nginx -t

# reload without downtime
sudo systemctl reload nginx
# or
sudo nginx -s reload

# restart (downtime for existing connections)
sudo systemctl restart nginx

# see active config (build + modules)
nginx -V
```

---

## 3) HTTP serving & locations (Intermediate)

### 3.1 Serving static files (root + index)
```nginx
root /home/raja/Rajendra/demo/;
index index.html index.htm index.php;
```
- When a directory is requested, NGINX tries `index.html`, then `index.htm`, then `index.php`.

### 3.2 Location matching order (high priority topic)
You had many commented examples showing all matching styles.

**Order to remember**
1. `location = /exact` (exact match)
2. `location ^~ /prefix` (prefer prefix, stop regex search)
3. `location ~ /regex` and `~*` (regex matches)
4. `location /prefix` (longest prefix match)

### 3.3 `try_files` (production must-have)
You use `try_files` in multiple locations:
```nginx
try_files $uri $uri/ =404;
```
Typical uses:
- Static site: serve file, else 404
- SPA: `try_files $uri /index.html;`
- Fallback custom error pages

### 3.4 Rewrites and redirects
You have several rewrite examples commented.

**When to use**
- Redirect old URLs to new ones (SEO)
- Rewrite internal routing for apps

Rule of thumb:
- Prefer `return 301/302` for simple redirects
- Use `rewrite` for more complex patterns

---

## 4) Performance tuning (Intermediate)

### 4.1 gzip compression
From your config:
```nginx
gzip on;
gzip_types text/plain application/xml text/css application/javascript;
gzip_comp_level 3;
```
**Recommended production add-ons**
```nginx
gzip_vary on;
gzip_min_length 1024;
gzip_proxied any;
```

### 4.2 Efficient file I/O
From your config:
```nginx
sendfile on;
tcp_nopush on;
```
- `sendfile` reduces copies between kernel/user space.
- `tcp_nopush` optimizes packet batching (good for large static files).

### 4.3 Buffers and timeouts (anti-slow-client + stability)
From your config:
```nginx
client_body_buffer_size 16k;
client_max_body_size 8m;
client_header_buffer_size 1k;

client_body_timeout 12;
client_header_timeout 12;
keepalive_timeout 15;
send_timeout 10;
```
**How to tune**
- Upload errors? Increase `client_max_body_size`.
- Slow clients? Reduce timeouts.
- Big headers (JWT/cookies)? Increase header buffers.

---

## 5) Security hardening (Intermediate → Advanced)

### 5.1 Hide server version
```nginx
server_tokens off;
```

### 5.2 Basic authentication
Your config protects `/`:
```nginx
location / {
  auth_basic "Secure Area";
  auth_basic_user_file /etc/nginx/.htpasswd;
  try_files $uri $uri/ =404;
}
```
Create users:
```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd user1
```

### 5.3 Security headers (modernize)
You currently set:
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block";
```
**Add these (common baseline)**
```nginx
add_header X-Content-Type-Options nosniff;
add_header Referrer-Policy strict-origin-when-cross-origin;
# Permissions-Policy depends on your app
# Content-Security-Policy requires careful tuning
```

### 5.4 Host header safety (important)
Your HTTP→HTTPS redirect uses `$host`.
If requests can come with untrusted Host headers, consider:
```nginx
return 301 https://127.0.0.1$request_uri;
# or your domain
```

---

## 6) TLS/SSL + HTTP/2 (Intermediate → Advanced)

### 6.1 TLS basics in your config
```nginx
listen 443 ssl http2;
ssl_certificate /etc/nginx/ssl/self.crt;
ssl_certificate_key /etc/nginx/ssl/self.key;

ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";

ssl_dhparam /etc/nginx/ssl/dhparam.pem;
add_header Strict-Transport-Security "max-age=31536000" always;

ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets on;
```

### 6.2 Recommended hardening (production baseline)
- Disable TLSv1 / TLSv1.1:
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```
- Consider turning session tickets off unless you rotate ticket keys:
```nginx
ssl_session_tickets off;
```

### 6.3 HTTP/2 notes
You include an HTTP/2 push example:
```nginx
location = /index.html {
  http2_push /style.css;
  http2_push /thumb.png;
}
```
Modern browsers have reduced support for server push; prefer `preload` + caching.

---

## 7) PHP-FPM + FastCGI (Intermediate)

### 7.1 PHP requests routing
From your config:
```nginx
location ~ \.php$ {
  include fastcgi.conf;
  fastcgi_pass unix://run/php/php8.1-fpm.sock;
}
```
**Operational checks**
- Socket exists:
```bash
ls -l /run/php/php8.1-fpm.sock
```
- PHP-FPM running:
```bash
systemctl status php8.1-fpm
```
- File permissions: NGINX (`www-data`) must read `root` content.

---

## 8) Caching & Microcaching (Advanced)

### 8.1 What you configured
```nginx
fastcgi_cache_path /tmp/nginx_cache levels=1:2 keys_zone=ZONE_1:100m inactive=10m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
add_header X-Cache $upstream_cache_status;

set $no_cache 0;
if ($arg_skip_cache = 1) { set $no_cache 1; }

fastcgi_cache ZONE_1;
fastcgi_cache_valid 200 302 404 10m;
fastcgi_cache_bypass $no_cache;
fastcgi_no_cache $no_cache;
```

### 8.2 Must-know cache concepts
- `keys_zone=ZONE_1:100m` stores cache **metadata** in shared memory.
- Cache entries stored on disk at `/tmp/nginx_cache`.
- `X-Cache` shows `HIT/MISS/BYPASS/EXPIRED`.

### 8.3 Make caching correct (real-time)
**Avoid caching private pages:**
- If user is logged in (cookie present)
- If request is POST
- If backend sets cookies

Common patterns (adjust for your app):
```nginx
# example only: be careful to not disable cache entirely
if ($request_method = POST) { set $no_cache 1; }
if ($http_cookie != "") { set $no_cache 1; }
```

### 8.4 Advanced cache improvements
Recommended directives:
```nginx
fastcgi_cache_lock on;
fastcgi_cache_use_stale error timeout invalid_header updating http_500 http_503;
```

---

## 9) Rate limiting & DDoS controls (Advanced)

### 9.1 Your current rate limit
```nginx
limit_req_zone $request_uri zone=MYZONE:10m rate=1r/s;

location = /thumb.png {
  limit_req zone=MYZONE burst=5 nodelay;
}
```

### 9.2 Key issue in your design
- Key is `$request_uri` → rate bucket is shared by everyone for that URI.
- If you want per-client limiting, use `$binary_remote_addr`.

Recommended:
```nginx
limit_req_zone $binary_remote_addr zone=MYZONE:10m rate=1r/s;
limit_req_status 429;
```

### 9.3 Add connection limiting (often needed)
```nginx
limit_conn_zone $binary_remote_addr zone=conn_zone:10m;
# inside server/location
limit_conn conn_zone 20;
```

---

## 10) Observability: logging, metrics, debugging (Advanced)

### 10.1 Logs (you should add)
```nginx
access_log /var/log/nginx/access.log;
error_log  /var/log/nginx/error.log warn;
```

### 10.2 Debugging requests
```bash
# show headers
curl -I https://127.0.0.1/ -k

# show cache status
curl -I https://127.0.0.1/index.php -k | grep -i x-cache
```

### 10.3 Useful log_format (suggestion)
Create a format with upstream timings:
```nginx
log_format main_ext '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" rt=$request_time urt=$upstream_response_time '
                    'uaddr=$upstream_addr cache=$upstream_cache_status';
access_log /var/log/nginx/access.log main_ext;
```

---

## 11) Deployment patterns (Advanced)

### 11.1 Split config (best practice)
Typical structure:
- `/etc/nginx/nginx.conf` (global)
- `/etc/nginx/conf.d/*.conf` (shared snippets)
- `/etc/nginx/sites-available/<site>` + symlink to `sites-enabled/`

### 11.2 Reverse proxy (when you add Node/Java)
Study these directives:
- `upstream {}`
- `proxy_pass` / `proxy_set_header`
- WebSocket proxying (`Upgrade`/`Connection` headers)
- `proxy_cache` (separate from fastcgi_cache)

### 11.3 Zero-downtime reload
- Use `nginx -t` before reload.
- Reload keeps old workers serving existing connections.

---

## 12) Troubleshooting playbook

### 12.1 Common errors and fixes

**A) 502 Bad Gateway (PHP)**
- Socket path wrong or PHP-FPM down.
- Fix: check `systemctl status php8.1-fpm`, socket file, permissions.

**B) 403 Forbidden**
- Permissions on `root` path.
- Fix: ensure NGINX user can read files.

**C) 413 Request Entity Too Large**
- `client_max_body_size` too low.
- Fix: increase at server/location.

**D) SSL handshake failures**
- Wrong cert paths/permissions or protocol mismatch.
- Fix: `nginx -t`, `openssl s_client -connect host:443`.

**E) Cache not working**
- Check `X-Cache` header.
- Ensure cache path is writable by NGINX.

---

## 13) Hands-on labs (practice tasks)

### Lab 1 — Location precedence
1. Create locations: `= /exact`, `/prefix`, `^~ /prefix2`, `~ \.php$`.
2. Hit endpoints and confirm which block matches.

### Lab 2 — gzip verification
```bash
curl -I -H 'Accept-Encoding: gzip' https://127.0.0.1/ -k
```
Check `Content-Encoding: gzip`.

### Lab 3 — FastCGI cache HIT/MISS
1. Request `index.php` twice.
2. Check `X-Cache` becomes HIT on the second request (if cacheable).

### Lab 4 — Rate limit 429
1. Switch key to `$binary_remote_addr`.
2. Add `limit_req_status 429;`
3. Stress test using `siege` (as you noted in comments).

### Lab 5 — Add static asset caching
Add a `location ~*` for static extensions and validate browser caching.

---

## 14) Interview/Revision flashcards

- What is the order of location matching?
- Difference between `rewrite` and `return`?
- What is `fastcgi_cache_lock` and why needed?
- How to avoid caching authenticated pages?
- What is HSTS and what is the risk?
- Why `server_tokens off` matters?
- Difference between `limit_req` and `limit_conn`?
- Why use `$binary_remote_addr` vs `$request_uri` for rate limit?
- What does `tcp_nopush` do and when to use it?

---

## 15) Appendix: your original config

```nginx
user www-data;

worker_processes auto;

load_module modules/ngx_http_image_filter_module.so;

events {
    worker_connections 1024;
    #worker _process x worker connections = max concurrent connections
}
http {
    include mime.types;
    #define limit rate
    #limit_req_zone $binary_remote_addr;
    limit_req_zone $request_uri zone=MYZONE:10m rate=1r/s; 
    gzip on;
    gzip_types text/plain application/xml text/css application/javascript;
    gzip_comp_level 3;

    #configure microcaching (fastcgi)
    fastcgi_cache_path /tmp/nginx_cache levels=1:2 keys_zone=ZONE_1:100m inactive=10m;
    fastcgi_cache_key "$scheme$request_method$host$request_uri";
    add_header X-Cache $upstream_cache_status;

    #hiding the servername in the curl response headers
    server_tokens off;

    #Buffer size for post submissions
    client_body_buffer_size 16k;
    client_max_body_size 8m;

    #Buffer size for headers
    client_header_buffer_size 1k;

    #max time to receive client header/body
    client_body_timeout 12;
    client_header_timeout 12;

    #max time to keep connection alive
    keepalive_timeout 15;

    #max time for client accept/receive a response
    send_timeout 10;

    #skip buffering for certain types of files
    sendfile on;

    #Optimize send packets
    tcp_nopush on;
    server {
        listen 80;
        server_name 127.0.0.1;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name 127.0.0.1;
        root /home/raja/Rajendra/demo/; 

        # #prefix match location
        # location /greet {

        #     return 200 "Hello still site is under construction!\n";
        # }

        # #exact match location
        # location = /welcome {
        #     return 200 "Welcome to our site!\n";
        # }

        # #regular expression location
        # location ~ /raja[0-9] {
        #     return 200 "PHP file requested\n";
        # }

        # #case insensitive regular expression location
        # location ~* /venakt[0-9] {
        #     return 200 "Case insensitive match\n";
        # }

        # #case sensitive regular expression location
        # location ~* /india[0-9] {
        #     return 200 "Case sensitive match\n";
        # }   

        # #preferential location
        # location ^~ /sreek1 {
        #     return 200 "This is the default location\n";
        # }
        # location /inspect {
        #     return 200 "$host\n$uri\n$remote_addr\n$remote_port\n$server_addr\n$server_port\n$scheme\n";
        # }
        #check static API requests

        # if ($arg_apikey != 12345) {
        #     return 403 "Forbidden: Invalid API Key\n";
        
        # }

        # location /inspect {
        #     return 200
        #         "name: $server_name\n
        #     uri: $uri\n
        #     remote address: $remote_addr\n
        #     remote port: $remote_port\n
        #     server address: $server_addr\n
        #     server port: $server_port\n
        #     scheme: $scheme\n";
        # }
        # set $weekend "No";

        # if ($date_local ~ (Sat|Sun)) {
        #     set $weekend "Yes";
        # }

        # location /isweekend {
        #     return 200 "Is it weekend? $weekend\n";
        # }

        # rewrite ^/user/(\w+) /greet/$1;

        # location /greet/raja {
        #    return 200 "thumb.png";
        # }

        #rewrite and redirect examples
        # rewrite ^/user/(\w+) /greet/$1;

        # location = /greet/raja {
        #    return 200 "Hello Raja!\n";
        # }

        # location /greet {
        #    return 200 "Hello User!\n";
        # }
        # rewrite ^/user/raja /thumb.png last;
        #     rewrite ^/user/(\w+) /greet/$1 last;

        #     location = /greet/raja {
        #         return 200 "Hello Raja!\n";
        #     }

        #     location /greet {
        #         return 200 "Hello User!\n";
        # }

        #try_files example
        
        # try_files $uri /cat.png /friendly_404;

        # location = /friendly_404 {
        #     internal;
        #     return 404 "Custom 404: File not found\n";
        # }

        # location /greet {
        #     return 200 "Hello User! This is try_files example\n";
        # }
        
        #logging example
        # location /secure {
        #     access_log /var/log/nginx/secure_access.log;
        #     error_log /var/log/nginx/secure_error.log;
        #     return 200 "This is secure location with custom logging\n";
        # }
        #cache by default
        set $no_cache 0;
        #check for cache bypass conditions
        if ($arg_skip_cache = 1) {
            set $no_cache 1;
        }
        # if ($request_method = POST) {
        #     set $no_cache 1;
        # }
        #inheritance and directive are of 3 types: standard array action 
        index index.html index.htm index.php;
        location ~ \.php$ {
         #pass fastcgi requests to the php-fpm socket
         include fastcgi.conf;
         fastcgi_pass unix://run/php/php8.1-fpm.sock; 
        }
        # #adding dynamic image filter
        # location = /thumb.png {
        #     image_filter rotate 90;
        # }
        # location = /thumb.png {
        #    add_header Cache-Control public;
        #    add_header Pragma public;
        #    add_header Vary Accept-Encoding;
        #    expires 60s;
        # }
        # location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        #    add_header Cache-Control public;
        #    add_header Pragma public;
        #    add_header Vary Accept-Encoding;
        #    expires 60s;
        # }
        #Enable caching
        fastcgi_cache ZONE_1;
        fastcgi_cache_valid 200 302 404 10m;
        fastcgi_cache_bypass $no_cache;
        fastcgi_no_cache $no_cache;

        #httpd2 have binary protocol compressed header and persistent connection and multiplex streaming server push
#openssl req -x509 -days 10 -nodes -newkey rsa:2048 -keyout /etc/nginx/ssl/self.key -out /etc/nginx/ssl/self.crt
        ssl_certificate /etc/nginx/ssl/self.crt;
        ssl_certificate_key /etc/nginx/ssl/self.key;
        #ssl disable
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
        #enable DH params
        ssl_dhparam /etc/nginx/ssl/dhparam.pem;
        # sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
        #enable hsts
        add_header Strict-Transport-Security "max-age=31536000" always;
        #ssl sessions
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_session_tickets on;
        
        #apt install nghttp2-client
        #nghttp -nysa https://localhost/index.html
        location = /index.html {
           http2_push /style.css;
           http2_push /thumb.png;
        }
        #Ratelimiting example
        # security against DDoS attacks reliability prevent traffic spikes and shaping - service prority 
        # install siege - 
        # siege -v -r 2 -c 5 https://localhost/thumb.png
        #siege -v -r 2 -c 5 --no-check-certificate https://127.0.0.1/thumb.png
        location = /thumb.png {
            limit_req zone=MYZONE burst=5 nodelay;
            try_files $uri $uri/ =404;
        }
        #hardening basic auth with nginx
        #apt install apache2-utils
        #sudo htpasswd -c /etc/nginx/.htpasswd user1
        location / {
            auth_basic "Secure Area";
            auth_basic_user_file /etc/nginx/.htpasswd;
            try_files $uri $uri/ =404;
        }
        #block iframe embedding
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
    }
}
```
