# Deploying to S3 with Github Actions

When building a frontend application, you often want every update pushed to the `master` branch to automatically build and deploy to production. Instead of manually running build commands and uploading files to Amazon S3, you can use GitHub Actions to automate the process.

This setup solves a common deployment problem:

> "When I commit frontend changes to `master`, I want the app to build and automatically deploy to my S3 static website bucket."

## What This Workflow Does

1. Runs whenever code is pushed to the `master` branch
2. Checks out the latest code
3. Sets up Node.js
4. Configures AWS credentials
5. Creates environment variables from GitHub Secrets
6. Installs dependencies
7. Builds the frontend
8. Re-uploads every built file to S3 with caching disabled, uploading `index.html` after assets
9. Removes stale files from the site bucket

## Required GitHub Secrets

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_KEY
AWS_REGION
S3_BUCKET_NAME
API_BASE_URL
TOKEN_BEARER
```

## Workflow File

Create:

```text
.github/workflows/deploy-to-s3.yml
```

```yaml
name: deploy-to-s3

on:
  push:
    branches: [master]

  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Setup Node
        uses: actions/setup-node@v6
        with:
          node-version: 26.13.1
          cache: npm

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Build
        run: |
          cd ${GITHUB_WORKSPACE}
          echo "API_BASE_URL=${{ secrets.API_BASE_URL }}" >> .env
          echo "TOKEN_BEARER=${{ secrets.TOKEN_BEARER }}" >> .env
          npm install
          npm run update_version
          npm run build

      - name: Deploy (no cache)
        run: |
          test -f ./public/index.html

          aws s3 cp ./public "s3://${{ secrets.S3_BUCKET_NAME }}/" \
            --recursive \
            --exclude "index.html" \
            --cache-control "no-cache, no-store, must-revalidate"

          aws s3 cp ./public/index.html "s3://${{ secrets.S3_BUCKET_NAME }}/index.html" \
            --cache-control "no-cache, no-store, must-revalidate"

          aws s3 sync ./public "s3://${{ secrets.S3_BUCKET_NAME }}/" \
            --delete \
            --cache-control "no-cache, no-store, must-revalidate"
```

## How Fresh Deployments Work

This recipe serves the site directly from S3 and prioritizes freshness over bandwidth savings. Every file, including JavaScript, CSS, images, fonts, and HTML, is uploaded with:

```text
Cache-Control: no-cache, no-store, must-revalidate
```

The recursive `aws s3 cp` uploads every asset on each deployment, even if its contents have not changed. This also replaces old cache headers. Using `s3 sync` alone is insufficient for that purpose because unchanged files can be skipped and retain their previous metadata. See the [AWS CLI sync reference](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html).

The workflow checks that `public/index.html` exists before uploading, uploads assets first, and then uploads `index.html`. A final `s3 sync --delete` removes remote files that are absent from the build output. Use a bucket dedicated to this site: this cleanup deletes any other files in that bucket. The cleanup uses the same cache headers in case it uploads any files.

These headers prevent future responses from being stored by compliant browser caches. They cannot revoke files already cached under the previous one-year `immutable` policy. For those existing cached files, use new asset URLs (such as content-hashed filenames) or clear the browser cache once. An already open page must reload to load the new application. This is not an atomic deployment; deleting old assets can also affect tabs still running an earlier version.

## Verify Cache Headers

After deploying, inspect `index.html` and a representative asset, replacing the bucket and asset key below:

```bash
aws s3api head-object --bucket your-bucket-name --key index.html \
  --query CacheControl --output text
aws s3api head-object --bucket your-bucket-name --key assets/app.js \
  --query CacheControl --output text
```

Both should return `no-cache, no-store, must-revalidate`. Deploy the same build again and confirm that the `cp` commands upload unchanged files and the headers remain correct.

In your browser's Network panel, reload the site and inspect the response headers for the document and an asset. Confirm that both have the expected `Cache-Control` value. If migrating from the previous cache policy, clear the old browser cache first; then reload again with the browser's normal caching enabled to verify the new behavior.

## Recommended IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

## Common Issues

### Deployment succeeds but site does not update

Check the response headers for both `index.html` and the affected assets. All should include:

```text
no-cache, no-store, must-revalidate
```

If an asset still comes from an old browser cache, changing its S3 headers does not invalidate that saved response. Change its URL or clear the browser cache once, then follow the verification steps above. If the app uses a service worker, its cache must also be updated by the app; S3 upload headers do not clear service-worker caches.

### Build succeeds but files are missing

Verify the build output folder. This workflow expects:

```text
./public
```

If your framework outputs to `dist` or another directory, update the deploy commands accordingly.

### Access Denied Errors

Confirm:

- IAM permissions are correct
- GitHub Secrets are configured correctly
- AWS Region matches the S3 bucket region

## Summary

This GitHub Actions workflow creates a simple CI/CD pipeline for static frontend applications hosted on Amazon S3. Every push to `master` automatically builds and deploys the application by re-uploading all build files with caching disabled and removing stale files. Previously cached browser responses require new URLs or a one-time cache clear.
