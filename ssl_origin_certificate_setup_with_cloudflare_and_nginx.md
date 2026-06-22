# SSL Origin Certificate Setup with Cloudflare and Nginx

## Overview

This guide explains how to install a Cloudflare Origin Certificate on an EC2 server running Nginx and configure Cloudflare Full (Strict) SSL.

Final architecture:

Browser -> Cloudflare (HTTPS) -> Nginx on EC2 (HTTPS) -> Backend Application

## Why This Is Needed

When Cloudflare SSL mode is changed from Flexible to Full (Strict), Cloudflare requires HTTPS connectivity to the origin server.

If Nginx only listens on port 80, Cloudflare may return:
- 521 Web Server Down
- 525 SSL Handshake Failed
- 526 Invalid SSL Certificate

## Step 1: Verify Current Nginx Listeners

```bash
sudo ss -tulpn | grep nginx
```

## Step 2: Create a Cloudflare Origin Certificate

Cloudflare Dashboard:

SSL/TLS -> Origin Server -> Create Certificate

Use:
- RSA 2048
- *.cloudbandsolutions.com
- cloudbandsolutions.com
- 15 years validity

## Step 3: Create SSL Directory

```bash
sudo mkdir -p /etc/nginx/ssl
```

## Step 4: Save Certificate

```bash
sudo nano /etc/nginx/ssl/cloudflare-origin.crt
```

Paste Origin Certificate.

## Step 5: Save Private Key

```bash
sudo nano /etc/nginx/ssl/cloudflare-origin.key
```

Paste Private Key.

## Step 6: Secure Files

```bash
sudo chown root:root /etc/nginx/ssl/cloudflare-origin.crt
sudo chown root:root /etc/nginx/ssl/cloudflare-origin.key
sudo chmod 644 /etc/nginx/ssl/cloudflare-origin.crt
sudo chmod 600 /etc/nginx/ssl/cloudflare-origin.key
```

## Step 7: Test and Reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Step 8: Open Security Group Ports

Allow:
- TCP 80
- TCP 443

## Step 9: Configure Cloudflare

SSL/TLS -> Full (Strict)

Keep DNS records proxied.

## Final Architecture

User Browser
-> Cloudflare
-> Nginx on EC2
-> Backend Application
