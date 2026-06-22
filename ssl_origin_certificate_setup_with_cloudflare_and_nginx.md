# SSL Origin Certificate Setup with Cloudflare and Nginx

## Overview

This guide explains how to install a Cloudflare Origin Certificate on an EC2 server running Nginx and configure Cloudflare SSL/TLS mode as **Full (Strict)**.

This is useful when your API subdomains are proxied through Cloudflare and served by Nginx on EC2.

Example API domains:

```text
sdm-erp-api.cloudbandsolutions.com
bni-pis-api.cloudbandsolutions.com
merfolk-pis-api.cloudbandsolutions.com
```

Final architecture:

```text
Browser
  ↓ HTTPS
Cloudflare
  ↓ HTTPS
Nginx on EC2
  ↓ HTTP localhost
Backend Application
```

---

## Problem Being Solved

When Cloudflare is set to **Flexible**, the connection is:

```text
Browser
  ↓ HTTPS
Cloudflare
  ↓ HTTP
Nginx / EC2
```

This works even if Nginx only listens on port `80`.

When Cloudflare is changed to **Full (Strict)**, the connection becomes:

```text
Browser
  ↓ HTTPS
Cloudflare
  ↓ HTTPS
Nginx / EC2
```

This means Nginx must:

1. Listen on port `443`
2. Have a valid certificate trusted by Cloudflare
3. Be reachable through the EC2 Security Group on port `443`

If not, you may see:

```text
521 Web Server Down
525 SSL Handshake Failed
526 Invalid SSL Certificate
```

---

# Step 1: Check Current Nginx Ports

SSH into the EC2 instance:

```bash
ssh ubuntu@YOUR_EC2_PUBLIC_IP
```

Check Nginx listeners:

```bash
sudo ss -tulpn | grep nginx
```

If you only see:

```text
0.0.0.0:80
```

then Nginx is not yet ready for Cloudflare Full (Strict).

You eventually want to see:

```text
0.0.0.0:80
0.0.0.0:443
```

---

# Step 2: Create a Cloudflare Origin Certificate

In Cloudflare:

```text
Cloudflare Dashboard
→ Select domain
→ SSL/TLS
→ Origin Server
→ Create Certificate
```

Choose:

```text
Generate private key and CSR with Cloudflare
```

Recommended settings:

```text
Private Key Type: RSA 2048

Hostnames:
*.cloudbandsolutions.com
cloudbandsolutions.com

Validity:
15 years
```

Cloudflare will show:

1. **Origin Certificate**
2. **Private Key**

Copy both values.

---

# Step 3: Create Certificate Directory on EC2

```bash
sudo mkdir -p /etc/nginx/ssl
```

---

# Step 4: Save the Origin Certificate

```bash
sudo nano /etc/nginx/ssl/cloudflare-origin.crt
```

Paste the **Origin Certificate**.

It should look like:

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

Save and exit.

For nano:

```text
CTRL + O
ENTER
CTRL + X
```

---

# Step 5: Save the Private Key

```bash
sudo nano /etc/nginx/ssl/cloudflare-origin.key
```

Paste the **Private Key**.

It should look like:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

Save and exit.

---

# Step 6: Secure the Certificate Files

Use `root:root`, not `ubuntu:ubuntu`.

```bash
sudo chown root:root /etc/nginx/ssl/cloudflare-origin.crt
sudo chown root:root /etc/nginx/ssl/cloudflare-origin.key

sudo chmod 644 /etc/nginx/ssl/cloudflare-origin.crt
sudo chmod 600 /etc/nginx/ssl/cloudflare-origin.key
```

Verify:

```bash
ls -l /etc/nginx/ssl
```

Expected:

```text
-rw-r--r-- 1 root root  ... cloudflare-origin.crt
-rw------- 1 root root  ... cloudflare-origin.key
```

---

# Step 7: Backup Existing Nginx Config

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup.$(date +%Y%m%d_%H%M%S)
```

---

# Step 8: Proper Nginx Config

Below is a complete Nginx configuration based on this setup:

- `pisbniapi` runs on `127.0.0.1:8080`
- `sdmerpapi` runs on `127.0.0.1:8081`
- `pismerfolkapi` runs on `127.0.0.1:3000`

Open Nginx config:

```bash
sudo nano /etc/nginx/nginx.conf
```

Use this configuration:

```nginx
user ubuntu;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    ##
    # Basic Settings
    ##

    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # SSL Settings
    ##

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    ##
    # Logging Settings
    ##

    access_log /var/log/nginx/access.log;

    ##
    # Gzip Settings
    ##

    gzip on;

    ##
    # Upstreams
    ##

    upstream pisbniapi {
        server 127.0.0.1:8080;
    }

    upstream sdmerpapi {
        server 127.0.0.1:8081;
    }

    upstream pismerfolkapi {
        server 127.0.0.1:3000;
    }

    ##
    # Redirect all HTTP API traffic to HTTPS
    ##

    server {
        listen 80;
        server_name
            merfolk-pis-api.cloudbandsolutions.com
            bni-pis-api.cloudbandsolutions.com
            sdm-erp-api.cloudbandsolutions.com;

        return 301 https://$host$request_uri;
    }

    ##
    # Merfolk PIS API
    ##

    server {
        listen 443 ssl;
        server_name merfolk-pis-api.cloudbandsolutions.com;

        ssl_certificate     /etc/nginx/ssl/cloudflare-origin.crt;
        ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;

        client_max_body_size 20m;

        location / {
            proxy_pass http://pismerfolkapi;

            proxy_http_version 1.1;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port 443;
        }
    }

    ##
    # BNI PIS API
    ##

    server {
        listen 443 ssl;
        server_name bni-pis-api.cloudbandsolutions.com;

        ssl_certificate     /etc/nginx/ssl/cloudflare-origin.crt;
        ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;

        client_max_body_size 20m;

        location / {
            proxy_pass http://pisbniapi;

            proxy_http_version 1.1;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port 443;
        }
    }

    ##
    # SDM ERP API
    ##

    server {
        listen 443 ssl;
        server_name sdm-erp-api.cloudbandsolutions.com;

        ssl_certificate     /etc/nginx/ssl/cloudflare-origin.crt;
        ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;

        client_max_body_size 20m;

        location / {
            proxy_pass http://sdmerpapi;

            proxy_http_version 1.1;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port 443;
        }
    }
}
```

Save and exit.

---

# Step 9: Validate Nginx Config

```bash
sudo nginx -t
```

Expected:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Do not reload Nginx if the test fails.

---

# Step 10: Reload Nginx

```bash
sudo systemctl reload nginx
```

Verify that Nginx is listening on both `80` and `443`:

```bash
sudo ss -tulpn | grep nginx
```

Expected:

```text
0.0.0.0:80
0.0.0.0:443
```

---

# Step 11: Open Port 443 in AWS EC2 Security Group

In AWS:

```text
EC2
→ Instances
→ Select instance
→ Security
→ Security Groups
→ Edit inbound rules
```

Allow:

```text
HTTP   TCP 80   0.0.0.0/0
HTTPS  TCP 443  0.0.0.0/0
```

You can later restrict access to Cloudflare IP ranges, but start with this while testing.

---

# Step 12: Check UFW Firewall

If UFW is active:

```bash
sudo ufw status
```

Allow HTTP and HTTPS:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

---

# Step 13: Set Cloudflare SSL Mode

In Cloudflare:

```text
SSL/TLS
→ Overview
→ Full (Strict)
```

This ensures:

```text
Browser
  ↓ HTTPS
Cloudflare
  ↓ HTTPS
Nginx
```

---

# Step 14: Ensure DNS Records Are Proxied

For Cloudflare Origin Certificates, the DNS records should normally be orange-cloud proxied:

```text
sdm-erp-api.cloudbandsolutions.com       Proxied
bni-pis-api.cloudbandsolutions.com       Proxied
merfolk-pis-api.cloudbandsolutions.com   Proxied
```

This matters because Cloudflare Origin Certificates are trusted by Cloudflare, not directly by browsers.

---

# Step 15: Test the API Domains

Test HTTP redirect:

```bash
curl -I http://sdm-erp-api.cloudbandsolutions.com
```

Expected:

```text
HTTP/1.1 301 Moved Permanently
Location: https://sdm-erp-api.cloudbandsolutions.com/
```

Test HTTPS:

```bash
curl -I https://sdm-erp-api.cloudbandsolutions.com
```

A valid result may be:

```text
HTTP/2 200
```

or:

```text
HTTP/2 404
```

A `404` can still be acceptable if the API root path does not exist.

The important thing is that you should no longer see:

```text
521
525
526
```

Test all APIs:

```bash
curl -I https://sdm-erp-api.cloudbandsolutions.com
curl -I https://bni-pis-api.cloudbandsolutions.com
curl -I https://merfolk-pis-api.cloudbandsolutions.com
```

---

# Troubleshooting

## 521 Web Server Down

Cloudflare cannot connect to the origin.

Check:

```bash
sudo ss -tulpn | grep nginx
```

Make sure Nginx listens on:

```text
0.0.0.0:443
```

Also verify AWS Security Group allows port `443`.

---

## 525 SSL Handshake Failed

Cloudflare reached the server, but SSL negotiation failed.

Check:

```bash
sudo nginx -t
```

Verify paths:

```nginx
ssl_certificate     /etc/nginx/ssl/cloudflare-origin.crt;
ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;
```

Also ensure the private key matches the certificate.

---

## 526 Invalid SSL Certificate

Cloudflare reached the origin but rejected the certificate.

Check:

- Cloudflare SSL mode is Full (Strict)
- DNS record is proxied
- Certificate covers `*.cloudbandsolutions.com`
- The certificate and key were copied correctly

---

## Browser Shows Certificate Error When Accessing EC2 Directly

This is expected.

Cloudflare Origin Certificates are trusted by Cloudflare, not by normal browsers.

Users should access the API through:

```text
https://sdm-erp-api.cloudbandsolutions.com
```

not through:

```text
https://EC2_PUBLIC_IP
```

---

# Optional Hardening

After everything works, you may restrict the EC2 Security Group to allow only Cloudflare IP ranges on ports `80` and `443`.

This reduces direct exposure of the origin server.

However, Cloudflare IP ranges can change, so this requires maintenance.

---

# Final Working Setup

```text
User Browser
    ↓ HTTPS
Cloudflare
    ↓ HTTPS using Cloudflare Origin Certificate
Nginx on EC2
    ↓ HTTP localhost
Backend Application
```

This setup lets your API endpoints work properly with:

```text
Cloudflare SSL/TLS: Full (Strict)
```

while keeping your backend applications behind Nginx.
