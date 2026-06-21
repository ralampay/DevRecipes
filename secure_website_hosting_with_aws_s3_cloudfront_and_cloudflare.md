# Secure Website Hosting with AWS S3, Cloudflare and AWS CloudFront

## Overview

This guide describes a secure, scalable, and cost-effective architecture for hosting static websites using:

- Amazon S3
- Amazon CloudFront
- AWS Certificate Manager (ACM)
- Cloudflare

## The Problem

Many websites start with:

```text
Visitor
  ↓
Cloudflare
  ↓
S3 Website Endpoint
```

This works, but has several drawbacks:

- S3 website endpoints require public access.
- S3 website endpoints do not support HTTPS for custom domains.
- Cloudflare Full (Strict) SSL cannot validate the S3 website endpoint.
- Content is served from a single AWS region.
- Direct S3 access may remain exposed.

## The Solution

Use CloudFront between Cloudflare and S3:

```text
Visitor
   ↓
Cloudflare
   ↓
CloudFront
   ↓
Private S3 Bucket
```

Benefits:

- End-to-end HTTPS
- Private S3 bucket
- Global CDN
- Better performance
- Better security
- Cloudflare Full (Strict) compatibility

---

# Architecture Components

## Amazon S3

Stores:

- HTML
- CSS
- JavaScript
- Images
- Downloads

S3 becomes the origin for CloudFront.

## Amazon CloudFront

Provides:

- Global CDN
- Edge caching
- SSL termination
- Origin protection

## AWS Certificate Manager (ACM)

Provides:

- Free SSL certificates
- Automatic renewal

**Important:** CloudFront requires ACM certificates in **us-east-1 (N. Virginia)**.

## Cloudflare

Provides:

- DNS hosting
- DDoS protection
- Optional WAF
- Additional caching

---

# Deployment Process

## Step 1: Create an S3 Bucket

Example:

```text
cloudbandsolutions.com
```

Upload:

```text
index.html
assets/
```

Do not enable public access permanently.

---

## Step 2: Request an ACM Certificate

Region:

```text
us-east-1
```

Request a certificate for:

```text
cloudbandsolutions.com
```

or

```text
alive.ateneo.edu
```

Choose:

```text
DNS Validation
```

---

## Step 3: Validate the Certificate

AWS provides a DNS CNAME record.

Example:

```text
Type: CNAME
Name: _xxxx.example.com
Value: _yyyy.acm-validations.aws
```

Create the record and wait until the certificate becomes:

```text
Issued
```

---

## Step 4: Create a CloudFront Distribution

Origin:

```text
mybucket.s3.ap-southeast-1.amazonaws.com
```

Use the S3 endpoint.

Do NOT use:

```text
mybucket.s3-website-ap-southeast-1.amazonaws.com
```

---

## Step 5: Enable Origin Access Control (OAC)

Enable:

```text
Origin Access Control (OAC)
```

This allows CloudFront to securely access a private bucket.

---

## Step 6: Configure CloudFront

Set:

```text
Default Root Object: index.html
```

Attach:

```text
ACM Certificate
```

Add:

```text
Alternate Domain Name
```

Examples:

```text
cloudbandsolutions.com
alive.ateneo.edu
```

---

## Step 7: Configure DNS

Point the domain to CloudFront.

Example:

```text
cloudbandsolutions.com
    →
d123456abcdef.cloudfront.net
```

Cloudflare record:

```text
Type: CNAME
Target: d123456abcdef.cloudfront.net
```

---

## Step 8: Configure Cloudflare SSL

Set:

```text
SSL/TLS → Full (Strict)
```

Result:

```text
Browser
  ↓ HTTPS
Cloudflare
  ↓ HTTPS
CloudFront
  ↓
Private S3
```

---

## Step 9: Lock Down S3

After CloudFront is working:

Disable:

```text
Static Website Hosting
```

Enable:

```text
Block Public Access
```

Keep the CloudFront OAC policy.

---

# Multi-Site Example

```text
cloudbandsolutions.com
bni-pis.cloudbandsolutions.com
merfolk-pis.cloudbandsolutions.com
sdm-erp.cloudbandsolutions.com
```

Recommended:

- One CloudFront distribution per site
- Private S3 bucket per site
- ACM certificate per domain or wildcard certificate

---

# University DNS Scenario

Example:

```text
alive.ateneo.edu
```

When DNS is managed by another team:

1. Request ACM certificate.
2. Send validation CNAME to DNS administrators.
3. Create CloudFront distribution.
4. Wait for ACM to become Issued.
5. Add Alternate Domain Name.
6. Ask administrators to point DNS to CloudFront.

Example:

```text
alive.ateneo.edu
→
d46jhfaxl5ft1.cloudfront.net
```

---

# Final Architecture

```text
Internet Users
       ↓
Cloudflare
       ↓
CloudFront
       ↓
Private S3 Bucket
```

This architecture provides:

- Secure HTTPS hosting
- Private storage
- Better performance
- Global CDN
- Cloudflare protection
- Automatic certificate renewal
- AWS best-practice security
