---
title: "How to setup Sentry self-hosted, Nginx with Let's Encrypt"
date:   2026-06-12 10:00:00 +0300
categories:
  - blog
tags:
  - sentry
  - crashes
  - docker
---

## 1. Install Sentry

First, prepare your Ubuntu server by adding a [user](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu) and installing [Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04).

These are the minimum requirements for Sentry self-hosted:

- 4 CPU Cores
- 16 GB RAM + 16 GB swap
- 20 GB Free Disk Space

### 1.1 Install from repository:
```
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/getsentry/self-hosted/releases/latest)
VERSION=${VERSION##*/}
git clone https://github.com/getsentry/self-hosted.git
cd self-hosted
git checkout ${VERSION}
./install.sh
```

### 1.2 Update sentry/config.yml

Update system.url-prefix:
```
system.url-prefix: 'https://sentry.example.com'
```


Update email settings:
```
###############
# Mail Server #
###############

mail.backend: 'smtp'
mail.host: 'mail.smtp2go.com'
mail.port: 587
mail.username: '<USERNAME>'
mail.password: '<PWD>'
```

### 1.3 Update sentry/sentry.conf.py

Update CSRF:
```
CSRF_TRUSTED_ORIGINS = ["https://sentry.example.com"]
```

Enable SSL/TLS
```
###########
# SSL/TLS #
###########

# If you're using a reverse SSL proxy, you should enable the X-Forwarded-Proto
# header and enable the settings below

SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
USE_X_FORWARDED_HOST = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SOCIAL_AUTH_REDIRECT_IS_HTTPS = True
```
### 1.4 Restart docker
```
docker composer down
docker composer up -d
```

## 2. Install Nginx
```
sudo apt install nginx certbot python3-certbot-nginx
```

### 2.2 Create 'sentry' nginx config:

```
sudo nano /etc/nginx/sites-available/sentry
```

set initial config:
```
server {
    listen 80;
    listen [::]:80;
    server_name sentry.example.com;
    root /var/www/html;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 404;
    }
}
```

### 2.2 Check config file and reload server
```
sudo nginx -t
sudo systemctl reload nginx
```

### 2.3 Request certifcate
```
sudo mkdir -p /var/www/html
sudo certbot certonly --webroot -w /var/www/html -d sentry.example.com --email support@example.com --agree-tos --no-eff-email
```

### 2.4 Create dhparam:
```
sudo ls /etc/letsencrypt/ffdhe2048.txt || sudo curl -o /etc/letsencrypt/ffdhe2048.txt https://ssl-config.mozilla.org/ffdhe2048.txt
```

### 2.5 Set full nginx config:

```
sudo nano /etc/nginx/sites-available/sentry
```

````
error_log /var/log/nginx/error.log warn;

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name sentry.example.com;

    ssl_certificate     /etc/letsencrypt/live/sentry.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sentry.example.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache   shared:MozSSL:10m;
    ssl_session_tickets off;
    ssl_dhparam         /etc/letsencrypt/ffdhe2048.txt;
    ssl_protocols       TLSv1.3;
    ssl_prefer_server_ciphers off;

    add_header Strict-Transport-Security "max-age=63072000" always;

    client_max_body_size 100m;

    proxy_buffering   on;
    proxy_buffer_size 128k;
    proxy_buffers     4 256k;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    location ~ ^/api/[0-9]+/(envelope|minidump|security|store|unreal)/ {
        add_header Access-Control-Allow-Origin      * always;
        add_header Access-Control-Allow-Credentials false always;
        add_header Access-Control-Allow-Methods     GET,POST,PUT always;
        add_header Access-Control-Allow-Headers     sentry-trace,baggage always;
        add_header Access-Control-Expose-Headers    sentry-trace,baggage always;

        include proxy_params;
        proxy_pass http://127.0.0.1:9000;
    }

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:9000;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name sentry.example.com;
    root /var/www/html;

    location /.well-known/ {
        try_files $uri =404;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
````

### 2.6 Check nginx config and reload:
```
sudo nginx -t
sudo systemctl reload nginx
curl -I https://sentry.example.com
```

### 2.6 Restart sentry docker:
```
docker compose down
docker compose up -d
```

## Links:
 - https://develop.sentry.dev/self-hosted/
 - https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04
 - https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu
 - https://support.google.com/accounts/answer/185833?hl=en