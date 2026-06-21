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
8. Uploads the built files to S3
9. Applies proper cache rules for frontend assets and `index.html`

## Required GitHub Secrets

```text
AWS_ACCESS_KEY
AWS_SECRET_KEY
AWS_REGION
S3_BUCKET_NAME
API_BASE_URL
TOKEN_BEARER
CURRENT_USER
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
        uses: actions/checkout@v6

      - name: Setup Node
        uses: actions/setup-node@v6
        with:
          node-version: 24.13.0
          cache: npm

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Build
        run: |
          cd ${GITHUB_WORKSPACE}
          echo "API_BASE_URL=${{ secrets.API_BASE_URL }}" >> .env
          echo "TOKEN_BEARER=${{ secrets.TOKEN_BEARER }}" >> .env
          echo "CURRENT_USER=${{ secrets.CURRENT_USER }}" >> .env
          npm install
          npm run update_version
          npm run build

      - name: Deploy (no cache)
        run: |
          aws s3 sync ./public s3://${{ secrets.S3_BUCKET_NAME }} \
            --exclude "index.html" \
            --delete \
            --cache-control "max-age=31536000, immutable"

          aws s3 cp ./public/index.html s3://${{ secrets.S3_BUCKET_NAME }}/index.html \
            --cache-control "no-cache, no-store, must-revalidate" \
            --metadata-directive REPLACE
```

## Why Cache Static Assets Differently?

Static assets such as:

- JavaScript bundles
- CSS files
- Images
- Fonts

can be cached for a long time:

```bash
--cache-control "max-age=31536000, immutable"
```

This improves performance and reduces bandwidth usage.

However, `index.html` should never be cached aggressively because it references the latest asset versions. It is uploaded separately with:

```bash
--cache-control "no-cache, no-store, must-revalidate"
```

This ensures users always receive the latest application version after deployment.

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

Usually caused by browser or CDN caching. Ensure `index.html` is uploaded with:

```text
no-cache, no-store, must-revalidate
```

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

This GitHub Actions workflow creates a simple CI/CD pipeline for static frontend applications hosted on Amazon S3. Every push to `master` automatically builds and deploys the application while applying optimal cache-control headers for both assets and HTML files.
