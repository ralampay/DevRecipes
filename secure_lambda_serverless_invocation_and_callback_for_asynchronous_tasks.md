# Secure Lambda Serverless Invocation and Callback for Asynchronous Tasks

## Overview

Modern web applications often need to perform work that does not fit cleanly inside a normal request-response cycle. Examples include parsing uploaded files, generating reports, processing media, importing large datasets, running document analysis, extracting metadata, or performing any task that requires more memory, more execution time, or more specialized dependencies than the main backend should carry. Running this work directly in the backend can make API requests slow, increase memory pressure, complicate deployments, and make failures harder to isolate.

The general solution is to keep the backend responsible for user authentication, authorization, request validation, job state, and final persistence, while delegating the expensive work to a serverless function such as AWS Lambda. The backend creates a durable job record, stores or references the input data, invokes the function asynchronously, and immediately returns control to the client. The client can then poll the backend or subscribe to updates while the work continues independently.

This guide describes a secure invocation-and-callback pattern for asynchronous tasks. The backend sends a small job payload to Lambda, usually containing a job id and an object-storage key. Lambda performs the work, writes large output back to object storage, and sends a signed callback to the backend when the job succeeds or fails. The callback does not use a user session or bearer token because it is a machine-to-machine event. Instead, Lambda signs the exact JSON callback body with a shared secret using HMAC-SHA256, and the backend verifies that signature before accepting the status update.

This pattern solves three common problems at once. First, it keeps long-running or resource-heavy work out of the web request path. Second, it avoids sending large files or generated output through API payloads by using object storage as the data plane. Third, it gives the backend a secure way to accept completion events from an external worker without exposing a callback endpoint that arbitrary callers can spoof.

## Architecture

The architecture has two separate flows: a **control plane** and a **data plane**. Separating these concerns is the key design choice. The control plane carries small messages that describe what should happen and what happened. The data plane carries large input and output artifacts through object storage.

The **control plane** starts in the backend. A user or client submits a request to create a job, such as uploading a file or asking for a document to be processed. The backend authenticates the user, validates permissions, creates a database record, and stores the job in an initial state such as `pending`. The backend then invokes Lambda with a compact payload that contains the job id and any references Lambda needs to find the input data. In AWS, the backend can invoke Lambda asynchronously with an event-style invocation so the backend is not waiting for the job to complete.

The **data plane** usually uses S3 or another durable object store. The backend writes the raw input to an input bucket or receives an existing object key from the client. Lambda reads that input object, performs the task, and writes generated output to an output bucket. Instead of returning a large result through the callback, Lambda returns only metadata such as an output object key, status, and message. The backend stores that output key on the job record and can read the output object later when the client requests it.

A typical production flow looks like this:

```text
Client
  -> Backend API: create job or upload input
  -> Backend stores input in object storage
  -> Backend creates job record with status=pending
  -> Backend invokes Lambda asynchronously with { job_id, input_key }
  -> Backend marks job as processing
  -> Lambda downloads input from object storage
  -> Lambda performs the expensive task
  -> Lambda uploads output to object storage
  -> Lambda signs callback body with a shared secret
  -> Lambda sends callback to Backend API
  -> Backend verifies HTTPS and HMAC signature
  -> Backend stores output_key and terminal status
```

The callback endpoint should be treated as a dedicated machine endpoint. It should not depend on the end user's authentication token, because the user is no longer involved when Lambda completes the task. Instead, the endpoint should require a cryptographic signature. Lambda and the backend both know the same secret value. Lambda computes:

```text
signature = HMAC_SHA256(SHARED_CALLBACK_SECRET, raw_json_body)
```

Lambda sends that signature in a header such as:

```http
X-Lambda-Signature: <hex_hmac_sha256_signature>
```

The backend reads the raw request body exactly as received, computes the expected HMAC using the same secret, and compares the expected value with the provided signature using a constant-time comparison function. If the signature is missing, invalid, or computed over different bytes, the backend rejects the callback.

The main components are:

- **Backend job endpoint**: Authenticates the user, validates the request, creates the job record, stores or references input data, and starts the asynchronous worker.
- **Object storage input bucket**: Holds raw input artifacts that are too large or inconvenient to pass directly through Lambda payloads.
- **Lambda worker**: Downloads the input, performs the task, writes the output, and sends the completion callback.
- **Object storage output bucket**: Holds generated output artifacts such as JSON, reports, thumbnails, transformed files, or extraction results.
- **Backend callback endpoint**: Verifies the HMAC signature, validates the payload, updates job state, and stores the output reference.
- **Shared callback secret**: A long random value available to both Lambda and the backend through secure environment configuration or a secrets manager.

## Methodology

Start by designing the backend as the source of truth for the job lifecycle. The backend should create a durable job record before invoking Lambda. This record gives the system a stable identifier for logs, retries, status polling, callbacks, and user-visible history. Common statuses are `pending`, `processing`, `succeeded`, and `failed`, though the exact names should match the domain. The important rule is that a job should exist before external work begins, so the callback has something authoritative to update.

Keep the Lambda invocation payload small and reference-based. A good payload usually contains a job id, an input object key, and perhaps a few simple options. It should not contain the uploaded file itself or the full output destination payload. For example:

```json
{
  "job_id": "9b6f5a8e-3f4a-4c2e-9a1f-2c7a1c5a2c10",
  "input_key": "uploads/2026/06/20/source-file.bin",
  "options": {
    "mode": "standard"
  }
}
```

Use object storage as the handoff mechanism for large data. The backend and Lambda need a shared understanding of where inputs and outputs live. In AWS, this usually means one bucket or prefix for inputs and another bucket or prefix for generated outputs. Lambda should have IAM permissions only for the object paths it needs. The backend should store output keys, not large generated payloads, in its database. This keeps database rows small and allows large artifacts to be fetched, streamed, cached, lifecycle-managed, or deleted independently.

Invoke Lambda asynchronously when the backend does not need the result immediately. With AWS Lambda, this means using an event invocation rather than a request-response invocation. The backend should treat successful invocation as "the job was accepted for processing," not "the job succeeded." After invocation, the backend updates the job status to `processing` and returns a response to the client. The client can poll a status endpoint, refresh a job list, or receive updates through websockets or notifications.

Make Lambda responsible for reporting both success and failure. On success, Lambda should upload the generated output and call the backend with a payload that includes a success status and the output key. On failure, Lambda should still call the backend with a failed status and a useful message, as long as it has enough context to identify the job. This prevents jobs from staying stuck in `processing` when parsing, transformation, or output upload fails.

Use a signed callback instead of trusting the callback endpoint by obscurity. A callback URL is not secret enough to protect state changes. Anyone who discovers the URL could attempt to mark arbitrary jobs as complete unless the backend verifies that the request came from a trusted worker. HMAC signing is a simple and reliable mechanism for this. The worker signs the exact body it sends, and the backend verifies that exact body before parsing or applying the update.

A success callback body can be shaped like this:

```json
{
  "status": "success",
  "output_key": "outputs/9b6f5a8e-3f4a-4c2e-9a1f-2c7a1c5a2c10.json",
  "message": "Task completed successfully"
}
```

A failure callback body can be shaped like this:

```json
{
  "status": "failed",
  "output_key": null,
  "message": "Input file could not be parsed"
}
```

The signature must be computed over the raw JSON string exactly as it is sent on the wire. This detail matters. If Lambda signs a compact JSON string but the backend verifies a pretty-printed or reserialized object, the signature will not match. The backend should read the raw request body, compute the HMAC, compare the signature, and only then trust the parsed payload enough to update state.

Require HTTPS for deployed callbacks. HMAC proves that the body was signed by a party that knows the secret, but HTTPS still protects the request in transit and prevents intermediaries from observing callback payloads or replaying useful metadata. For local development, it is reasonable to allow HTTP callbacks behind an explicit development-only flag, but that exception should still require a valid signature and should never be enabled in production.

Make callback updates idempotent. Serverless systems and HTTP callbacks can retry after timeouts, partial failures, or ambiguous network errors. If Lambda sends the same success callback twice, the backend should not fail the second request or corrupt the job. A practical rule is to accept repeated terminal updates when the existing status and output key match the incoming values. If the incoming callback conflicts with an existing terminal state, the backend should reject it or log it for investigation.

Protect secrets as deployment configuration, not source code. The shared callback secret should be generated with a cryptographically strong random generator, stored in a secrets manager or secure environment configuration, and injected into both the backend and Lambda at runtime. Use a different secret per environment. Rotate the secret if it is exposed. Do not log it, commit it, or include it in client-side configuration.

## Local Testing Setup

A local setup should preserve the same architecture while replacing the deployed Lambda with a local serverless runtime. With AWS SAM, the backend can run on the developer machine, the Lambda function can run inside a local container, and object storage can remain in AWS. This allows developers to test the complete flow without deploying every code change.

The local flow looks like this:

```text
Local client or curl
  -> Local backend API
  -> Local SAM Lambda container
  -> Object storage
  -> Local backend callback endpoint
```

The backend needs environment variables for object storage, Lambda invocation, local serverless routing, and callback verification. Generic names might look like this:

```env
APP_ENV=development
CLOUD_REGION=ap-southeast-1
INPUT_STORAGE_BUCKET=<input-artifact-bucket>
OUTPUT_STORAGE_BUCKET=<output-artifact-bucket>
ASYNC_WORKER_FUNCTION_NAME=<deployed-worker-function-name>
LOCAL_WORKER_URL=http://127.0.0.1:8080/process
ALLOW_INSECURE_LOCAL_CALLBACKS=true
LAMBDA_CALLBACK_SECRET=<shared-callback-secret>
```

The local Lambda or SAM environment needs matching storage and callback settings:

```env
CLOUD_REGION=ap-southeast-1
INPUT_STORAGE_BUCKET=<input-artifact-bucket>
OUTPUT_STORAGE_BUCKET=<output-artifact-bucket>
BACKEND_API_BASE_URL=http://localhost:3000
LOCAL_BACKEND_HOST=host.docker.internal
LAMBDA_CALLBACK_SECRET=<shared-callback-secret>
```

Generate a strong shared secret for development:

```bash
openssl rand -hex 64
```

Use the exact same value for the backend and the local Lambda runtime. This is what lets the local callback exercise the same HMAC verification path used in production.

### Allow HTTP for Local Callbacks

Local callback testing often uses plain HTTP because the backend is running on a developer machine, such as `http://localhost:3000` or `http://host.docker.internal:3000`. Some frameworks and middleware reject or redirect non-HTTPS callback requests by default. For example, a Rails application with SSL enforcement enabled can redirect the local Lambda callback from HTTP to HTTPS, which prevents the worker from reaching the callback endpoint correctly.

Allow HTTP callbacks only in the local or development environment, and keep the HMAC signature verification enabled. The local exception should be controlled by an explicit setting such as `ALLOW_INSECURE_LOCAL_CALLBACKS=true`, not by disabling callback security entirely. In production and shared deployed environments, callback URLs should remain HTTPS-only.

For Rails-style applications, this usually means making SSL enforcement environment-aware. The exact code depends on the application, but the policy should be:

```ruby
# config/environments/development.rb
config.force_ssl = false
```

If the application has custom callback validation, it should also allow the `http` scheme only when the development-only flag is enabled:

```ruby
allow_http_callback = Rails.env.development? && ENV["ALLOW_INSECURE_LOCAL_CALLBACKS"] == "true"
callback_uri = URI.parse(callback_url)

raise "Callback URL must use HTTPS" unless callback_uri.scheme == "https" || allow_http_callback
```

Run the backend on a host and port that the Lambda container can reach. For many frameworks, binding to all interfaces is necessary because the callback originates from a Docker container rather than from the host process itself:

```bash
backend-server --host 0.0.0.0 --port 3000
```

Run the local serverless API with SAM or an equivalent tool:

```bash
sam build
sam local start-api -p 8080
```

When the backend is configured with `LOCAL_WORKER_URL=http://127.0.0.1:8080/process`, it should call the local serverless endpoint instead of invoking the deployed Lambda function. This makes the full application flow testable: create a job through the backend, let the local Lambda process it, write output to object storage, and send a signed callback back to the local backend.

One local networking issue is worth calling out. Inside a Docker container, `localhost` usually refers to the container itself, not the developer machine. If Lambda needs to call the backend at `http://localhost:3000`, the local runtime may need to rewrite the callback host to `host.docker.internal` or another host gateway address. A common local callback URL from the container is:

```text
http://host.docker.internal:3000/jobs/{job_id}/callback
```

For manual testing, call the local serverless endpoint with a job id and input key that already exist:

```bash
curl -X POST http://127.0.0.1:8080/process \
  -H "Content-Type: application/json" \
  -d '{
    "job_id": "9b6f5a8e-3f4a-4c2e-9a1f-2c7a1c5a2c10",
    "input_key": "uploads/example-input.bin"
  }'
```

The expected local result is the same as production. The worker receives the job request, downloads the input artifact, performs the task, uploads the output artifact, signs the callback, and sends the callback to the backend. The backend verifies the signature, updates the job status, and stores the output key.

If local testing fails, check the basics in this order:

- The backend and Lambda use the same `LAMBDA_CALLBACK_SECRET`.
- The backend input bucket configuration matches the worker input bucket configuration.
- The backend output bucket configuration matches the worker output bucket configuration.
- The input object key exists and the worker has permission to read it.
- The worker has permission to write the output object.
- The callback base URL is reachable from inside the Lambda container.
- Local HTTP callbacks are explicitly allowed only in development.
- Production callback URLs use HTTPS.

## Solution Summary

This pattern provides a reusable way to run asynchronous serverless work without weakening the backend's authority over application state. The backend remains the system of record. It authenticates users, creates jobs, controls permissions, and stores final status. Lambda remains a focused worker. It handles the resource-heavy task, writes output to object storage, and reports completion.

The most important design choice is to avoid moving large artifacts through control messages. Invocation payloads and callback bodies should stay small. Files, generated JSON, reports, images, or other large outputs should move through object storage. The database should store durable references to those artifacts, such as object keys, rather than the artifacts themselves.

The most important security choice is to sign callbacks with a shared secret. A callback endpoint often cannot use normal user authentication because it is called by infrastructure, not by a user. HMAC-SHA256 over the exact raw request body gives the backend a concrete way to verify that the callback was produced by a trusted worker and that the body was not changed after signing.

The production-ready version of this solution should include HTTPS-only callbacks, constant-time signature comparison, environment-specific secrets, least-privilege storage permissions, idempotent callback handling, clear job states, and logs that include job ids but never secrets. With those pieces in place, the backend can safely delegate heavy work to Lambda while still keeping control over data, status, and user-visible results.
