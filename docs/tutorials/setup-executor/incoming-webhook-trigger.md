# Setting up an Incoming Webhook Trigger

Incoming webhook triggers allow external services to invoke actions in your Tailor application via HTTP requests. In this tutorial, we'll create an executor that accepts project updates from external tools and updates project status in your application.

> The webhook URL includes a Tailor-managed secret. For production integrations, treat that URL secret as only one layer. Also validate the HTTP method, content type, timestamp, and a signed payload before mutating TailorDB.

- To follow along with this tutorial, first complete the [SDK Quickstart](../../sdk/quickstart) and the [Data Schema Basics](../manage-data-schema/data-schema-basics) tutorial.

## Tutorial Steps

To create an incoming webhook trigger, you'll need to:

1. Configure the Executor service
2. Create a signing secret in Secret Manager
3. Create the executor with an incoming webhook trigger
4. Deploy the changes
5. Verify the trigger by sending signed webhook requests

### 1. Configure the Executor Service

Update your `tailor.config.ts` to include the executor service:

```typescript
import { defineConfig } from "@tailor-platform/sdk";

export default defineConfig({
  name: "project-management",
  db: {
    "main-db": {
      files: ["db/**/*.ts"],
    },
  },
  executor: {
    files: ["executor/**/*.ts"],
  },
});
```

This configures the SDK to load executor definitions from the `executor/` directory.

### 2. Create a signing secret in Secret Manager

Store a signing secret in Secret Manager. This tutorial uses the `default` vault and the secret name `PROJECT_WEBHOOK_SIGNING_SECRET`.

You can create it with the CLI:

```bash
tailor-sdk secret vault create default
tailor-sdk secret create \
  --vault-name default \
  --name PROJECT_WEBHOOK_SIGNING_SECRET \
  --value "<long-random-secret>"
```

Use a long random value. Do not hardcode it in source code.

### 3. Create the Executor with Incoming Webhook Trigger

Create a new file `executor/webhook-update-project.ts`:

```typescript
import { createExecutor, incomingWebhookTrigger } from "@tailor-platform/sdk";
import { getDB } from "../generated/tailordb";

type ProjectUpdateBody = {
  projectId: string;
  status: string;
  description?: string;
};

type WebhookRequest = {
  body: ProjectUpdateBody;
  headers: Record<string, string>;
  method: "POST" | "GET" | "PUT" | "DELETE";
  rawBody: string;
};

const encoder = new TextEncoder();

function normalizeHeaders(headers: Record<string, string>): Record<string, string> {
  return Object.fromEntries(
    Object.entries(headers).map(([key, value]) => [key.toLowerCase(), value]),
  );
}

function constantTimeEqual(left: string, right: string): boolean {
  if (left.length !== right.length) return false;

  let diff = 0;
  for (let i = 0; i < left.length; i += 1) {
    diff |= left.charCodeAt(i) ^ right.charCodeAt(i);
  }

  return diff === 0;
}

async function hmacSha256Hex(secret: string, value: string): Promise<string> {
  const key = await crypto.subtle.importKey(
    "raw",
    encoder.encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"],
  );

  const signature = await crypto.subtle.sign("HMAC", key, encoder.encode(value));

  return Array.from(new Uint8Array(signature))
    .map((byte) => byte.toString(16).padStart(2, "0"))
    .join("");
}

export default createExecutor({
  name: "webhook-update-project",
  description: "Update project status via signed webhook from external tools",
  trigger: incomingWebhookTrigger<WebhookRequest>(),
  operation: {
    kind: "function",
    body: async ({ body, headers, method, rawBody }) => {
      const db = getDB("main-db");
      const normalizedHeaders = normalizeHeaders(headers);

      if (method !== "POST") {
        throw new Error("Only POST requests are allowed");
      }

      const contentType = normalizedHeaders["content-type"] ?? "";
      if (!contentType.startsWith("application/json")) {
        throw new Error("Content-Type must be application/json");
      }

      const timestamp = normalizedHeaders["x-webhook-timestamp"];
      const signature = normalizedHeaders["x-webhook-signature"];

      if (!timestamp || !signature) {
        throw new Error("Missing required webhook signature headers");
      }

      const timestampMs = Date.parse(timestamp);
      if (!Number.isFinite(timestampMs)) {
        throw new Error("Invalid x-webhook-timestamp header");
      }

      const maxAgeMs = 5 * 60 * 1000;
      if (Math.abs(Date.now() - timestampMs) > maxAgeMs) {
        throw new Error("Webhook request is too old");
      }

      const signingSecret = await tailor.secretmanager.getSecret(
        "default",
        "PROJECT_WEBHOOK_SIGNING_SECRET",
      );

      if (!signingSecret) {
        throw new Error("Missing PROJECT_WEBHOOK_SIGNING_SECRET");
      }

      const expectedSignature = await hmacSha256Hex(
        signingSecret,
        `${timestamp}.${rawBody}`,
      );

      if (!constantTimeEqual(signature, expectedSignature)) {
        throw new Error("Invalid webhook signature");
      }

      if (!body.projectId || !body.status) {
        throw new Error("Missing required fields: projectId and status");
      }

      const project = await db
        .selectFrom("Project")
        .selectAll()
        .where("id", "=", body.projectId)
        .executeTakeFirst();

      if (!project) {
        throw new Error(`Project not found: ${body.projectId}`);
      }

      const updatedProject = await db
        .updateTable("Project")
        .set({
          status: body.status,
          description: body.description ?? project.description,
        })
        .where("id", "=", body.projectId)
        .returningAll()
        .executeTakeFirst();

      return {
        success: true,
        message: `Project ${updatedProject?.name} updated to ${body.status}`,
        projectId: updatedProject?.id,
      };
    },
  },
});
```
What this validation does
It keeps Tailor's built-in webhook URL secret in place
It only accepts POST
It only accepts application/json
It requires a timestamp header and rejects stale requests
It verifies an HMAC signature against the exact rawBody
It validates business data before updating TailorDB

If your sender already provides an official signing scheme, validate that scheme in the executor function. The important point is to verify the signature before mutating TailorDB.

### 4. Deploy the Changes

Deploy your application:

```bash
npm run deploy -- --workspace-id <your-workspace-id>
```

The SDK will create the incoming webhook endpoint for your executor.

### 5. Verify the Trigger
**Step 1: Get the webhook URL**

Open the Console
 and navigate to your workspace. Select Executors and click on `Webhook-update-project` to view the webhook URL.

The webhook URL format is:
```
https://api.tailor.tech/v1/executor/workspaces/{WORKSPACE_ID}/executors/webhook-update-project/invokeIncomingWebhook/{WEBHOOK_SECRET}
```

Alternatively, use the Tailor CLI:
```bash
npx tailor-sdk executor webhook list
```
**Step 2: Create a test project**

First, create a project to update. In the GraphQL Playground:

```graphql
mutation {
  createProject(
    input: {
      name: "API Integration Test"
      description: "Testing webhook integration"
      status: "planning"
      createdAt: "2026-02-09T10:00:00Z"
      updatedAt: "2026-02-09T10:00:00Z"
    }
  ) {
    id
    name
    status
  }
}
```

Note the project ID returned.

**Step 3: Send a signed webhook request**

Set these shell variables first:

```bash
WEBHOOK_URL="https://api.tailor.tech/v1/executor/workspaces/{WORKSPACE_ID}/executors/webhook-update-project/invokeIncomingWebhook/{WEBHOOK_SECRET}"
WEBHOOK_SIGNING_SECRET="<the same value stored in Secret Manager>"
PROJECT_ID="<your-project-id>"
TIMESTAMP="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
BODY="{\"projectId\":\"${PROJECT_ID}\",\"status\":\"active\",\"description\":\"Updated via signed webhook from external tool\"}"
SIGNATURE="$(node -e 'const crypto = require("crypto"); const secret = process.argv[1]; const timestamp = process.argv[2]; const body = process.argv[3]; process.stdout.write(crypto.createHmac("sha256", secret).update(`${timestamp}.${body}`).digest("hex"));' "$WEBHOOK_SIGNING_SECRET" "$TIMESTAMP" "$BODY")"
```

Send the request:
```bash
curl -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Timestamp: $TIMESTAMP" \
  -H "X-Webhook-Signature: $SIGNATURE" \
  -d "$BODY"
```

Expected response:

```json
{
  "success": true,
  "message": "Project API Integration Test updated to active",
  "projectId": "<your-project-id>"
}
```

**Step 4: Verify the update**

Open the GraphQL Playground and query the project:

```graphql
query {
  project(id: "<your-project-id>") {
    id
    name
    status
    description
    updatedAt
  }
}
```

You should see the updated status and description.

**Step 5: View executor logs**

In the Console, select the `Jobs` tab under your executor to see the execution history. Each webhook call creates a job entry with:

- Request payload
- Execution status
- Response data
- Any errors encountered

Avoid logging the signing secret or the computed signature.

## Optional compatibility note

Incoming webhook triggers can also parse application/x-www-form-urlencoded, but this tutorial recommends POST plus application/json as the default path. If a third-party sender requires form-urlencoded payloads, keep the same signature and timestamp checks and compute the signature from the exact rawBody.

## Security Considerations
The webhook URL secret is one authentication layer, not the only layer.
Validate the request before touching TailorDB:
allow only the expected HTTP method
allow only the expected content type
require signature headers
reject stale timestamps
verify the signature against the exact rawBody
Keep signing secrets in Secret Manager, not in source code.
Return generic error messages to callers and keep sensitive details out of logs.
Prefer idempotent webhook handlers when the sender may retry delivery.

## Next Steps

Learn more about executors:

- [Executor Service](../../sdk/services/executor) - Complete executor documentation
- [Trigger Types](../../sdk/services/executor#trigger-types) - All available trigger types
- [Operation Types](../../sdk/services/executor#operation-types) - Different operation kinds
- [Event-based Triggers](event-based-trigger) - Create database event-driven executors
