# Secure Lambda Serverless Invocation and Callback for Asynchronous Tasks

## Overview

Some application tasks are too expensive, slow, or specialized to run inside the main web backend. File parsing, media processing, report generation, imports, and machine-learning jobs often need more memory, longer execution time, isolated dependencies, or elastic compute. The backend still needs to own user authentication, permissions, request state, and the final database record, but the heavy work can be delegated to a serverless worker.

This pattern solves that problem by splitting the workflow into two directions:

1. The backend invokes a Lambda function asynchronously with a small job payload.
2. The Lambda function performs the work, writes large output to shared object storage, then calls the backend through a signed callback.

In the Envelop codebase, this pattern is used for IFC file parsing. Rails accepts an authenticated file upload, stores the raw file in S3 through Active Storage, invokes the IFC parser Lambda with the database record id and S3 object key, and moves the record to `processing`. The Lambda downloads the raw IFC file from S3, parses it, uploads parsed JSON to a parser-output S3 bucket, then calls Rails at:

```http
PUT /ifc_files/:id/serverless_update
```

The callback is not authenticated with a user bearer token because it is a machine-to-machine completion event. Instead, Lambda signs the exact JSON body with `LAMBDA_CALLBACK_SECRET` using HMAC-SHA256 and sends the signature in `X-Lambda-Signature`. Rails independently computes the expected signature over the raw request body and compares the values securely before updating the record.

## Architecture

The architecture separates the control plane from the data plane.

Control plane:

- The backend creates and owns the job record.
- The backend invokes Lambda with a compact payload: the job id and source S3 object key.
- Lambda calls the backend callback endpoint when the job reaches a terminal state.
- The backend validates the callback signature and persists the final status.

Data plane:

- The raw upload is stored in an S3 bucket that both Rails and Lambda can access.
- Lambda writes generated output to a parser-output S3 bucket.
- The backend stores only the output object key, then reads the parsed JSON from S3 when needed.

The Envelop flow is:

```text
Browser
  -> Rails API: POST /ifc_files
  -> Rails Active Storage uploads raw IFC to S3
  -> Rails invokes Lambda with { id, ifc_file_key }
  -> Rails marks IfcFile as processing
  -> Lambda downloads ifc_file_key from AWS_IFC_FILES_BUCKET
  -> Lambda parses the IFC file
  -> Lambda uploads parsed JSON to S3_BUCKET
  -> Lambda signs callback body with LAMBDA_CALLBACK_SECRET
  -> Lambda PUTs /ifc_files/:id/serverless_update
  -> Rails verifies HTTPS and HMAC signature
  -> Rails stores data_key and maps success to done or failed to failed
```

The main moving parts are:

- Rails `IfcFiles::Create`: creates the upload record and starts parsing in development or production.
- Rails `IfcFiles::InvokeAwsLambdaIfcParser`: sends `{ id, ifc_file_key }` to Lambda. In AWS it uses asynchronous `InvocationType=Event`; locally it can POST to SAM using `IFC_PARSER_LOCAL_URL`.
- Lambda handler: validates the parser request, downloads from S3, parses the file, uploads parsed JSON, and notifies Rails.
- Lambda `NotifyBackend`: builds the callback URL, signs the raw JSON body, and sends the callback.
- Rails `LambdaCallbacks::VerifySignature`: requires HTTPS, requires `LAMBDA_CALLBACK_SECRET`, computes `HMAC_SHA256(secret, raw_request_body)`, and uses secure comparison.
- Rails `IfcFiles::ServerlessUpdate`: updates `data_key` and terminal status, while allowing idempotent retry callbacks.

The shared secret is configured on both sides:

- Rails: `LAMBDA_CALLBACK_SECRET`
- SAM/Lambda parameter: `LambdaCallbackSecret`
- Lambda runtime variable: `LAMBDA_CALLBACK_SECRET`

The secret should be long, random, environment-specific, and never committed.

## Methodology

1. Create the job record before invoking Lambda.

   The backend should persist a durable record before starting external work. This creates a stable id for retries, polling, logs, and callbacks. In Envelop, new IFC records start as `pending`, then move to `processing` after the parser is invoked.

2. Pass only a small invocation payload.

   Lambda receives the database record id and source object key:

   ```json
   {
     "id": "ifc-file-id",
     "ifc_file_key": "active-storage-s3-object-key"
   }
   ```

   Large files and large parser outputs do not move through Lambda invocation or callback bodies. They move through S3.

3. Use S3 as the shared data plane.

   The backend writes the raw upload to an S3 bucket. Lambda reads that bucket through `AWS_IFC_FILES_BUCKET`. Lambda writes parsed output to `S3_BUCKET`. The backend reads parsed output later through its matching `AWS_IFC_FILES_DATA_BUCKET`.

   This avoids request timeouts, large HTTP bodies, and duplicated file transport logic.

4. Invoke Lambda asynchronously in deployed environments.

   Rails invokes the Lambda function by name using `InvocationType=Event`. That lets the API return quickly while the job continues out of process. The backend state communicates progress to the frontend through statuses such as `pending`, `processing`, `done`, and `failed`.

5. Sign the callback with HMAC-SHA256.

   Lambda serializes the callback JSON, signs that exact string, and sends both body and signature:

   ```text
   signature = HMAC_SHA256(LAMBDA_CALLBACK_SECRET, raw_json_body)
   ```

   Example success payload:

   ```json
   {
     "data_key": "generated-parser-json-key",
     "status": "success",
     "message": "IFC file parsed successfully"
   }
   ```

   Example failure payload:

   ```json
   {
     "data_key": null,
     "status": "failed",
     "message": "error details"
   }
   ```

6. Verify the raw body, not a parsed or reformatted body.

   The backend must compute the expected HMAC over the exact raw request body bytes. If it parses and reserializes JSON before verification, harmless formatting changes can break signatures and malicious transformations can become harder to reason about.

7. Require HTTPS for deployed callbacks.

   Rails accepts callbacks only over HTTPS or when `X-Forwarded-Proto` says `https`. Envelop has a development-only escape hatch, `ALLOW_INSECURE_LAMBBDA_CALLBACKS=true`, so local SAM can call local Rails over HTTP while still requiring a valid HMAC signature.

8. Make callbacks idempotent.

   Serverless callbacks can be retried after network failures or ambiguous responses. The backend should safely accept a repeated terminal update when the status and output key match the existing record. Envelop does this for repeated `done` or `failed` updates with the same `data_key`.

9. Keep authorization boundaries clear.

   User-facing endpoints still require normal authentication and ownership checks. The callback endpoint intentionally skips user bearer authentication because Lambda is not acting as a user. Its authority comes from possession of the shared secret and a valid HMAC signature.

## Local Testing Setup

Local testing keeps Rails and SAM on the developer machine while S3 remains in AWS:

```text
Local browser or curl
  -> local Rails API
  -> local SAM Lambda container
  -> AWS S3
  -> local Rails callback
```

Backend environment:

```env
RAILS_ENV=development
AWS_REGION=ap-southeast-1
AWS_BUCKET=<active-storage-ifc-upload-bucket>
AWS_IFC_FILES_DATA_BUCKET=<parser-output-bucket>
AWS_LAMBDA_IFC_PARSER_NAME=<deployed-lambda-function-name>
IFC_PARSER_LOCAL_URL=http://127.0.0.1:8080/parse-ifc
ALLOW_INSECURE_LAMBBDA_CALLBACKS=true
LAMBDA_CALLBACK_SECRET=<shared-callback-secret>
```

Serverless environment:

```env
AWS_REGION=ap-southeast-1
AWS_IFC_FILES_BUCKET=<active-storage-ifc-upload-bucket>
S3_BUCKET=<parser-output-bucket>
ENVELOP_API_BASE_URL=http://localhost:3000
LOCAL_BACKEND_HOST=host.docker.internal
LAMBDA_CALLBACK_SECRET=<shared-callback-secret>
```

Generate the shared secret:

```bash
openssl rand -hex 64
```

Use the exact same value in Rails and Lambda/SAM.

Run Rails so the SAM container can reach it:

```bash
cd envelop-api
bundle install
bundle exec rails db:create db:migrate
bundle exec rails server -b 0.0.0.0 -p 3000
```

Run SAM locally:

```bash
cd envelop-serverless
sam build
sam local start-api -p 8080
```

With `IFC_PARSER_LOCAL_URL=http://127.0.0.1:8080/parse-ifc`, Rails posts parser jobs to the local SAM API instead of invoking the deployed Lambda function. The Lambda container cannot call `localhost:3000` to reach Rails because `localhost` points to the container itself. The serverless code rewrites local callback URLs to `LOCAL_BACKEND_HOST`, usually `host.docker.internal`, so the callback reaches Rails at `http://host.docker.internal:3000`.

For manual invocation of the local parser:

```bash
curl -X POST http://127.0.0.1:8080/parse-ifc \
  -H "Content-Type: application/json" \
  -d '{
    "id": "ifc-file-id",
    "ifc_file_key": "active-storage-blob-key"
  }'
```

Expected result:

1. SAM receives the parser request.
2. Lambda downloads the raw file from `AWS_IFC_FILES_BUCKET`.
3. Lambda writes parsed JSON to `S3_BUCKET`.
4. Lambda calls Rails at `/ifc_files/:id/serverless_update`.
5. Rails verifies the HMAC signature.
6. Rails stores `data_key` and updates the status to `done`, or stores `failed` on parser failure.

Troubleshooting checks:

- `AWS_BUCKET` in Rails must match `AWS_IFC_FILES_BUCKET` in Lambda.
- `AWS_IFC_FILES_DATA_BUCKET` in Rails must match `S3_BUCKET` in Lambda.
- `LAMBDA_CALLBACK_SECRET` must be identical on both sides.
- Local Rails must be reachable from Docker, usually through `host.docker.internal`.
- Local HTTP callbacks require `ALLOW_INSECURE_LAMBBDA_CALLBACKS=true` in Rails development.
- Deployed callbacks should use an HTTPS `ENVELOP_API_BASE_URL`.

## Solution Summary

This solution provides a secure, practical pattern for asynchronous serverless work:

- The backend owns authentication, authorization, job records, and final state.
- Lambda owns expensive or specialized processing.
- S3 carries large input and output data between systems.
- The Lambda invocation payload stays small and durable.
- The completion callback is protected with an HMAC-SHA256 signature over the exact raw JSON body.
- HTTPS is required outside local development.
- The backend stores an output key instead of large generated data.
- Idempotent callbacks make retries safe.

The result is a clean division of responsibility. The web API remains responsive and authoritative, Lambda can scale and use isolated dependencies for heavy work, and the callback can be exposed without user credentials because it is authenticated cryptographically through a shared secret.
