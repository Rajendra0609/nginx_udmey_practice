# NGINX Configuration Study & Operations Guide (Generated from `nginx_backup[.txt`)

> This README explains **what your NGINX config does**, **why each directive exists**, and provides a **real‑time operations + revision checklist**.
> Use it to quickly refresh NGINX concepts (HTTP, SSL/TLS, caching, rate limiting, PHP-FPM) and to harden/productionize the setup.

---

## 1) What this configuration is for (high level)

This configuration sets up:

- **NGINX worker/process tuning** (auto workers, 1024 connections)
- **HTTP features**: gzip compression, client timeouts/buffers, sendfile/tcp_nopush
- **Rate limiting** using `limit_req_zone` + `limit_req` (DDoS/abuse mitigation)
- **PHP-FPM** handling for `*.php` via a Unix socket (`php8.1-fpm.sock`)
- **Microcaching** using `fastcgi_cache` for dynamic responses
- **TLS + HTTP/2** with self-signed cert + DH params + HSTS
- **Basic Auth** to protect `/` (whole site) using `.htpasswd`
- Security headers: `X-Frame-Options`, `X-XSS-Protection`

It also contains many commented mini-labs for:

- Location matching types (`=`, prefix, regex, `^~`)
- Rewrites/redirects
- `try_files` patterns
- Custom logging locations
- Image filter module usage

---

## 2) Current behavior (what happens when traffic comes in)

### 2.1 Port 80 → redirect to HTTPS
```nginx
server {
  listen 80;
  server_name 127.0.0.1;
  return 301 https://$host$request_uri;
}
```
- All HTTP requests are redirected to HTTPS.
- Redirect uses `$host` (Host header) and the original `$request_uri`.

### 2.2 HTTPS server with HTTP/2 + PHP + caching + auth
```nginx
server {
  listen 443 ssl http2;
  server_name 127.0.0.1;
  root /home/raja/Rajendra/demo/;
  index index.html index.htm index.php;

  location ~ \.php$ {
    include fastcgi.conf;
    fastcgi_pass unix://run/php/php8.1-fpm.sock;
  }

  # caching
  fastcgi_cache ZONE_1;
  fastcgi_cache_valid 200 302 404 10m;
  fastcgi_cache_bypass $no_cache;
  fastcgi_no_cache $no_cache;

  # rate-limit only for /thumb.png
  location = /thumb.png {
    limit_req zone=MYZONE burst=5 nodelay;
    try_files $uri $uri/ =404;
  }

  # protect entire site with basic auth
  location / {
    auth_basic "Secure Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    try_files $uri $uri/ =404;
  }

  # security headers
  add_header X-Frame-Options "SAMEORIGIN";
  add_header X-XSS-Protection "1; mode=block";
}
```

---

## 3) Config walkthrough by topic (study notes)

### 3.1 Workers, events, concurrency
```nginx
user www-data;
worker_processes auto;

events {
  worker_connections 1024;
  # worker_processes * worker_connections ~= max concurrent connections
}
```
**Why it matters**
- `worker_processes auto;` sizes workers to CPU cores.
- `worker_connections` controls connections per worker.

**Real-time tips**
- Also consider `worker_rlimit_nofile`, `multi_accept on;` and OS limits (`ulimit -n`).

---

### 3.2 Loading dynamic modules (image filter)
```nginx
load_module modules/ngx_http_image_filter_module.so;
```
**What it does**
- Enables `image_filter` directives (rotate/resize/crop images on the fly).

**Gotcha**
- The module must exist at that path and match your NGINX build.

---

### 3.3 MIME types
```nginx
include mime.types;
```
- Ensures correct `Content-Type` headers for common file extensions.

---

### 3.4 Compression (gzip)
```nginx
gzip on;
gzip_types text/plain application/xml text/css application/javascript;
gzip_comp_level 3;
```
**Purpose**: Smaller responses → faster load times.

**Recommended additions (often used in prod)**
- `gzip_vary on;`
- `gzip_min_length 1024;`
- `gzip_proxied any;` (if behind a proxy)

---

### 3.5 Client buffers, size limits, and timeouts
```nginx
client_body_buffer_size 16k;
client_max_body_size 8m;
client_header_buffer_size 1k;

client_body_timeout 12;
client_header_timeout 12;
keepalive_timeout 15;
send_timeout 10;
```
**Why it matters**
- Helps mitigate slow-client attacks and controls memory usage.
- `client_max_body_size` affects uploads (file uploads/API payload limits).

**Real-time tuning**
- If you upload large files, increase `client_max_body_size`.
- You can set different limits per `server`/`location`.

---

### 3.6 Efficient file transfer
```nginx
sendfile on;
tcp_nopush on;
```
- Improves static file serving efficiency.

---

### 3.7 Server identity hardening
```nginx
server_tokens off;
```
- Hides NGINX version in error pages and some headers.

---

### 3.8 Rate limiting (anti-abuse / burst control)
```nginx
limit_req_zone $request_uri zone=MYZONE:10m rate=1r/s;

location = /thumb.png {
  limit_req zone=MYZONE burst=5 nodelay;
}
```
**What it does**
- Creates a shared memory zone `MYZONE` and tracks request rate.
- Applies limit only to `/thumb.png` with burst of 5.

**Important note (design concern)**
- Key is `$request_uri` → all users share the same bucket per URI.
- If you want per-client limiting, use `$binary_remote_addr`.

**Real-time improvements**
- Add `limit_req_status 429;`.
- Consider `limit_conn_zone` + `limit_conn` for connection limiting.

---

### 3.9 Microcaching for PHP (fastcgi_cache)
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
**How it works**
- Cache is stored in `/tmp/nginx_cache`.
- Key is scheme+method+host+URI.
- Adds `X-Cache: HIT/MISS/BYPASS/...` to responses.

**Common production additions**
- `fastcgi_cache_use_stale error timeout invalid_header updating http_500 http_503;`
- `fastcgi_cache_lock on;` (avoid thundering herd)
- Exclude cookies/sessions from cache
- Avoid caching POST requests (you already have a commented example)

---

### 3.10 PHP-FPM handling
```nginx
location ~ \.php$ {
  include fastcgi.conf;
  fastcgi_pass unix://run/php/php8.1-fpm.sock;
}
```
**Key checks**
- Socket path must exist.
- PHP-FPM pool user permissions must allow reading the `root` directory.

---

### 3.11 TLS/SSL + HTTP/2
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
**What it does**
- Enables HTTPS with HTTP/2.
- Uses self-signed cert (as per your openssl comment).
- Enables HSTS.

**Security improvements (recommended)**
- Use only `TLSv1.2 TLSv1.3` (TLSv1/1.1 are obsolete).
- Consider `ssl_session_tickets off;` unless you rotate keys.
- Add OCSP stapling when using publicly trusted certs.

---

### 3.12 HTTP/2 server push (demo)
```nginx
location = /index.html {
  http2_push /style.css;
  http2_push /thumb.png;
}
```
**Note**
- HTTP/2 server push is deprecated by some browsers.
- Prefer `preload` links + good caching.

---

### 3.13 Basic authentication
```nginx
location / {
  auth_basic "Secure Area";
  auth_basic_user_file /etc/nginx/.htpasswd;
}
```
**How to create `.htpasswd`**
```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd user1
```

---

### 3.14 Security headers
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block";
```
**Recommended modern header set**
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy: ...`
- `Content-Security-Policy: ...` (tune per app)

---

## 4) Commented examples (what you can learn from them)

Your file includes many commented blocks—treat them like a mini NGINX lab:

- Location matching: prefix, exact `=`, regex `~`, case-insensitive `~*`, preferential `^~`
- Variables inspection: `$host`, `$uri`, `$remote_addr`, `$scheme`, etc.
- API key validation using `if` + `$arg_apikey`
- Rewrite patterns and redirects
- `try_files` patterns for SPA / fallback 404
- Custom logs via `access_log` + `error_log`
- Image filter rotate example
- Static asset caching headers example
- Load testing with `siege`

---

## 5) What’s missing / suggested for “real production”

Below are additions you typically need in real environments.

### 5.1 Logging (highly recommended)
Add at `http {}` or inside server:
```nginx
access_log /var/log/nginx/access.log;
error_log  /var/log/nginx/error.log warn;
```
Optional: create a `log_format` including `$request_time`, `$upstream_response_time`.

### 5.2 Safer redirect on port 80
Using `$host` can be abused if Host header is not controlled. Prefer:
```nginx
return 301 https://127.0.0.1$request_uri;
# or use your real domain
```

### 5.3 Proper `server_name` and default server
In production, use domain name(s) and consider a default catch-all server:
```nginx
server {
  listen 80 default_server;
  server_name _;
  return 444;
}
```

### 5.4 TLS hardening defaults
Recommended baseline:
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_session_tickets off;
```

### 5.5 Cache correctness (avoid caching private pages)
- Don’t cache authenticated pages
- Don’t cache responses that set cookies
- Bypass cache for admin URLs

Example patterns:
```nginx
if ($request_method = POST) { set $no_cache 1; }
if ($http_cookie != "") { set $no_cache 1; }
```

### 5.6 Static asset caching
```nginx
location ~* \.(css|js|png|jpg|jpeg|gif|ico)$ {
  expires 30d;
  add_header Cache-Control "public, max-age=2592000, immutable";
}
```

### 5.7 Health endpoint
```nginx
location = /healthz {
  access_log off;
  return 200 "ok
";
}
```

### 5.8 Reverse proxy patterns (when you add upstreams)
Learn and practice:
- `upstream {}` blocks
- `proxy_pass`, `proxy_set_header`, `proxy_read_timeout`
- `proxy_cache` (for proxied apps)

---

## 6) Operations: validate, reload, debug

### Validate config
```bash
sudo nginx -t
```

### Reload without downtime
```bash
sudo systemctl reload nginx
# or
sudo nginx -s reload
```

### Check listening ports
```bash
sudo ss -lntp | grep nginx
```

### Debug headers & caching
```bash
curl -I https://127.0.0.1/index.php -k
# Look for: X-Cache: HIT/MISS/BYPASS
```

---

## 7) Study / Revision checklist (quick)

### Core NGINX
- [ ] Contexts: `main`, `events`, `http`, `server`, `location`
- [ ] Location precedence: `=`, `^~`, prefix, regex `~`/`~*`
- [ ] Variables: `$host`, `$uri`, `$args`, `$remote_addr`, `$request_uri`

### Performance
- [ ] gzip basics + tuning
- [ ] keepalive + timeouts
- [ ] sendfile/tcp_nopush/tcp_nodelay
- [ ] buffers and body size limits

### Security
- [ ] `server_tokens off`
- [ ] Basic auth
- [ ] Rate limit (zone, burst, nodelay, 429)
- [ ] Security headers (modern set)

### TLS/HTTP2
- [ ] Cert/key paths + permissions
- [ ] Protocols/ciphers baseline
- [ ] HSTS rollout and risks
- [ ] HTTP/2 troubleshooting

### Dynamic content
- [ ] PHP-FPM socket vs TCP
- [ ] fastcgi params (`SCRIPT_FILENAME`)

### Caching
- [ ] cache path/zone sizing
- [ ] cache key correctness
- [ ] bypass rules
- [ ] stale serving + cache lock

---

## 8) Quick next steps (recommended)

1) Fix rate limiting key if you want per-client limiting:
```nginx
limit_req_zone $binary_remote_addr zone=MYZONE:10m rate=1r/s;
```

2) Harden TLS (disable TLSv1/1.1; consider session tickets off).

3) Add logging and keep custom logs for sensitive paths.

4) Improve cache rules (exclude auth/session; add `cache_use_stale`/`cache_lock`).

5) Add modern security headers (CSP, nosniff, referrer-policy).

---

## 9) Reference: Original config (as uploaded)

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
