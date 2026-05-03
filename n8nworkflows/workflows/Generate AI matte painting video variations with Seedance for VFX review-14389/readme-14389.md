Generate AI matte painting video variations with Seedance for VFX review

https://n8nworkflows.xyz/workflows/generate-ai-matte-painting-video-variations-with-seedance-for-vfx-review-14389


# Generate AI matte painting video variations with Seedance for VFX review

## 1. Workflow Overview

This workflow orchestrates the generation of multiple AI matte painting video variations using Seedance, tracks render completion through polling, prepares downstream VFX-review metadata, creates review tasks, and notifies stakeholders. It also includes a separate error-handling entry point that pushes workflow failures to Slack.

Because the provided workflow JSON contains node names and topology but almost no node parameter payloads, the functional intent can be inferred reliably from naming and connections, while many field-level settings must be treated as unspecified.

### 1.1 Input Reception and Validation
The workflow starts from a webhook that receives a matte painting request. A code node validates the incoming payload and extracts the required fields for downstream use.

### 1.2 Variation Expansion and Request Preparation
The validated request is expanded into four atmosphere-based variations. Each variation is transformed into the request body expected by the Seedance generation API.

### 1.3 Seedance Job Submission and Polling
Each variation is submitted to Seedance through an HTTP request. The returned job identifier is merged with the original metadata, then the workflow repeatedly polls Seedance until the render is complete, waiting 20 seconds between checks when necessary.

### 1.4 Asset Preparation and Review Packaging
Once rendering completes, the workflow builds asset metadata, generates a Nuke comp template, and branches into additional delivery/production-support actions including file download and ClickUp record creation.

### 1.5 Review Tasking and Stakeholder Notifications
A Jira task is created for review, all variations are aggregated, and a Slack message is sent to notify a supervisor. In parallel from the metadata stage, the generated asset is downloaded and sent by Gmail.

### 1.6 Error Handling
A dedicated `Error Trigger` entry point catches workflow-level failures and sends an alert to Slack.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Trigger and Validation

**Overview:**  
This block receives incoming requests and prepares a normalized input object for the rest of the workflow. It is the primary entry point for matte painting generation.

**Nodes Involved:**  
- Webhook: Matte Painting Request2
- Validate & Extract Input2

### Node Details

#### Webhook: Matte Painting Request2
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Receives external HTTP requests and starts the workflow.
- **Configuration choices:**  
  The JSON does not expose the webhook HTTP method, path, response mode, or authentication settings. Only the webhook ID is present.
- **Key expressions or variables used:**  
  Not visible in the JSON.
- **Input and output connections:**  
  - Input: none, entry point
  - Output: `Validate & Extract Input2`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Invalid payload shape
  - Missing required headers or auth if webhook protection is enabled
  - Incorrect content type
  - Timeout if the workflow is expected to return synchronously and downstream steps are long-running
- **Sub-workflow reference:**  
  None

#### Validate & Extract Input2
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates inbound request content and maps it into a cleaner internal structure.
- **Configuration choices:**  
  No code is included in the JSON export. Based on its name, it likely checks required request fields such as source image/video reference, prompt, project identifiers, shot info, delivery contact, or stylistic constraints.
- **Key expressions or variables used:**  
  Not visible; likely uses `$json` from webhook payload.
- **Input and output connections:**  
  - Input: `Webhook: Matte Painting Request2`
  - Output: `Fan-Out: 4 Atmosphere Variations2`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Throwing a runtime error if required fields are absent
  - Expression failures when referencing undefined nested payload fields
  - JSON parsing assumptions inside the code node
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: Fan-Out and Request Build

**Overview:**  
This block duplicates the validated request into four atmosphere-based variants, then prepares a request body for each variation to be sent to Seedance.

**Nodes Involved:**  
- Fan-Out: 4 Atmosphere Variations2
- Build Request Body

### Node Details

#### Fan-Out: 4 Atmosphere Variations2
- **Type and technical role:** `n8n-nodes-base.code`  
  Creates multiple variation items from one validated input.
- **Configuration choices:**  
  The name strongly suggests generation of four atmosphere variations, likely by cloning the input and assigning variation-specific prompt modifiers such as mood, lighting, weather, or environment treatment.
- **Key expressions or variables used:**  
  Not visible; likely loops over an internal array of four atmosphere presets.
- **Input and output connections:**  
  - Input: `Validate & Extract Input2`
  - Output: `Build Request Body`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Producing zero items if preset logic fails
  - Incorrect item structure causing downstream HTTP body generation to fail
  - Unexpected prompt duplication or malformed variant labels
- **Sub-workflow reference:**  
  None

#### Build Request Body
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts each variation item into the exact request payload expected by Seedance.
- **Configuration choices:**  
  Not visible. Likely maps internal fields into API fields such as prompt, reference asset, model settings, output format, duration, or render options.
- **Key expressions or variables used:**  
  Not visible; likely constructs JSON from `$json`.
- **Input and output connections:**  
  - Input: `Fan-Out: 4 Atmosphere Variations2`
  - Output: `Seedance: Generate Variation1`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Missing required API fields
  - Bad data types in the body
  - Prompt length or asset URL constraints imposed by Seedance
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Job Submission and Polling

**Overview:**  
This block submits each variation to Seedance, extracts or merges the resulting job identifiers with local metadata, and repeatedly polls until rendering completes.

**Nodes Involved:**  
- Seedance: Generate Variation1
- Merge Job ID + Metadata
- Poll: Check Job Status1
- Render Complete?2
- Wait 20s1

### Node Details

#### Seedance: Generate Variation1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Seedance generation API to create one render job per variation.
- **Configuration choices:**  
  The exact URL, method, authentication mode, headers, and body are not visible. Given its role, this node is likely configured for a POST request with JSON body from the previous code node.
- **Key expressions or variables used:**  
  Presumably request body expressions referencing prior node output.
- **Input and output connections:**  
  - Input: `Build Request Body`
  - Output: `Merge Job ID + Metadata`
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - 4xx auth or validation errors
  - 429 rate limits when submitting multiple variations
  - 5xx provider-side failures
  - Timeout on slow API response
  - Response schema mismatch if job ID is not where expected
- **Sub-workflow reference:**  
  None

#### Merge Job ID + Metadata
- **Type and technical role:** `n8n-nodes-base.code`  
  Combines Seedance’s job response with original variation metadata required for polling and later packaging.
- **Configuration choices:**  
  Code is not visible. It likely preserves fields like shot name, variation label, requester, prompt, and adds `jobId`.
- **Key expressions or variables used:**  
  Not visible; likely references HTTP response JSON and current item metadata.
- **Input and output connections:**  
  - Input: `Seedance: Generate Variation1`
  - Output: `Poll: Check Job Status1`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Missing `jobId`
  - Overwriting metadata accidentally
  - Inconsistent merged schema affecting polling logic
- **Sub-workflow reference:**  
  None

#### Poll: Check Job Status1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries Seedance for current job status.
- **Configuration choices:**  
  Exact endpoint and method are not exposed. This is likely a GET request using the merged job ID.
- **Key expressions or variables used:**  
  Likely injects `jobId` into URL path, query string, or request body.
- **Input and output connections:**  
  - Input: `Merge Job ID + Metadata` and loop-back from `Wait 20s1`
  - Output: `Render Complete?2`
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - Polling before job is registered
  - Missing or stale job ID
  - API rate limits due to frequent polling
  - Failure state returned by Seedance but not explicitly handled
- **Sub-workflow reference:**  
  None

#### Render Complete?2
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the render is complete.
- **Configuration choices:**  
  Condition details are not visible. It likely checks a field such as `status === "completed"` or similar.
- **Key expressions or variables used:**  
  Not visible; likely references status from polling response.
- **Input and output connections:**  
  - Input: `Poll: Check Job Status1`
  - True output: `Build Asset Metadata3`
  - False output: `Wait 20s1`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Status vocabulary mismatch (`done` vs `completed`)
  - Failure/cancelled statuses incorrectly routed to wait loop
  - Null status causing expression errors
- **Sub-workflow reference:**  
  None

#### Wait 20s1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays the workflow before polling again.
- **Configuration choices:**  
  The name indicates a 20-second pause, but actual wait settings are not visible in the JSON. A wait webhook ID is present, which is internal to n8n’s resume mechanism.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: `Render Complete?2` false branch
  - Output: `Poll: Check Job Status1`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - Excessively frequent polling if actual delay differs from the node name
  - Long queue accumulation under high load
  - Execution expiration if polling lasts too long
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: Metadata, Download, and Production Packaging

**Overview:**  
After a render completes, this block prepares structured metadata, generates a Nuke comp template, downloads the produced file, and creates an external production-tracking record.

**Nodes Involved:**  
- Build Asset Metadata3
- Generate Nuke Comp Template1
- Download File
- Add Record in Clickup

### Node Details

#### Build Asset Metadata3
- **Type and technical role:** `n8n-nodes-base.code`  
  Constructs a normalized metadata object for the generated asset and fans it out to downstream systems.
- **Configuration choices:**  
  Not visible. Likely includes render URL, variation name, review context, project/shot identifiers, and prompt metadata.
- **Key expressions or variables used:**  
  Not visible; likely combines polling response with original request info.
- **Input and output connections:**  
  - Input: `Render Complete?2` true branch
  - Outputs:
    - `Generate Nuke Comp Template1`
    - `Download File`
    - `Add Record in Clickup`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Missing output URL from final Seedance status
  - Incorrect file extension or MIME assumptions
  - Missing project metadata needed by Jira/ClickUp/email
- **Sub-workflow reference:**  
  None

#### Generate Nuke Comp Template1
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces a Nuke composition template or template payload based on generated asset metadata.
- **Configuration choices:**  
  No code is shown. It likely assembles comp script references, input plate path placeholders, matte painting asset references, or review instructions.
- **Key expressions or variables used:**  
  Not visible.
- **Input and output connections:**  
  - Input: `Build Asset Metadata3`
  - Output: `Jira: Create Review Task`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Invalid template text generation
  - Missing required metadata fields for Nuke pipeline conventions
  - Formatting errors if Jira expects plain text vs markdown
- **Sub-workflow reference:**  
  None

#### Download File
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves the generated media file from a remote URL for later email delivery.
- **Configuration choices:**  
  Exact request details are not present. This node is likely configured to download binary data from the asset URL emitted by the metadata node.
- **Key expressions or variables used:**  
  Likely uses asset download URL from `Build Asset Metadata3`.
- **Input and output connections:**  
  - Input: `Build Asset Metadata3`
  - Output: `Send a message`
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - Expired signed URL
  - Large file download causing timeout or memory issues
  - Wrong response format if binary mode is not enabled
- **Sub-workflow reference:**  
  None

#### Add Record in Clickup
- **Type and technical role:** `n8n-nodes-base.clickUp`  
  Creates or updates a ClickUp record tied to the generated asset.
- **Configuration choices:**  
  The exact operation, workspace/list/task target, and fields are not visible. Based on the name, this node likely records variation metadata for production tracking.
- **Key expressions or variables used:**  
  Not visible.
- **Input and output connections:**  
  - Input: `Build Asset Metadata3`
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Invalid ClickUp credentials
  - Missing list/task identifiers
  - Field mapping mismatches
  - API quota issues
- **Sub-workflow reference:**  
  None

---

## 2.5 Block: Review Task Creation and Notification

**Overview:**  
This block creates a review task in Jira, aggregates the finished variations, and notifies a supervisor in Slack. It represents the formal review handoff stage.

**Nodes Involved:**  
- Jira: Create Review Task
- Aggregate All Variations3
- Slack: Notify Supervisor

### Node Details

#### Jira: Create Review Task
- **Type and technical role:** `n8n-nodes-base.jira`  
  Creates a Jira issue for review of the generated matte painting variation(s).
- **Configuration choices:**  
  The project key, issue type, summary/body mapping, and assignee rules are not visible. The node name suggests task creation specifically for review.
- **Key expressions or variables used:**  
  Likely references generated metadata and Nuke template from the previous node.
- **Input and output connections:**  
  - Input: `Generate Nuke Comp Template1`
  - Output: `Aggregate All Variations3`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Jira credential/auth issues
  - Missing required project or issue fields
  - Invalid rich-text formatting
  - Permissions preventing issue creation
- **Sub-workflow reference:**  
  None

#### Aggregate All Variations3
- **Type and technical role:** `n8n-nodes-base.code`  
  Collects or reformats information from the multiple variation branches into a single review summary.
- **Configuration choices:**  
  No code is visible. It likely groups completed items by original request and prepares a consolidated message.
- **Key expressions or variables used:**  
  Not visible.
- **Input and output connections:**  
  - Input: `Jira: Create Review Task`
  - Output: `Slack: Notify Supervisor`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Incorrect aggregation if items arrive independently
  - Duplicate notifications
  - Missing variation labels or URLs
- **Sub-workflow reference:**  
  None

#### Slack: Notify Supervisor
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack notification summarizing review-ready assets.
- **Configuration choices:**  
  Channel, message formatting, and mention behavior are not visible.
- **Key expressions or variables used:**  
  Likely includes Jira task link, variation list, and asset URLs.
- **Input and output connections:**  
  - Input: `Aggregate All Variations3`
  - Output: none
- **Version-specific requirements:**  
  Type version `2.3`
- **Edge cases or potential failure types:**  
  - Slack auth issues
  - Invalid channel ID
  - Message formatting errors
  - Rate limits
- **Sub-workflow reference:**  
  None

---

## 2.6 Block: Email Delivery

**Overview:**  
This small branch downloads the generated file and emails it out. It appears to be a delivery-oriented side branch triggered after metadata construction.

**Nodes Involved:**  
- Download File
- Send a message

### Node Details

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email, likely with the downloaded file attached or linked.
- **Configuration choices:**  
  Subject, recipient, body, and attachment configuration are not visible. The Gmail webhook ID shown is internal metadata, not an email trigger.
- **Key expressions or variables used:**  
  Likely uses binary data from `Download File` and metadata from upstream.
- **Input and output connections:**  
  - Input: `Download File`
  - Output: none
- **Version-specific requirements:**  
  Type version `2.2`
- **Edge cases or potential failure types:**  
  - Gmail OAuth scope issues
  - Attachment size limits
  - Missing binary property name
  - Invalid recipient address
- **Sub-workflow reference:**  
  None

---

## 2.7 Block: Error Handling

**Overview:**  
This is a separate workflow entry path that activates whenever the workflow errors. It sends an alert to Slack for operational visibility.

**Nodes Involved:**  
- On Workflow Error
- Slack: Error Alert

### Node Details

#### On Workflow Error
- **Type and technical role:** `n8n-nodes-base.errorTrigger`  
  Starts a separate execution whenever the parent workflow encounters an error.
- **Configuration choices:**  
  No configurable details are shown here beyond its presence.
- **Key expressions or variables used:**  
  Typically provides error metadata such as execution ID, failed node, and message via `$json`.
- **Input and output connections:**  
  - Input: none, entry point
  - Output: `Slack: Error Alert`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Limited context if the failed node did not expose structured output
  - Recursive alerting if Slack alert itself fails and workflow settings re-trigger
- **Sub-workflow reference:**  
  None

#### Slack: Error Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an operational error alert.
- **Configuration choices:**  
  Exact target channel and message body are not visible.
- **Key expressions or variables used:**  
  Likely references error information from `On Workflow Error`.
- **Input and output connections:**  
  - Input: `On Workflow Error`
  - Output: none
- **Version-specific requirements:**  
  Type version `2.3`
- **Edge cases or potential failure types:**  
  - Missing Slack credentials
  - Alert loops if not isolated from workflow-wide failure handling
- **Sub-workflow reference:**  
  None

---

## 2.8 Block: Documentation and Layout Notes

**Overview:**  
These nodes are non-executable canvas annotations used to structure the workflow visually. In the provided JSON, all sticky note contents are empty.

**Nodes Involved:**  
- 📋 Overview
- Section: Trigger & Validation
- Section: Fan-Out & Request Build
- Section: Job Submission & Polling
- Section: Metadata & Nuke Template
- Section: Review & Notifications
- Security Notes
- Section: Error Handler

### Node Details

For each sticky note:
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Provides visual grouping/documentation only.
- **Configuration choices:**  
  All `content` fields are empty in the JSON.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None in execution; only documentation impact
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook: Matte Painting Request2 | Webhook | Receives external matte painting generation requests |  | Validate & Extract Input2 |  |
| Validate & Extract Input2 | Code | Validates incoming payload and extracts normalized fields | Webhook: Matte Painting Request2 | Fan-Out: 4 Atmosphere Variations2 |  |
| Fan-Out: 4 Atmosphere Variations2 | Code | Expands one request into four atmosphere variations | Validate & Extract Input2 | Build Request Body |  |
| Build Request Body | Code | Builds Seedance API payload per variation | Fan-Out: 4 Atmosphere Variations2 | Seedance: Generate Variation1 |  |
| Seedance: Generate Variation1 | HTTP Request | Submits generation jobs to Seedance | Build Request Body | Merge Job ID + Metadata |  |
| Merge Job ID + Metadata | Code | Merges Seedance job ID with local metadata | Seedance: Generate Variation1 | Poll: Check Job Status1 |  |
| Poll: Check Job Status1 | HTTP Request | Polls render/job status from Seedance | Merge Job ID + Metadata; Wait 20s1 | Render Complete?2 |  |
| Render Complete?2 | If | Checks whether the Seedance render is complete | Poll: Check Job Status1 | Build Asset Metadata3; Wait 20s1 |  |
| Wait 20s1 | Wait | Delays before the next polling cycle | Render Complete?2 | Poll: Check Job Status1 |  |
| Build Asset Metadata3 | Code | Builds final asset metadata for downstream systems | Render Complete?2 | Generate Nuke Comp Template1; Download File; Add Record in Clickup |  |
| Generate Nuke Comp Template1 | Code | Builds Nuke comp template or payload | Build Asset Metadata3 | Jira: Create Review Task |  |
| Jira: Create Review Task | Jira | Creates a review task in Jira | Generate Nuke Comp Template1 | Aggregate All Variations3 |  |
| Aggregate All Variations3 | Code | Aggregates all generated variations into a summary | Jira: Create Review Task | Slack: Notify Supervisor |  |
| Slack: Notify Supervisor | Slack | Notifies supervisor that review assets are ready | Aggregate All Variations3 |  |  |
| Download File | HTTP Request | Downloads the generated file for email delivery | Build Asset Metadata3 | Send a message |  |
| Send a message | Gmail | Sends delivery email with downloaded asset | Download File |  |  |
| Add Record in Clickup | ClickUp | Adds a production-tracking record in ClickUp | Build Asset Metadata3 |  |  |
| On Workflow Error | Error Trigger | Starts error-handling execution when the workflow fails |  | Slack: Error Alert |  |
| Slack: Error Alert | Slack | Sends operational failure alert to Slack | On Workflow Error |  |  |
| 📋 Overview | Sticky Note | Visual documentation/grouping |  |  |  |
| Section: Trigger & Validation | Sticky Note | Visual section marker |  |  |  |
| Section: Fan-Out & Request Build | Sticky Note | Visual section marker |  |  |  |
| Section: Job Submission & Polling | Sticky Note | Visual section marker |  |  |  |
| Section: Metadata & Nuke Template | Sticky Note | Visual section marker |  |  |  |
| Section: Review & Notifications | Sticky Note | Visual section marker |  |  |  |
| Security Notes | Sticky Note | Visual documentation/grouping |  |  |  |
| Section: Error Handler | Sticky Note | Visual section marker |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

Because most node parameters are empty in the provided JSON export, the steps below reconstruct the workflow structure faithfully and identify where you must define your own field mappings, API endpoints, and credentials.

1. **Create a new workflow**
   - Name it something like: `Generate AI Matte Paintings with Seedance and Automate Variations, Review, and Delivery`.
   - Keep execution order as default or set it to `v1` if you want to mirror the exported workflow setting.

2. **Add the main webhook trigger**
   - Add a **Webhook** node.
   - Rename it to **Webhook: Matte Painting Request2**.
   - Configure:
     - HTTP method: choose the method your caller will use, typically `POST`
     - Path: define a stable endpoint, e.g. `/matte-painting-request`
     - Authentication: optional but recommended
     - Response behavior: if the workflow will take a long time, prefer an immediate acknowledgement rather than waiting for completion
   - This node is the primary workflow entry point.

3. **Add input validation**
   - Add a **Code** node after the webhook.
   - Rename it to **Validate & Extract Input2**.
   - Connect: `Webhook: Matte Painting Request2 -> Validate & Extract Input2`
   - In the code:
     - Validate required fields from `$json`
     - Recommended fields:
       - `projectId`
       - `shotId`
       - `requestId`
       - `prompt`
       - `referenceAssetUrl` or `plateUrl`
       - `requesterEmail`
       - `supervisorSlackChannel` or recipient info
     - Throw an error if any critical field is missing
     - Return a normalized object with consistent keys

4. **Add variation fan-out logic**
   - Add a **Code** node.
   - Rename it to **Fan-Out: 4 Atmosphere Variations2**.
   - Connect: `Validate & Extract Input2 -> Fan-Out: 4 Atmosphere Variations2`
   - In the code:
     - Create four items from the validated input
     - Add fields like:
       - `variationIndex`
       - `variationName`
       - `atmosphereModifier`
     - Example variations:
       - moody dusk
       - overcast cinematic
       - golden hour
       - storm-lit dramatic
     - Return one n8n item per variation

5. **Add request-body construction**
   - Add another **Code** node.
   - Rename it to **Build Request Body**.
   - Connect: `Fan-Out: 4 Atmosphere Variations2 -> Build Request Body`
   - In the code:
     - Build the exact JSON body expected by Seedance
     - Include:
       - prompt
       - reference input URL or asset ID
       - model or generation mode
       - duration if generating video
       - aspect ratio or resolution
       - variation metadata for later tracking
     - Return items with both metadata and request body

6. **Add Seedance generation request**
   - Add an **HTTP Request** node.
   - Rename it to **Seedance: Generate Variation1**.
   - Connect: `Build Request Body -> Seedance: Generate Variation1`
   - Configure:
     - Method: likely `POST`
     - URL: Seedance generation endpoint
     - Authentication: according to Seedance API, often header-based bearer token
     - Send body as JSON
     - Map the body from fields prepared in `Build Request Body`
   - Credential setup:
     - If using predefined n8n generic HTTP auth, store API key securely
     - Otherwise add an `Authorization: Bearer ...` header via expressions/credentials
   - Expected output:
     - Response containing a job ID or task ID

7. **Add job/metadata merge**
   - Add a **Code** node.
   - Rename it to **Merge Job ID + Metadata**.
   - Connect: `Seedance: Generate Variation1 -> Merge Job ID + Metadata`
   - In the code:
     - Read the Seedance response
     - Extract `jobId` or equivalent
     - Merge it with upstream fields:
       - project/shot info
       - variation info
       - prompt
       - requester info
     - Throw an error if no job ID is returned

8. **Add polling request**
   - Add an **HTTP Request** node.
   - Rename it to **Poll: Check Job Status1**.
   - Connect: `Merge Job ID + Metadata -> Poll: Check Job Status1`
   - Configure:
     - Method: likely `GET`
     - URL: Seedance status endpoint, including the job ID
     - Authentication: same Seedance credential strategy as above
   - Expected response:
     - status field
     - output asset URL when completed
     - optional error/failure details

9. **Add completion check**
   - Add an **If** node.
   - Rename it to **Render Complete?2**.
   - Connect: `Poll: Check Job Status1 -> Render Complete?2`
   - Configure condition:
     - Left side: expression pointing to job status
     - Operator: equals
     - Right side: your completion value, e.g. `completed`
   - Consider adding alternative handling for explicit failure statuses such as:
     - `failed`
     - `cancelled`
     - `error`

10. **Add wait node for polling loop**
    - Add a **Wait** node.
    - Rename it to **Wait 20s1**.
    - Connect the **false** branch of `Render Complete?2` to `Wait 20s1`
    - Configure a time-based wait of **20 seconds**
    - Connect: `Wait 20s1 -> Poll: Check Job Status1`
    - This creates the polling loop

11. **Add asset metadata builder**
    - Add a **Code** node.
    - Rename it to **Build Asset Metadata3**.
    - Connect the **true** branch of `Render Complete?2` to `Build Asset Metadata3`
    - In the code:
      - Extract final asset URL and any returned render metadata
      - Preserve request context and variation info
      - Build a structured object such as:
        - `assetUrl`
        - `assetFilename`
        - `variationName`
        - `projectId`
        - `shotId`
        - `reviewSummary`
        - `requesterEmail`
      - This node will branch into three downstream actions

12. **Add Nuke comp template generation**
    - Add a **Code** node.
    - Rename it to **Generate Nuke Comp Template1**.
    - Connect: `Build Asset Metadata3 -> Generate Nuke Comp Template1`
    - In the code:
      - Build a comp template text block, JSON payload, or formatted description
      - Include references to:
        - generated matte painting/video asset
        - source plate or shot
        - expected compositing notes
      - Return data suitable for Jira issue creation

13. **Add Jira review task creation**
    - Add a **Jira** node.
    - Rename it to **Jira: Create Review Task**
    - Connect: `Generate Nuke Comp Template1 -> Jira: Create Review Task`
    - Configure Jira credentials:
      - Jira Cloud email + API token, or OAuth if preferred
    - Set operation:
      - likely `Create Issue`
    - Configure:
      - Project
      - Issue type such as `Task` or `Review`
      - Summary built from project/shot/variation fields
      - Description including comp template and asset links

14. **Add aggregation node**
    - Add a **Code** node.
    - Rename it to **Aggregate All Variations3**
    - Connect: `Jira: Create Review Task -> Aggregate All Variations3`
    - In the code:
      - Aggregate or summarize generated variations
      - Depending on execution mode, you may need to:
        - combine all items into one message
        - or simply format each completed item independently
      - Include Jira issue key/link and asset references

15. **Add supervisor Slack notification**
    - Add a **Slack** node.
    - Rename it to **Slack: Notify Supervisor**
    - Connect: `Aggregate All Variations3 -> Slack: Notify Supervisor`
    - Configure Slack credentials:
      - OAuth2 app or Slack token credential in n8n
    - Set operation to send a message
    - Configure:
      - Channel
      - Message body with summary, variation links, and Jira review link

16. **Add download branch**
    - Add an **HTTP Request** node.
    - Rename it to **Download File**
    - Connect: `Build Asset Metadata3 -> Download File`
    - Configure:
      - Method: `GET`
      - URL: final asset URL from metadata
      - Response format: **File/Binary**
    - Make sure binary download is enabled and the binary property name is known

17. **Add Gmail delivery node**
    - Add a **Gmail** node.
    - Rename it to **Send a message**
    - Connect: `Download File -> Send a message`
    - Configure Gmail credentials:
      - OAuth2 with send-mail scope
    - Configure:
      - To: requester or delivery list
      - Subject: include project/shot/variation info
      - Body: include review context or links
      - Attachment: binary property from `Download File`
    - Watch Gmail attachment size limits

18. **Add ClickUp tracking branch**
    - Add a **ClickUp** node.
    - Rename it to **Add Record in Clickup**
    - Connect: `Build Asset Metadata3 -> Add Record in Clickup`
    - Configure ClickUp credentials:
      - API token
    - Choose the operation that fits your workspace model:
      - create task
      - create checklist item
      - update task
      - add custom-field data
    - Map metadata:
      - project
      - shot
      - variation
      - asset URL
      - status

19. **Add error trigger**
    - Add an **Error Trigger** node on the canvas as a separate entry point.
    - Rename it to **On Workflow Error**
    - This node should not be connected to the main webhook path.

20. **Add Slack error alert**
    - Add a **Slack** node.
    - Rename it to **Slack: Error Alert**
    - Connect: `On Workflow Error -> Slack: Error Alert`
    - Configure:
      - Channel for operations alerts
      - Message content including:
        - workflow name
        - failed node
        - error message
        - execution ID
    - Reuse Slack credentials or create separate ops credentials

21. **Add optional sticky notes for layout**
    - Add sticky notes matching the exported workflow if you want visual parity:
      - `📋 Overview`
      - `Section: Trigger & Validation`
      - `Section: Fan-Out & Request Build`
      - `Section: Job Submission & Polling`
      - `Section: Metadata & Nuke Template`
      - `Section: Review & Notifications`
      - `Security Notes`
      - `Section: Error Handler`
    - In the provided workflow, all sticky note contents are empty

22. **Validate binary and multi-item behavior**
    - Ensure `Download File` passes binary data correctly to Gmail
    - Confirm your code nodes preserve required fields during item fan-out and polling
    - Decide whether `Aggregate All Variations3` should wait for all variations before notifying; this may require explicit aggregation logic, static data, or redesign if the current simple chain sends one message per variation

23. **Test with a sample webhook payload**
    - Send a sample request to the webhook
    - Verify:
      - validation catches bad inputs
      - four variations are created
      - Seedance job submission works
      - polling exits correctly
      - metadata includes final asset URL
      - Jira/Slack/Gmail/ClickUp actions succeed

24. **Harden the workflow for production**
    - Add explicit handling for non-completion terminal statuses
    - Add retry logic or “Continue On Fail” selectively where appropriate
    - Consider a max polling attempt count to avoid infinite loops
    - Secure webhook access
    - Store all API credentials in n8n credentials, never inline in code

### Credential Configuration Summary
- **Seedance:** likely API key or bearer token in HTTP Request nodes
- **Jira:** Jira Cloud API token or OAuth
- **Slack:** Slack OAuth/token credential
- **Gmail:** OAuth2 with send permissions
- **ClickUp:** API token credential

### Sub-workflow Setup
This workflow does **not** include any `Execute Workflow` node or embedded sub-workflow invocation. There are multiple entry points, however:
- Main entry point: `Webhook: Matte Painting Request2`
- Error entry point: `On Workflow Error`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The provided workflow JSON contains almost no node parameter content, so field-level behavior must be inferred from node names and graph structure. | Applies to the whole workflow |
| All sticky notes are present only as visual labels; their content is empty in the export. | Canvas documentation |
| No external links, blog links, or branded notes are included in the provided JSON. | General |
| The workflow contains two independent entry points: one webhook for processing requests and one error trigger for operational alerting. | Architecture note |
| Aggregation behavior is not guaranteed by topology alone; if a single consolidated notification is required after all four variations finish, `Aggregate All Variations3` must explicitly implement cross-item aggregation. | Important implementation note |
| Polling loops should include terminal failure handling and optional max-attempt limits to prevent endless waiting. | Reliability note |

If you want, I can also convert this into a more compact machine-oriented specification, or produce a node-by-node implementation draft with example code for each Code node.