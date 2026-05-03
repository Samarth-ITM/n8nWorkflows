Manage Strapi CMS v5 content types via webhook using HTTP requests

https://n8nworkflows.xyz/workflows/manage-strapi-cms-v5-content-types-via-webhook-using-http-requests-14471


# Manage Strapi CMS v5 content types via webhook using HTTP requests

# 1. Workflow Overview

This workflow exposes a single **POST webhook** that lets external callers manage **Strapi CMS v5** content entries through direct REST API calls. It supports three operations determined by the incoming payload:

- **create** a new entry
- **update** an existing entry by `documentId`
- **get_all** entries for a content type, optionally filtered by status and pagination

The workflow is designed as a lightweight Strapi v5 integration pattern for n8n because the official Strapi node does not support Strapi v5. Instead, it uses **HTTP Request** nodes authenticated with a **Strapi API Token** credential.

## 1.1 Input Reception

The workflow starts with a webhook that accepts a JSON body. The request body is expected to contain routing and API parameters such as:

- `action_type`
- `strapi_base_url`
- `content_type_plural`
- `status`
- `documentId`
- `data`
- `page_size`
- `page_number`

## 1.2 Action Routing

A **Switch** node inspects `body.action_type` and routes execution to one of three branches:

- `create`
- `update`
- `get_all`

## 1.3 Strapi API Operations

Each branch uses a dedicated **HTTP Request** node to call the Strapi v5 REST API:

- POST for create
- PUT for update
- GET for retrieval

All requests use the same Strapi API token credential and construct URLs dynamically from the webhook payload.

## 1.4 Webhook Response Handling

The selected Strapi response is sent back to the original webhook caller using a **Respond to Webhook** node configured to return all incoming items.

## 1.5 Execution Failure Labeling

After the HTTP response has already been returned to the client, the workflow checks whether Strapi returned an `error` object. If so, execution is marked as failed with a **Stop and Error** node. This is useful for n8n execution visibility, logs, and monitoring, even though the client has already received the Strapi payload.

---

# 2. Block-by-Block Analysis

## Block 1 — Webhook Entry

### Overview
This block receives incoming POST requests and hands the payload to the routing logic. It is the only entry point in the workflow.

### Nodes Involved
- Webhook

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry node that exposes an HTTP endpoint to trigger the workflow.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `strapi-v5-content`
  - Response mode: `responseNode`, meaning the workflow must explicitly answer through a Respond to Webhook node
  - `alwaysOutputData: true`, which helps ensure downstream nodes receive an item even in edge scenarios
- **Key expressions or variables used:**  
  None internally, but downstream nodes rely on:
  - `$json.body.action_type`
  - `$json.body.strapi_base_url`
  - `$json.body.content_type_plural`
  - `$json.body.status`
  - `$json.body.documentId`
  - `$json.body.data`
  - `$json.body.page_size`
  - `$json.body.page_number`
- **Input and output connections:**
  - Input: none
  - Output: Action detection
- **Version-specific requirements:**
  - Uses `typeVersion: 2.1`
  - `responseMode: responseNode` requires a matching **Respond to Webhook** node later in the flow
- **Edge cases or potential failure types:**
  - Webhook path collision if another active workflow uses the same path
  - No authentication is configured by default, so the endpoint is public unless secured manually
  - If callers send malformed JSON or omit required fields, downstream expressions may generate invalid requests
- **Sub-workflow reference:**  
  None

---

## Block 2 — Action Routing

### Overview
This block determines which Strapi operation should run based on `action_type` in the incoming request body. It forms the main control structure of the workflow.

### Nodes Involved
- Action detection

### Node Details

#### Action detection
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branches execution depending on the requested operation.
- **Configuration choices:**
  - Evaluates `$json.body.action_type`
  - Case-sensitive string comparison
  - Three explicit routes:
    - `create`
    - `update`
    - `get_all`
- **Key expressions or variables used:**
  - `={{ $json.body.action_type }}`
- **Input and output connections:**
  - Input: Webhook
  - Outputs:
    - Output 0 → Create a new Insight
    - Output 1 → Update an insight
    - Output 2 → Get all Insights
- **Version-specific requirements:**
  - Uses `typeVersion: 3.4`
  - Conditions are built with Switch rules version 3 syntax
- **Edge cases or potential failure types:**
  - If `action_type` is missing or has an unsupported value, no branch will run
  - In that case, because the webhook expects a response node, execution may complete without a proper response unless additional fallback handling is added
  - Case sensitivity means `Create` or `GET_ALL` will not match
- **Sub-workflow reference:**  
  None

---

## Block 3 — Create Content Entry

### Overview
This branch creates a new Strapi content entry for the content type provided by the caller. The body payload is wrapped into the Strapi v5 expected `{ data: ... }` structure.

### Nodes Involved
- Create a new Insight

### Node Details

#### Create a new Insight
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the target Strapi REST endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL dynamically built as:  
    `{{$json.body.strapi_base_url}}/api/{{$json.body.content_type_plural}}?status={{$json.body.status}}`
  - Sends JSON body
  - JSON body structure: `{ data: $('Webhook').item.json.body.data }`
  - Adds header `Content-Type: application/json`
  - Uses predefined credential type `strapiTokenApi`
  - `neverError: true` is enabled so non-2xx responses are returned as data rather than immediately failing the node
- **Key expressions or variables used:**
  - URL:
    - `={{ $json.body.strapi_base_url }}/api/{{ $json.body.content_type_plural }}?status={{ $json.body.status }}`
  - Body:
    - `={{ { data: $('Webhook').item.json.body.data } }}`
- **Input and output connections:**
  - Input: Action detection (create branch)
  - Output: Respond to Webhook
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`
  - Requires a configured **Strapi Token API** credential in n8n
- **Edge cases or potential failure types:**
  - Missing or malformed `strapi_base_url`
  - Invalid `content_type_plural`
  - Missing `data`
  - Token lacks create permission for the content type
  - `status` may be empty or invalid; the query string still gets appended
  - If `strapi_base_url` includes a trailing slash, the URL may contain a double slash before `/api`
  - Strapi validation errors are returned in the response body due to `neverError: true`
- **Sub-workflow reference:**  
  None

---

## Block 4 — Update Content Entry

### Overview
This branch updates an existing Strapi entry using its `documentId`. Only the fields included in `data` are sent to Strapi.

### Nodes Involved
- Update an insight

### Node Details

#### Update an insight
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a PUT request to update a specific Strapi document.
- **Configuration choices:**
  - Method: `PUT`
  - URL dynamically built as:  
    `{{$json.body.strapi_base_url}}/api/{{$json.body.content_type_plural}}/{{$json.body.documentId}}?status={{$json.body.status}}`
  - Sends JSON body
  - JSON body structure: `{ data: $('Webhook').item.json.body.data }`
  - Adds header `Content-Type: application/json`
  - Uses predefined credential type `strapiTokenApi`
  - `neverError: true` is enabled
- **Key expressions or variables used:**
  - URL:
    - `={{ $json.body.strapi_base_url }}/api/{{ $json.body.content_type_plural }}/{{ $json.body.documentId }}?status={{ $json.body.status }}`
  - Body:
    - `={{ { data: $('Webhook').item.json.body.data } }}`
- **Input and output connections:**
  - Input: Action detection (update branch)
  - Output: Respond to Webhook
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`
  - Requires **Strapi Token API** credential
- **Edge cases or potential failure types:**
  - Missing `documentId`
  - Invalid content type or document ID
  - Missing `data`
  - Token lacks update permission
  - Invalid `status` parameter
  - Strapi may return a structured error payload instead of a node failure because `neverError: true` is enabled
- **Sub-workflow reference:**  
  None

---

## Block 5 — Retrieve Content Entries

### Overview
This branch fetches entries from a Strapi content type. It supports optional status filtering, relation/component population, and pagination controls.

### Nodes Involved
- Get all Insights

### Node Details

#### Get all Insights
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a GET request to retrieve entries from Strapi.
- **Configuration choices:**
  - Method defaults to GET
  - URL dynamically built as:  
    `{{$json.body.strapi_base_url}}/api/{{$json.body.content_type_plural}}?status={{$json.body.status}}&populate=*&pagination[pageSize]={{$json.body.page_size}}&pagination[page]={{$json.body.page_number}}`
  - Adds header `Content-Type: application/json`
  - Uses predefined credential type `strapiTokenApi`
  - `neverError: true` is enabled
  - `populate=*` requests expanded related data where supported by Strapi
- **Key expressions or variables used:**
  - `={{ $json.body.strapi_base_url }}/api/{{ $json.body.content_type_plural }}?status={{ $json.body.status }}&populate=*&pagination[pageSize]={{ $json.body.page_size }}&pagination[page]={{ $json.body.page_number }}`
- **Input and output connections:**
  - Input: Action detection (get_all branch)
  - Output: Respond to Webhook
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`
  - Requires **Strapi Token API** credential
- **Edge cases or potential failure types:**
  - Missing or invalid base URL
  - Invalid content type
  - Token lacks read permission
  - Empty pagination values still appear in the query string
  - `populate=*` can increase payload size and response time
  - Strapi-side validation or permission failures return as response data due to `neverError: true`
- **Sub-workflow reference:**  
  None

---

## Block 6 — Webhook Response

### Overview
This block sends the selected Strapi response back to the original HTTP caller. It acts as the workflow’s explicit HTTP response because the webhook is configured in response-node mode.

### Nodes Involved
- Respond to Webhook

### Node Details

#### Respond to Webhook
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns the incoming item data to the original webhook request.
- **Configuration choices:**
  - `respondWith: allIncomingItems`
  - No custom status code or body transformation is configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**
  - Inputs:
    - Create a new Insight
    - Update an insight
    - Get all Insights
  - Output: Check if Error Occurred
- **Version-specific requirements:**
  - Uses `typeVersion: 1.5`
- **Edge cases or potential failure types:**
  - If no route reaches this node, the webhook caller may not receive the expected response
  - Since the node returns all incoming items directly, any internal response structure from Strapi is exposed as-is
- **Sub-workflow reference:**  
  None

---

## Block 7 — Error Detection and Failure Marking

### Overview
This block inspects the Strapi response after it has already been returned to the client. If an `error` object exists, the workflow execution is intentionally marked as failed for observability in n8n.

### Nodes Involved
- Check if Error Occurred
- Mark Execution as Failed

### Node Details

#### Check if Error Occurred
- **Type and technical role:** `n8n-nodes-base.if`  
  Tests whether the response payload contains a non-empty `error` property.
- **Configuration choices:**
  - Condition checks `{{$json.error}}`
  - Operation: object `notEmpty`
  - Loose type validation
  - Case sensitivity disabled
- **Key expressions or variables used:**
  - `={{ $json.error }}`
- **Input and output connections:**
  - Input: Respond to Webhook
  - True output: Mark Execution as Failed
  - False output: none connected
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - If Strapi returns errors in another shape, this check may miss them
  - If `error` exists but is not an object or is empty, behavior depends on how n8n evaluates the condition
- **Sub-workflow reference:**  
  None

#### Mark Execution as Failed
- **Type and technical role:** `n8n-nodes-base.stopAndError`  
  Explicitly stops the workflow and marks the execution as failed in n8n.
- **Configuration choices:**
  - Error message:
    - `{{$json.error.message || 'Strapi operation failed'}}`
- **Key expressions or variables used:**
  - `={{ $json.error.message || 'Strapi operation failed' }}`
- **Input and output connections:**
  - Input: Check if Error Occurred (true branch)
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - If `error.message` is missing, a fallback message is used
  - This failure occurs after the webhook response has already been sent, so it affects execution logs rather than the client’s HTTP response
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receives POST requests for Strapi actions |  | Action detection | # Create, update, and retrieve content entries in Strapi CMS v5<br>Manage your Strapi CMS v5 content entries directly from n8n via webhook or MCP — without touching the Strapi UI. Supports creating, updating, and retrieving entries for any content type.<br>> **Note:** The official n8n Strapi node only supports Strapi v3 and v4. This workflow uses the Strapi REST API directly via HTTP Request nodes, making it fully compatible with **Strapi v5**.<br>## Who's it for<br>## Sharing (n8n.io templates and GitHub)<br>Use the **same JSON file** for both channels. **n8n.io:** Submit through the template flow; importers get a copy and map the **Strapi API** credential after import. **GitHub:** Commit this `.json` (or link to **Raw**); users use **Import from File** or **Import from URL** in n8n. After any edit, re-import once in a clean n8n project to confirm the workflow loads and the webhook path is free on that instance.<br>Teams and developers running Strapi CMS v5 who want to automate content operations — publishing blog posts, syncing CMS entries from other tools, or letting AI agents manage content via MCP — without waiting for the official Strapi node to catch up.<br>## How it works<br>A single webhook endpoint accepts a JSON payload with an `action_type` parameter that routes the request to one of three branches:<br>- **`get_all`** — fetches entries for the specified content type, with optional status filtering and pagination<br>- **`create`** — creates a new entry with the fields you provide<br>- **`update`** — updates an existing entry by its Strapi `documentId`<br>The content type is passed dynamically via `content_type_plural`, so the same workflow works across all your Strapi content types without modification. The workflow also supports being called directly by an AI agent via n8n's MCP server.<br>## How to set up<br>1. Import the workflow into your n8n instance.<br>2. Create a **Strapi API Token** in your Strapi admin panel (Settings → API Tokens).<br>3. Add the token as a **Strapi Token API** credential in n8n and assign it to every **HTTP Request** node (Create, Update, Get all).<br>4. **Secure the webhook (strongly recommended):** This template’s **Webhook** node has **no authentication** by default so you can test immediately. Before production, open the Webhook node → **Authentication** → add **Header Auth** (or another method your team prefers). Create a credential with header name `X-API-Key` and a long random secret, attach it to the Webhook node, and send that header on every `POST`.<br>5. **Activate** the workflow, then copy the **Production URL** from the Webhook node — that is the URL your clients must call.<br>No Strapi base URL is stored in the workflow. Pass `strapi_base_url` in each request body as documented below.<br>## Requirements<br>- n8n (self-hosted or cloud)<br>- Strapi CMS v5 instance (cloud or self-hosted)<br>- A Strapi API Token with appropriate read/write permissions (connected to the HTTP Request nodes)<br>- **Recommended for production:** authentication on the Webhook node (e.g. **Header Auth** with `X-API-Key`) so only trusted callers can trigger Strapi operations<br>## Calling the webhook<br>Send a `POST` to the **Production webhook URL** from the Webhook node after activation. It usually looks like `https://<your-n8n-host>/webhook/<path>`. This template’s default path is `strapi-v5-content` (so the URL ends with `/webhook/strapi-v5-content`). Change it in the Webhook node if that path is already used by another **active** workflow on the same n8n instance (for example, if you imported this template twice). The workflow JSON intentionally **omits** `webhookId` so n8n can register a new webhook for each import instead of reusing the author’s instance id.<br>If you configured **Header Auth** on the Webhook node, include your secret on every request:<br>`X-API-Key: your-long-random-secret`<br>> **Security:** Without webhook authentication, anyone who can reach the URL can drive create/update/read operations using your Strapi credential inside n8n. Use authentication and HTTPS before exposing this to the internet. Because `strapi_base_url` comes from the request body, only allow trusted callers or consider fixing the base URL inside the workflow if you need stricter control.<br>### Get all entries<br>Payload must include `action_type: "get_all"`, `strapi_base_url`, and `content_type_plural`; optional `status`, `page_size`, `page_number`.<br>### Create a new entry<br>Payload must include `action_type: "create"`, `strapi_base_url`, `content_type_plural`, and `data`.<br>### Update an existing entry<br>Payload must include `action_type: "update"`, `strapi_base_url`, `content_type_plural`, `documentId`, and `data`.<br>## Using with MCP (AI agents)<br>To call this workflow from an AI agent via n8n’s MCP integration, turn on **Available in MCP** (or equivalent) in the workflow settings so the workflow is exposed to your MCP server. Agents then use the same `action_type`, `strapi_base_url`, and payload fields as the webhook examples — no separate webhook URL on the agent side. If you *also* use the public HTTP webhook, still apply **Header Auth** (or another gate) on the Webhook node as described above.<br>## How to customize<br>- **Point to a different Strapi instance:** Pass a different `strapi_base_url` per request — no workflow edits needed.<br>- **Change the content type:** Pass a different `content_type_plural` value per request — no workflow edits needed.<br>- **Filter by status:** Use `status: "draft"` or `status: "published"` in `get_all` to scope results.<br>- **Paginate large result sets:** Use `page_size` and `page_number` together.<br>- **Add more actions:** Duplicate a branch in the Switch node and add a `delete` or `get_one` action following the same pattern. |
| Respond to Webhook | Respond to Webhook | Returns the Strapi result to the caller | Create a new Insight; Update an insight; Get all Insights | Check if Error Occurred | ## Responds back to webhook with Strapi CMS API output |
| Create a new Insight | HTTP Request | Creates a new Strapi entry | Action detection | Respond to Webhook | ## Creates a request to Strapi CMS API |
| Update an insight | HTTP Request | Updates an existing Strapi entry | Action detection | Respond to Webhook | ## Creates a request to Strapi CMS API |
| Check if Error Occurred | If | Detects whether Strapi returned an error payload | Respond to Webhook | Mark Execution as Failed | ## Determines if Strapi CMS responded with error to label workflow's execution as failed |
| Mark Execution as Failed | Stop and Error | Marks execution as failed in n8n logs | Check if Error Occurred |  | ## Determines if Strapi CMS responded with error to label workflow's execution as failed |
| Sticky Note1 | Sticky Note | Documentation note |  |  | # Create, update, and retrieve content entries in Strapi CMS v5 |
| Sticky Note | Sticky Note | Documentation note |  |  | ## Determines which type of action (get, create or update) to take |
| Action detection | Switch | Routes by action_type | Webhook | Create a new Insight; Update an insight; Get all Insights | ## Determines which type of action (get, create or update) to take |
| Sticky Note2 | Sticky Note | Documentation note |  |  | ## Creates a request to Strapi CMS API |
| Sticky Note3 | Sticky Note | Documentation note |  |  | ## Responds back to webhook with Strapi CMS API output |
| Sticky Note4 | Sticky Note | Documentation note |  |  | ## Determines if Strapi CMS responded with error to label workflow's execution as failed |
| Get all Insights | HTTP Request | Retrieves Strapi entries with optional pagination and status filter | Action detection | Respond to Webhook | ## Creates a request to Strapi CMS API |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Manage Strapi CMS Content Type Template`

2. **Add a Webhook node**
   - Type: **Webhook**
   - HTTP Method: `POST`
   - Path: `strapi-v5-content`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`
   - Leave authentication disabled for initial testing if desired, but for production configure:
     - Authentication: **Header Auth**
     - Header name: `X-API-Key`
     - Secret: a long random value
   - This node is the entry point.

3. **Add a Switch node**
   - Name: `Action detection`
   - Connect: `Webhook -> Action detection`
   - Configure it to evaluate `{{$json.body.action_type}}`
   - Add three rules:
     1. equals `create`
     2. equals `update`
     3. equals `get_all`

4. **Create the Strapi credential**
   - In n8n credentials, create a **Strapi Token API** credential
   - Name it something like: `Strapi API`
   - Paste your Strapi API token from Strapi admin
   - This same credential will be attached to all HTTP Request nodes

5. **Add the create branch HTTP Request node**
   - Type: **HTTP Request**
   - Name: `Create a new Insight`
   - Connect: `Action detection` create output -> `Create a new Insight`
   - Method: `POST`
   - URL:
     - `={{ $json.body.strapi_base_url }}/api/{{ $json.body.content_type_plural }}?status={{ $json.body.status }}`
   - Authentication: `Predefined Credential Type`
   - Credential type: `Strapi Token API`
   - Select credential: `Strapi API`
   - Send Headers: enabled
   - Header:
     - `Content-Type: application/json`
   - Send Body: enabled
   - Body Content Type / Specify Body: JSON
   - JSON Body:
     - `={{ { data: $('Webhook').item.json.body.data } }}`
   - In node options, set response handling so the request **does not fail automatically** on HTTP errors:
     - Enable `Never Error` / equivalent response setting

6. **Add the update branch HTTP Request node**
   - Type: **HTTP Request**
   - Name: `Update an insight`
   - Connect: `Action detection` update output -> `Update an insight`
   - Method: `PUT`
   - URL:
     - `={{ $json.body.strapi_base_url }}/api/{{ $json.body.content_type_plural }}/{{ $json.body.documentId }}?status={{ $json.body.status }}`
   - Authentication: `Predefined Credential Type`
   - Credential type: `Strapi Token API`
   - Credential: `Strapi API`
   - Send Headers: enabled
   - Header:
     - `Content-Type: application/json`
   - Send Body: enabled
   - Body as JSON
   - JSON Body:
     - `={{ { data: $('Webhook').item.json.body.data } }}`
   - Enable `Never Error`

7. **Add the retrieval branch HTTP Request node**
   - Type: **HTTP Request**
   - Name: `Get all Insights`
   - Connect: `Action detection` get_all output -> `Get all Insights`
   - Method: `GET`
   - URL:
     - `={{ $json.body.strapi_base_url }}/api/{{ $json.body.content_type_plural }}?status={{ $json.body.status }}&populate=*&pagination[pageSize]={{ $json.body.page_size }}&pagination[page]={{ $json.body.page_number }}`
   - Authentication: `Predefined Credential Type`
   - Credential type: `Strapi Token API`
   - Credential: `Strapi API`
   - Send Headers: enabled
   - Header:
     - `Content-Type: application/json`
   - Enable `Never Error`

8. **Add a Respond to Webhook node**
   - Type: **Respond to Webhook**
   - Name: `Respond to Webhook`
   - Configure:
     - Respond With: `All Incoming Items`
   - Connect:
     - `Create a new Insight -> Respond to Webhook`
     - `Update an insight -> Respond to Webhook`
     - `Get all Insights -> Respond to Webhook`

9. **Add an If node for error inspection**
   - Type: **If**
   - Name: `Check if Error Occurred`
   - Connect: `Respond to Webhook -> Check if Error Occurred`
   - Condition:
     - Left value: `={{ $json.error }}`
     - Operator: `is not empty` / object `notEmpty`
   - Leave false output unconnected if you want successful runs to simply end

10. **Add a Stop and Error node**
    - Type: **Stop and Error**
    - Name: `Mark Execution as Failed`
    - Connect: `Check if Error Occurred` true output -> `Mark Execution as Failed`
    - Error message:
      - `={{ $json.error.message || 'Strapi operation failed' }}`

11. **Optionally add documentation sticky notes**
    - One around the routing block:
      - `Determines which type of action (get, create or update) to take`
    - One around the HTTP Request nodes:
      - `Creates a request to Strapi CMS API`
    - One around the response node:
      - `Responds back to webhook with Strapi CMS API output`
    - One around the error handling block:
      - `Determines if Strapi CMS responded with error to label workflow's execution as failed`

12. **Test the create operation**
    - Use the webhook test URL or activate the workflow for the production URL
    - Example body:
      ```json
      {
        "action_type": "create",
        "strapi_base_url": "https://your-strapi-instance.com",
        "content_type_plural": "articles",
        "data": {
          "title": "My article",
          "body": "Hello world"
        }
      }
      ```

13. **Test the update operation**
    - Example body:
      ```json
      {
        "action_type": "update",
        "strapi_base_url": "https://your-strapi-instance.com",
        "content_type_plural": "articles",
        "documentId": "your-document-id",
        "data": {
          "title": "Updated title"
        }
      }
      ```

14. **Test the retrieval operation**
    - Example body:
      ```json
      {
        "action_type": "get_all",
        "strapi_base_url": "https://your-strapi-instance.com",
        "content_type_plural": "articles",
        "status": "published",
        "page_size": 10,
        "page_number": 1
      }
      ```

15. **Activate the workflow**
    - Copy the production webhook URL from the Webhook node
    - Ensure the path is unique on your n8n instance
    - If using Header Auth, always send the required header

16. **Recommended hardening improvements**
    - Add validation before the Switch node to ensure required fields exist
    - Restrict or hardcode `strapi_base_url` if only one Strapi instance should be reachable
    - Add a fallback branch for unsupported `action_type`
    - Normalize query parameters so empty values are not appended

### Sub-workflow setup
This workflow does **not** use any sub-workflows and does **not** invoke any child workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is built for Strapi CMS v5 because the official n8n Strapi node only supports v3 and v4. | Workflow-level design note |
| The same workflow can be used through webhook calls or via n8n MCP if the workflow setting is enabled. | Workflow settings (`Available in MCP`) |
| Secure the webhook before production; recommended header is `X-API-Key`. | Webhook authentication practice |
| Because `strapi_base_url` comes from the request body, only trusted callers should be allowed, or the base URL should be hardcoded. | Security consideration |
| Default webhook path is `strapi-v5-content`; if already used by another active workflow, change it. | Webhook deployment note |
| `populate=*` in the get_all branch may increase payload size and response time. | Retrieval branch behavior |
| The workflow setting `availableInMCP` is currently `false`, so MCP exposure is not enabled in the provided version. | Current workflow settings |