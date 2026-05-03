Generate AI videos from prompts with Seedance, Jira, Slack, and Gmail

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-prompts-with-seedance--jira--slack--and-gmail-14391


# Generate AI videos from prompts with Seedance, Jira, Slack, and Gmail

# 1. Workflow Overview

This workflow receives a prompt through a webhook, prepares it for AI video generation, submits it to an external video-generation API, polls until the job completes, then creates review artifacts and notifications across Jira, Slack, and Gmail. It also includes a separate error-handling entry point that posts workflow failures to Slack.

The workflow is organized into these logical blocks:

## 1.1 Input Reception & Prompt Preparation
Receives an inbound webhook request, normalizes or enriches incoming fields, and runs JavaScript-based prompt processing before making the first external API call.

## 1.2 Video Generation Submission & Polling
Starts the AI video generation process via HTTP, checks status repeatedly, and uses a wait/retry loop until the generation is complete or the condition changes.

## 1.3 Metadata Construction & Review Ticketing
Once the video is ready, derives structured metadata and tags, then creates a Jira issue for review or downstream production handling.

## 1.4 Notifications & Delivery
Notifies a VFX supervisor in Slack, reshapes payload fields for final delivery, fetches or prepares final asset data through HTTP, and sends an email via Gmail.

## 1.5 Error Handling
A dedicated error-trigger branch catches workflow-level failures and posts an alert to Slack.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception & Prompt Preparation

**Overview:**  
This block acts as the main entry point for video requests. It receives the source payload through a webhook, reshapes the input, and applies JavaScript logic to sanitize or prepare the prompt before calling the generation API.

**Nodes Involved:**  
- Webhook
- Edit Fields2
- Code in JavaScript1

### Node: Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that exposes an HTTP endpoint for external systems to trigger the workflow.
- **Configuration choices:**  
  The JSON does not expose detailed parameters, but the node is configured with a webhook ID, meaning it can receive requests at a generated n8n webhook URL.
- **Key expressions or variables used:**  
  Not visible in the provided JSON.
- **Input and output connections:**  
  - Input: none; this is a trigger node  
  - Output: `Edit Fields2`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`, which is compatible with current n8n webhook behavior but may differ slightly in options from newer minor versions.
- **Edge cases or potential failure types:**  
  - Invalid request body shape
  - Missing expected fields
  - Authentication or signature validation issues if expected externally but not configured here
  - Timeout if the caller expects a synchronous response and the workflow runs too long
- **Sub-workflow reference:**  
  None

### Node: Edit Fields2
- **Type and technical role:** `n8n-nodes-base.set`  
  Used to create, rename, delete, or normalize fields from the webhook payload.
- **Configuration choices:**  
  Parameters are empty in the JSON excerpt, so the exact field mappings are not visible. In practice, this node likely standardizes prompt-related variables for the next code step.
- **Key expressions or variables used:**  
  Not visible in the provided JSON.
- **Input and output connections:**  
  - Input: `Webhook`
  - Output: `Code in JavaScript1`
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`, which supports modern Set-node field editing behavior.
- **Edge cases or potential failure types:**  
  - Referencing fields that do not exist in the incoming payload
  - Overwriting required values accidentally
  - Producing null/empty prompt data
- **Sub-workflow reference:**  
  None

### Node: Code in JavaScript1
- **Type and technical role:** `n8n-nodes-base.code`  
  Runs custom JavaScript to sanitize, enrich, or transform the prompt before submission.
- **Configuration choices:**  
  Parameters are not present in the JSON, so the exact script is unknown. Based on placement and section label, this likely handles prompt sanitization and request preparation.
- **Key expressions or variables used:**  
  Likely accesses `$json` data from `Edit Fields2`; exact variable references are not visible.
- **Input and output connections:**  
  - Input: `Edit Fields2`
  - Output: `HTTP Request3`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`, which corresponds to the newer Code node rather than the legacy Function node.
- **Edge cases or potential failure types:**  
  - JavaScript runtime errors
  - Undefined properties in `$json`
  - Invalid output structure for subsequent HTTP nodes
- **Sub-workflow reference:**  
  None

---

## 2.2 Video Generation Submission & Polling

**Overview:**  
This block submits the prepared prompt to the video-generation service, then repeatedly checks job status until completion. The `If` and `Wait` nodes form the polling loop.

**Nodes Involved:**  
- HTTP Request3
- HTTP Request4
- Wait1
- If1

### Node: HTTP Request3
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the initial request to the external AI video generation service, presumably Seedance based on the workflow title.
- **Configuration choices:**  
  Parameters are not visible, but this node likely performs a POST request with the sanitized prompt and generation options.
- **Key expressions or variables used:**  
  Likely uses values from `$json` generated by `Code in JavaScript1`.
- **Input and output connections:**  
  - Input: `Code in JavaScript1`
  - Output: `HTTP Request4`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.3`, which supports current HTTP Request features such as JSON body handling, headers, and auth modes.
- **Edge cases or potential failure types:**  
  - API authentication failure
  - Invalid prompt payload
  - Rate limiting
  - 4xx/5xx API responses
  - Network timeouts
- **Sub-workflow reference:**  
  None

### Node: HTTP Request4
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Checks the status of the generation job, likely using an ID returned by `HTTP Request3`.
- **Configuration choices:**  
  Exact configuration is not visible. Based on the graph, this node is used both after the initial submission and again when polling repeats.
- **Key expressions or variables used:**  
  Likely references a job/task ID from the earlier HTTP response.
- **Input and output connections:**  
  - Inputs: `HTTP Request3`, `If1` false branch
  - Output: `Wait1`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.3`.
- **Edge cases or potential failure types:**  
  - Missing or malformed job ID
  - Status endpoint returning incomplete data
  - API rate limiting during polling
  - Long-running jobs causing many loop iterations
- **Sub-workflow reference:**  
  None

### Node: Wait1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution between polling attempts to avoid aggressive repeated API calls.
- **Configuration choices:**  
  Parameters are not visible. The node includes a webhook ID, which means it supports resumable execution internally through n8n’s wait mechanism.
- **Key expressions or variables used:**  
  Not visible in the provided JSON.
- **Input and output connections:**  
  - Input: `HTTP Request4`
  - Output: `If1`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`.
- **Edge cases or potential failure types:**  
  - Misconfigured wait duration causing too-frequent or too-slow polling
  - Instance restarts or persistence issues if wait state is not stored reliably
  - Webhook resume problems in self-hosted environments with incorrect base URL config
- **Sub-workflow reference:**  
  None

### Node: If1
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the generation job is complete. If complete, workflow continues to metadata/ticketing; otherwise it loops back to poll again.
- **Configuration choices:**  
  The exact condition is not visible, but it likely checks a status field such as `completed`, `success`, or similar.
- **Key expressions or variables used:**  
  Likely references response JSON from `HTTP Request4`, for example a status property.
- **Input and output connections:**  
  - Input: `Wait1`
  - True output: `Build Clip Metadata & Tags`
  - False output: `HTTP Request4`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Wrong status expression leading to infinite polling
  - Unexpected API status values
  - Null response fields
- **Sub-workflow reference:**  
  None

---

## 2.3 Metadata Construction & Review Ticketing

**Overview:**  
After a successful generation, this block builds structured metadata for the clip and opens a Jira issue so the output can be reviewed or processed by a production team.

**Nodes Involved:**  
- Build Clip Metadata & Tags
- Create an issue1

### Node: Build Clip Metadata & Tags
- **Type and technical role:** `n8n-nodes-base.code`  
  Generates structured metadata, naming, tags, or descriptions derived from the generated video and original prompt.
- **Configuration choices:**  
  Exact JavaScript is not visible, but the node name strongly indicates it assembles clip-level metadata needed by Jira and downstream notifications.
- **Key expressions or variables used:**  
  Likely combines generation result fields, prompt text, asset URLs, and status data via `$json`.
- **Input and output connections:**  
  - Input: `If1` true branch
  - Output: `Create an issue1`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Missing asset URL or generation metadata
  - String formatting errors
  - Producing fields that do not match Jira requirements
- **Sub-workflow reference:**  
  None

### Node: Create an issue1
- **Type and technical role:** `n8n-nodes-base.jira`  
  Creates a Jira issue for review, approval, or post-production workflow tracking.
- **Configuration choices:**  
  Parameters are hidden, but this node likely maps summary, description, project, issue type, and perhaps labels/tags from the previous code node.
- **Key expressions or variables used:**  
  Likely uses generated metadata fields such as title, description, tags, clip URL, and prompt.
- **Input and output connections:**  
  - Input: `Build Clip Metadata & Tags`
  - Output: `Slack: Notify VFX Supervisor1`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Jira authentication failure
  - Invalid project key or issue type
  - Required fields missing
  - Text exceeding Jira field limits
- **Sub-workflow reference:**  
  None

---

## 2.4 Notifications & Delivery

**Overview:**  
This block distributes the result to people and systems. It alerts a VFX supervisor in Slack, reshapes the data for final delivery, optionally fetches or resolves final asset details over HTTP, and sends an email via Gmail.

**Nodes Involved:**  
- Slack: Notify VFX Supervisor1
- Edit Fields3
- HTTP Request2
- Send a message

### Node: Slack: Notify VFX Supervisor1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a Slack message notifying a supervisor that a new AI-generated clip is ready or needs review.
- **Configuration choices:**  
  Parameters are not visible, but likely include channel selection and a formatted message with Jira issue reference and video details.
- **Key expressions or variables used:**  
  Likely references Jira issue key, clip metadata, and asset link fields.
- **Input and output connections:**  
  - Input: `Create an issue1`
  - Output: `Edit Fields3`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Invalid Slack credentials
  - Missing channel permissions
  - Message formatting issues
  - Slack API rate limiting
- **Sub-workflow reference:**  
  None

### Node: Edit Fields3
- **Type and technical role:** `n8n-nodes-base.set`  
  Prepares a clean payload for the final HTTP step and/or email delivery.
- **Configuration choices:**  
  Exact field definitions are hidden. This node probably selects only the values needed for final notification and asset resolution.
- **Key expressions or variables used:**  
  Likely references Slack/Jira output plus original metadata fields.
- **Input and output connections:**  
  - Input: `Slack: Notify VFX Supervisor1`
  - Output: `HTTP Request2`
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - Field name mismatches
  - Accidental loss of values needed by Gmail
- **Sub-workflow reference:**  
  None

### Node: HTTP Request2
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs an additional HTTP call before email delivery. This may retrieve final asset data, a downloadable file URL, or enrich the final message body.
- **Configuration choices:**  
  Parameters are not visible.
- **Key expressions or variables used:**  
  Likely uses fields set by `Edit Fields3`.
- **Input and output connections:**  
  - Input: `Edit Fields3`
  - Output: `Send a message`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.3`.
- **Edge cases or potential failure types:**  
  - Endpoint auth failure
  - Response schema mismatch
  - Broken media URL
- **Sub-workflow reference:**  
  None

### Node: Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the final email notification with the generated clip and related context.
- **Configuration choices:**  
  Parameters are not shown, but likely include recipient, subject, body, and possibly links to the generated asset and Jira issue.
- **Key expressions or variables used:**  
  Likely uses fields from `HTTP Request2`, such as final URL, metadata, recipient information, and review references.
- **Input and output connections:**  
  - Input: `HTTP Request2`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth2 authentication issues
  - Invalid recipient address
  - Message body expression failures
  - Google API quota or sending restrictions
- **Sub-workflow reference:**  
  None

---

## 2.5 Error Handling

**Overview:**  
This branch catches workflow execution failures and forwards them to Slack for operational awareness. It is independent of the main webhook flow.

**Nodes Involved:**  
- On Workflow Error
- Slack: Error Alert

### Node: On Workflow Error
- **Type and technical role:** `n8n-nodes-base.errorTrigger`  
  Dedicated trigger that starts a separate execution when this workflow fails.
- **Configuration choices:**  
  No visible parameters; standard usage is to receive execution error context from n8n automatically.
- **Key expressions or variables used:**  
  Typically exposes execution metadata, node name, and error message through `$json`.
- **Input and output connections:**  
  - Input: none; this is an error trigger
  - Output: `Slack: Error Alert`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Error trigger not firing as expected if workflow or instance-level error settings differ
  - Recursive alerting risk if the Slack node itself errors and is also monitored elsewhere
- **Sub-workflow reference:**  
  None

### Node: Slack: Error Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an operational error message to Slack when a workflow execution fails.
- **Configuration choices:**  
  Parameters are not shown, but it likely posts error details including failed node name and exception text.
- **Key expressions or variables used:**  
  Likely uses error trigger payload values such as execution ID, workflow name, node, and message.
- **Input and output connections:**  
  - Input: `On Workflow Error`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Slack auth or permission failures
  - Overly large error payloads
  - Missing formatting safeguards for nested error objects
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receives inbound prompt submission |  | Edit Fields2 | Section: Trigger & Prompt Sanitisation |
| Edit Fields2 | Set | Normalizes incoming webhook fields | Webhook | Code in JavaScript1 | Section: Trigger & Prompt Sanitisation |
| Code in JavaScript1 | Code | Sanitizes/transforms prompt and request payload | Edit Fields2 | HTTP Request3 | Section: Trigger & Prompt Sanitisation |
| HTTP Request3 | HTTP Request | Submits AI video generation request | Code in JavaScript1 | HTTP Request4 | Section: Video Generation & Polling |
| HTTP Request4 | HTTP Request | Polls video generation status | HTTP Request3, If1 | Wait1 | Section: Video Generation & Polling |
| Wait1 | Wait | Delays between polling attempts | HTTP Request4 | If1 | Section: Video Generation & Polling |
| If1 | If | Decides whether generation is complete or should continue polling | Wait1 | Build Clip Metadata & Tags, HTTP Request4 | Section: Video Generation & Polling |
| Build Clip Metadata & Tags | Code | Builds clip metadata and tagging for tracking/review | If1 | Create an issue1 | Section: Metadata & Review Ticketing |
| Create an issue1 | Jira | Creates Jira review/production issue | Build Clip Metadata & Tags | Slack: Notify VFX Supervisor1 | Section: Metadata & Review Ticketing |
| Slack: Notify VFX Supervisor1 | Slack | Sends Slack review notification | Create an issue1 | Edit Fields3 | Section: Notifications & Delivery |
| Edit Fields3 | Set | Prepares fields for final delivery | Slack: Notify VFX Supervisor1 | HTTP Request2 | Section: Notifications & Delivery |
| HTTP Request2 | HTTP Request | Retrieves or enriches final asset data before email | Edit Fields3 | Send a message | Section: Notifications & Delivery |
| Send a message | Gmail | Sends final email notification | HTTP Request2 |  | Section: Notifications & Delivery |
| On Workflow Error | Error Trigger | Starts error-handling execution |  | Slack: Error Alert | Section: Error Handler |
| Slack: Error Alert | Slack | Sends workflow failure alert to Slack | On Workflow Error |  | Section: Error Handler |
| Overview | Sticky Note | Canvas annotation |  |  |  |
| Section: Error Handler | Sticky Note | Canvas annotation |  |  |  |
| Section: Trigger & Prompt Sanitisation | Sticky Note | Canvas annotation |  |  |  |
| Section: Video Generation & Polling | Sticky Note | Canvas annotation |  |  |  |
| Section: Metadata & Review Ticketing | Sticky Note | Canvas annotation |  |  |  |
| Section: Notifications & Delivery | Sticky Note | Canvas annotation |  |  |  |
| Credentials & Security | Sticky Note | Canvas annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Because the exported JSON contains node structure and connectivity but not the internal parameter details, the steps below reconstruct the workflow architecture and intended behavior. Where exact values are absent, configure them according to your Seedance, Jira, Slack, and Gmail setup.

1. **Create a new workflow**
   - Name it something like: `Generate AI Videos from Prompts with Seedance and Automate Review`.
   - Keep execution order at the default compatible mode if you want to mirror the JSON: `v1`.

2. **Add the main trigger node**
   - Create a **Webhook** node named `Webhook`.
   - Choose the HTTP method expected by your source system, typically `POST`.
   - Set the path to a meaningful endpoint such as `/generate-ai-video`.
   - Decide whether the caller should receive an immediate response or whether you will use a separate response node pattern.
   - If needed, enable authentication or request validation.

3. **Add the first Set node**
   - Create a **Set** node named `Edit Fields2`.
   - Connect `Webhook -> Edit Fields2`.
   - Add fields that normalize the inbound payload, for example:
     - `prompt`
     - `style`
     - `aspect_ratio`
     - `requester_email`
     - `slack_channel`
     - `jira_project`
   - Use expressions referencing webhook data, such as values from request body or query parameters.

4. **Add the prompt-processing code node**
   - Create a **Code** node named `Code in JavaScript1`.
   - Connect `Edit Fields2 -> Code in JavaScript1`.
   - Configure it to:
     - trim whitespace
     - remove unsupported characters if required by the generation API
     - enforce defaults for optional parameters
     - build a clean request object
   - Example responsibilities:
     - validate that `prompt` is not empty
     - shorten prompts if the target API has a character limit
     - create fields like `seedancePayload`

5. **Add the generation submission HTTP node**
   - Create an **HTTP Request** node named `HTTP Request3`.
   - Connect `Code in JavaScript1 -> HTTP Request3`.
   - Configure it for the external video API:
     - Method: usually `POST`
     - URL: Seedance generation endpoint
     - Authentication: API key, bearer token, or custom header
     - Send JSON body: yes
     - Body: map fields from the code node, likely the prepared prompt and generation settings
   - Expect the response to include a generation/job ID.

6. **Add the polling HTTP node**
   - Create an **HTTP Request** node named `HTTP Request4`.
   - Connect `HTTP Request3 -> HTTP Request4`.
   - Configure it to query job status:
     - Method: usually `GET`
     - URL: status endpoint with job ID from `HTTP Request3`
     - Authentication: same as the generation API
   - Ensure the returned payload includes a status field and, when complete, output asset details such as a video URL.

7. **Add the wait node**
   - Create a **Wait** node named `Wait1`.
   - Connect `HTTP Request4 -> Wait1`.
   - Configure a sensible delay, for example 30 seconds or 1 minute between polling attempts.
   - If your jobs are long-running, use a larger interval to reduce API usage.

8. **Add the completion check**
   - Create an **If** node named `If1`.
   - Connect `Wait1 -> If1`.
   - Configure a condition based on the polling response, for example:
     - `status` equals `completed`
     - or `success` equals `true`
   - The true branch should indicate the video is ready.
   - The false branch should indicate polling must continue.

9. **Create the polling loop**
   - Connect the **false** output of `If1` back to `HTTP Request4`.
   - This recreates the retry cycle:
     - check status
     - wait
     - test condition
     - repeat if not finished

10. **Add the metadata-building code node**
    - Create a **Code** node named `Build Clip Metadata & Tags`.
    - Connect the **true** output of `If1 -> Build Clip Metadata & Tags`.
    - Configure it to generate structured values such as:
      - clip title
      - short description
      - tags/labels
      - review summary
      - asset URL
      - original prompt
    - Also format fields so Jira can consume them easily.

11. **Add the Jira node**
    - Create a **Jira** node named `Create an issue1`.
    - Connect `Build Clip Metadata & Tags -> Create an issue1`.
    - Configure Jira credentials:
      - Jira Cloud email + API token, or OAuth2 depending on your environment
    - Set operation to **Create Issue**.
    - Configure required fields:
      - Project key
      - Issue type
      - Summary
      - Description
      - Labels or tags if needed
    - Map values from the metadata node.

12. **Add the Slack notification node for review**
    - Create a **Slack** node named `Slack: Notify VFX Supervisor1`.
    - Connect `Create an issue1 -> Slack: Notify VFX Supervisor1`.
    - Configure Slack credentials:
      - OAuth2 or bot token
    - Set operation to send a message to a channel or user.
    - Include useful details:
      - generated clip title
      - Jira issue key/link
      - video URL
      - prompt summary

13. **Add the second Set node**
    - Create a **Set** node named `Edit Fields3`.
    - Connect `Slack: Notify VFX Supervisor1 -> Edit Fields3`.
    - Prepare the final payload for email delivery, for example:
      - `recipient`
      - `email_subject`
      - `email_body`
      - `video_url`
      - `jira_url`

14. **Add the final HTTP enrichment node**
    - Create an **HTTP Request** node named `HTTP Request2`.
    - Connect `Edit Fields3 -> HTTP Request2`.
    - Use this step if you need to:
      - resolve a signed asset URL
      - fetch additional metadata
      - convert internal references to public links
    - If no enrichment is needed in your implementation, this node can simply call a media endpoint or could be replaced by another formatting step.

15. **Add the Gmail node**
    - Create a **Gmail** node named `Send a message`.
    - Connect `HTTP Request2 -> Send a message`.
    - Configure Gmail credentials using OAuth2.
    - Set operation to send email.
    - Map:
      - To: requester or review distribution list
      - Subject: generated clip title or job summary
      - Body: include video link, Jira link, and prompt details

16. **Add the error trigger**
    - Create an **Error Trigger** node named `On Workflow Error`.
    - This creates a second entry point for failure monitoring.

17. **Add the Slack error node**
    - Create a **Slack** node named `Slack: Error Alert`.
    - Connect `On Workflow Error -> Slack: Error Alert`.
    - Configure it to send operational alerts to a monitoring/admin channel.
    - Include expressions for:
      - workflow name
      - failed node
      - error message
      - execution ID

18. **Add canvas notes if desired**
    - Create sticky notes matching the original workflow sections:
      - `Overview`
      - `Section: Trigger & Prompt Sanitisation`
      - `Section: Video Generation & Polling`
      - `Section: Metadata & Review Ticketing`
      - `Section: Notifications & Delivery`
      - `Section: Error Handler`
      - `Credentials & Security`

19. **Configure credentials**
    - **Seedance / video API**
      - Usually API key or bearer token in the HTTP Request nodes
    - **Jira**
      - Jira Cloud API token or OAuth2
    - **Slack**
      - Bot token or OAuth2 app with permission to post to the target channels
    - **Gmail**
      - Google OAuth2 with send-mail scope

20. **Test the workflow in stages**
    - First test only the webhook input and field normalization.
    - Then test the generation request and confirm the returned job ID.
    - Then test the polling loop with a known successful job.
    - Finally test Jira creation, Slack messaging, and Gmail delivery.

21. **Add safeguards before production use**
    - Add timeout handling for generation jobs that never complete.
    - Add a maximum retry counter to avoid infinite loops.
    - Add validation for required inbound fields.
    - Add explicit error branches for non-200 HTTP responses if needed.

### Input/Output Expectations

- **Inbound webhook input:**  
  Should provide at least a prompt and likely recipient/review metadata.
- **Generation API output:**  
  Must provide a job ID for polling and eventually a completed asset URL or file reference.
- **Jira output:**  
  Should return issue key and URL for inclusion in notifications.
- **Final email output:**  
  Sends a delivery/review message with links and metadata.

### Sub-workflow Setup
This workflow does **not** include any Execute Workflow / sub-workflow node. No sub-workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| No note text was present in the `Overview` sticky note. | Workflow canvas annotation |
| No note text was present in the `Credentials & Security` sticky note. | Workflow canvas annotation |
| The workflow contains section labels but no additional instructional text or external links inside sticky notes. | General canvas structure |
| Title provided by user: `Generate AI videos from prompts with Seedance, Jira, Slack, and Gmail` | Workflow context |
| Internal workflow name in JSON: `Generate AI Videos from Prompts with Seedance and Automate Review` | Workflow metadata |

## Additional implementation notes
- The workflow has **two entry points**:
  1. `Webhook` for normal execution
  2. `On Workflow Error` for failure handling
- The main operational risk is the polling loop. If the `If1` condition is not configured correctly, the workflow can loop indefinitely.
- Since most node parameters are empty in the provided export, exact API endpoints, field mappings, and message templates must be defined manually during reconstruction.
- The workflow is currently marked **inactive** in the JSON (`"active": false`), so it was not active at export time.