Generate multi-pass Seedance AI roto mattes with QC and Nuke handoff

https://n8nworkflows.xyz/workflows/generate-multi-pass-seedance-ai-roto-mattes-with-qc-and-nuke-handoff-14390


# Generate multi-pass Seedance AI roto mattes with QC and Nuke handoff

# 1. Workflow Overview

This workflow automates a roto-matte generation pipeline for VFX/post-production using Seedance AI, with quality control, artist review preparation, and team notifications.

Its main purpose is to:
- receive a roto brief from Google Sheets,
- split the request into multiple roto passes,
- submit each pass to Seedance AI,
- poll until rendering is complete,
- score each pass for QC,
- either escalate failed passes for manual roto or prepare Nuke handoff assets,
- create a Jira review task,
- distribute outputs through email and Telegram,
- notify on workflow-level failures through Slack.

The workflow is organized into the following logical blocks.

## 1.1 Input Reception and Brief Validation
Receives a new row/event from Google Sheets and validates/extracts the roto brief needed for downstream processing.

## 1.2 Multi-Pass Roto Request Generation
Expands the brief into four roto passes, builds API request payloads, and submits them to Seedance AI.

## 1.3 Render Polling and Completion Handling
Stores Seedance job metadata, polls for completion status, and loops with a 20-second wait until each pass is finished.

## 1.4 QC Scoring and Routing
Builds render metadata, calculates or attaches QC scoring, and decides whether the pass is acceptable or requires manual intervention.

## 1.5 Nuke Handoff, Delivery, and Review Creation
Generates a Nuke roto template, downloads the roto pass video, creates a Jira review task, aggregates outputs, and sends downstream notifications.

## 1.6 Team Notifications
Sends QC-failure notices to Slack, emails generated media, and posts final aggregation status to Telegram.

## 1.7 Error Handling
Listens for workflow-level execution failures and sends a Slack alert.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Brief Validation

### Overview
This block starts the workflow when Google Sheets emits a trigger event. It then validates the incoming row content and extracts the roto brief fields required for pass generation.

### Nodes Involved
- Google Sheets Trigger
- Validate & Extract Roto Brief

### Node Details

#### Google Sheets Trigger
- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`; entry-point trigger node for spreadsheet-driven automation.
- **Configuration choices:** The JSON does not expose parameter values, but this node is clearly intended to watch a Google Sheet for newly added or updated roto requests.
- **Key expressions or variables used:** Not visible in the JSON.
- **Input and output connections:**
  - Input: none, this is a trigger.
  - Output: `Validate & Extract Roto Brief`
- **Version-specific requirements:** Type version `1`. Requires a compatible Google Sheets Trigger implementation in the n8n instance.
- **Edge cases or potential failure types:**
  - Google OAuth/authentication failure
  - Sheet not found or permission denied
  - Trigger misconfigured to wrong worksheet/range
  - Row structure changes causing downstream expression/code failures
- **Sub-workflow reference:** None.

#### Validate & Extract Roto Brief
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript logic for validation and field normalization.
- **Configuration choices:** Parameters are empty in JSON, meaning the code body is not present in the export snippet or omitted. Functionally, it should validate mandatory brief fields such as source clip reference, shot ID, pass requirements, thresholds, output destinations, and reviewer info.
- **Key expressions or variables used:** Not visible. Likely produces normalized fields used downstream by:
  - `Fan-Out: 4 Roto Passes`
  - API body construction
  - QC logic
- **Input and output connections:**
  - Input: `Google Sheets Trigger`
  - Output: `Fan-Out: 4 Roto Passes`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Missing required columns in the sheet
  - Malformed JSON/string fields in brief data
  - Empty prompt text or missing media URL
  - Validation exceptions in JavaScript
- **Sub-workflow reference:** None.

---

## 2.2 Multi-Pass Roto Request Generation

### Overview
This block turns a single validated roto brief into four separate roto passes, constructs the Seedance API request body for each one, and starts AI render jobs.

### Nodes Involved
- Fan-Out: 4 Roto Passes
- Build Roto Request Body
- Seedance: Generate Roto Pass
- Merge Roto Job ID + Metadata

### Node Details

#### Fan-Out: 4 Roto Passes
- **Type and technical role:** `n8n-nodes-base.code`; duplicates or expands one request into four items, one per roto pass.
- **Configuration choices:** No raw code is visible, but the node name indicates hardcoded or programmatic generation of exactly four passes.
- **Key expressions or variables used:** Likely generates pass identifiers such as pass number, mask category, layer name, or segmentation target.
- **Input and output connections:**
  - Input: `Validate & Extract Roto Brief`
  - Output: `Build Roto Request Body`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Unexpected input shape causing fan-out logic to fail
  - Duplicate pass naming
  - Wrong assumption that exactly four passes are always needed
- **Sub-workflow reference:** None.

#### Build Roto Request Body
- **Type and technical role:** `n8n-nodes-base.code`; prepares the HTTP payload for Seedance AI.
- **Configuration choices:** Likely maps the normalized brief plus per-pass metadata into the API schema expected by Seedance.
- **Key expressions or variables used:** Probably includes:
  - source media URL or asset identifier
  - pass prompt/instructions
  - frame range
  - output format or render parameters
  - QC-related metadata for later merge
- **Input and output connections:**
  - Input: `Fan-Out: 4 Roto Passes`
  - Output: `Seedance: Generate Roto Pass`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Invalid request structure
  - Required API fields omitted
  - Prompt/body serialization errors
- **Sub-workflow reference:** None.

#### Seedance: Generate Roto Pass
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the generation request to Seedance AI.
- **Configuration choices:** Parameters are not visible, but this should be configured as an HTTP API call, most likely `POST`, with JSON body and authentication headers.
- **Key expressions or variables used:** Should consume the request body assembled in the previous node and return a job ID or render ID.
- **Input and output connections:**
  - Input: `Build Roto Request Body`
  - Output: `Merge Roto Job ID + Metadata`
- **Version-specific requirements:** HTTP Request node version `4.3`.
- **Edge cases or potential failure types:**
  - API authentication failure
  - 4xx validation errors from Seedance
  - 5xx server-side failure
  - request timeout
  - rate limiting if all four passes are started quickly
- **Sub-workflow reference:** None.

#### Merge Roto Job ID + Metadata
- **Type and technical role:** `n8n-nodes-base.code`; combines Seedance response data with original pass context.
- **Configuration choices:** Likely preserves pass name, brief info, and Seedance job ID in one item for polling.
- **Key expressions or variables used:** Probably maps:
  - job ID / task ID from API response
  - shot metadata
  - pass metadata
  - downstream poll URL or status key
- **Input and output connections:**
  - Input: `Seedance: Generate Roto Pass`
  - Output: `Poll: Check Roto Job Status`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Missing job ID in API response
  - malformed JSON response
  - overwriting original metadata accidentally
- **Sub-workflow reference:** None.

---

## 2.3 Render Polling and Completion Handling

### Overview
This block repeatedly checks Seedance render status for each pass until completion. It uses a wait-loop pattern: poll, branch on completion, otherwise wait 20 seconds and poll again.

### Nodes Involved
- Poll: Check Roto Job Status
- Roto Render Complete?
- Wait 20s

### Node Details

#### Poll: Check Roto Job Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; queries Seedance for current render job status.
- **Configuration choices:** Likely `GET` request using a job identifier from upstream metadata.
- **Key expressions or variables used:** Likely references:
  - job ID
  - status endpoint URL
  - auth token/header
- **Input and output connections:**
  - Input: `Merge Roto Job ID + Metadata`, `Wait 20s`
  - Output: `Roto Render Complete?`
- **Version-specific requirements:** HTTP Request node version `4.3`.
- **Edge cases or potential failure types:**
  - polling an invalid/expired job ID
  - API throttling due to repeated polling
  - network timeouts
  - status payload shape changes
- **Sub-workflow reference:** None.

#### Roto Render Complete?
- **Type and technical role:** `n8n-nodes-base.if`; branches based on whether the Seedance job has completed successfully.
- **Configuration choices:** The exact condition is not visible, but the true branch goes to metadata/QC and the false branch goes to the wait node.
- **Key expressions or variables used:** Likely tests status values such as `completed`, `succeeded`, or availability of an output URL.
- **Input and output connections:**
  - Input: `Poll: Check Roto Job Status`
  - True output: `Build Roto Metadata + QC Score`
  - False output: `Wait 20s`
- **Version-specific requirements:** IF node version `2`.
- **Edge cases or potential failure types:**
  - ambiguous statuses such as `failed`, `canceled`, or `partial`
  - condition written only for completion/non-completion and not for terminal failures
  - endless loop if failed states are not trapped separately
- **Sub-workflow reference:** None.

#### Wait 20s
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution before repolling.
- **Configuration choices:** Intended to delay 20 seconds between polling attempts. The actual duration is not visible in parameters, but the node name indicates the intended wait.
- **Key expressions or variables used:** None visible.
- **Input and output connections:**
  - Input: `Roto Render Complete?` false branch
  - Output: `Poll: Check Roto Job Status`
- **Version-specific requirements:** Wait node version `1.1`.
- **Edge cases or potential failure types:**
  - wait not actually configured to 20 seconds despite node name
  - resumed executions disabled or not supported properly in environment
  - large volume of concurrent waits increasing execution load
- **Sub-workflow reference:** None.

---

## 2.4 QC Scoring and Routing

### Overview
Once a pass is complete, this block assembles output metadata and quality information, then decides whether the result is good enough for downstream artist handoff.

### Nodes Involved
- Build Roto Metadata + QC Score
- QC Gate: Passes Threshold?
- Slack: QC Failed — Manual Roto Required

### Node Details

#### Build Roto Metadata + QC Score
- **Type and technical role:** `n8n-nodes-base.code`; computes or formats QC score and final render metadata.
- **Configuration choices:** No code visible. Likely extracts final asset URLs, timing info, pass labels, confidence/quality metrics, and a numeric threshold input for gating.
- **Key expressions or variables used:** Likely produces fields such as:
  - `qc_score`
  - `threshold`
  - `output_url`
  - `pass_name`
  - `shot_id`
- **Input and output connections:**
  - Input: `Roto Render Complete?` true branch
  - Output: `QC Gate: Passes Threshold?`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - missing output URL despite “completed” status
  - score field not numeric
  - threshold missing from brief or hardcoded incorrectly
- **Sub-workflow reference:** None.

#### QC Gate: Passes Threshold?
- **Type and technical role:** `n8n-nodes-base.if`; routes successful renders onward or notifies the team that manual roto is needed.
- **Configuration choices:** The exact condition is not visible, but one branch is success and the other is failure.
- **Key expressions or variables used:** Likely compares `qc_score >= threshold`.
- **Input and output connections:**
  - Input: `Build Roto Metadata + QC Score`
  - True output: `Generate Nuke Roto Template`
  - False output: `Slack: QC Failed — Manual Roto Required`
- **Version-specific requirements:** IF node version `2`.
- **Edge cases or potential failure types:**
  - null/NaN score causing invalid comparison
  - threshold mis-specified as string
  - pass-level QC accepted while sequence-level quality is still poor
- **Sub-workflow reference:** None.

#### Slack: QC Failed — Manual Roto Required
- **Type and technical role:** `n8n-nodes-base.slack`; sends failure/escalation alert.
- **Configuration choices:** Likely posts to a production, comp, or roto operations channel with shot/pass context.
- **Key expressions or variables used:** Should include shot ID, pass name, QC score, and manual action request.
- **Input and output connections:**
  - Input: `QC Gate: Passes Threshold?` false branch
  - Output: none
- **Version-specific requirements:** Slack node version `2.3`.
- **Edge cases or potential failure types:**
  - Slack auth/token issue
  - invalid channel ID
  - message formatting failure due to undefined variables
- **Sub-workflow reference:** None.

---

## 2.5 Nuke Handoff, Delivery, and Review Creation

### Overview
This block prepares downstream artist deliverables when QC passes. It generates a Nuke template, downloads the roto result, creates a Jira review task, aggregates all passes, and then emits final notifications.

### Nodes Involved
- Generate Nuke Roto Template
- Download Roto Pass Video
- Send a message
- Jira: Create Review Task
- Aggregate All Roto Passes
- Send Telegram1

### Node Details

#### Generate Nuke Roto Template
- **Type and technical role:** `n8n-nodes-base.code`; creates a Nuke-compatible handoff structure, likely text/template content or metadata for compositors.
- **Configuration choices:** No code visible. It likely generates file naming, layer references, shot metadata, and pointers to rendered pass outputs.
- **Key expressions or variables used:** Likely includes:
  - shot ID
  - pass names
  - output asset URLs
  - project path conventions
- **Input and output connections:**
  - Input: `QC Gate: Passes Threshold?` true branch
  - Outputs:
    - `Download Roto Pass Video`
    - `Jira: Create Review Task`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - malformed template text
  - invalid path conventions for Nuke pipeline
  - missing fields needed by Jira or delivery steps
- **Sub-workflow reference:** None.

#### Download Roto Pass Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`; retrieves the rendered video/binary output from Seedance or storage.
- **Configuration choices:** Likely configured to download binary data from the final output URL.
- **Key expressions or variables used:** Should use a completed render URL generated in earlier steps.
- **Input and output connections:**
  - Input: `Generate Nuke Roto Template`
  - Output: `Send a message`
- **Version-specific requirements:** HTTP Request node version `4.3`.
- **Edge cases or potential failure types:**
  - URL expired or unauthorized
  - binary download not enabled
  - large file size causing memory/time issues
- **Sub-workflow reference:** None.

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an email, likely with roto review/handoff information and possibly the downloaded asset.
- **Configuration choices:** Exact fields are not visible. The naming suggests a default Gmail send-message operation rather than a renamed business-specific label.
- **Key expressions or variables used:** Probably includes recipient, subject, shot summary, and attachment/binary reference.
- **Input and output connections:**
  - Input: `Download Roto Pass Video`
  - Output: none
- **Version-specific requirements:** Gmail node version `2.2`.
- **Edge cases or potential failure types:**
  - Gmail OAuth issue
  - attachment size limits
  - recipient not defined in source data
  - HTML/body expression failures
- **Sub-workflow reference:** None.

#### Jira: Create Review Task
- **Type and technical role:** `n8n-nodes-base.jira`; creates an issue for artist or supervisor review.
- **Configuration choices:** Exact operation is not visible, but based on naming it should create a new review task/ticket populated with shot and roto details.
- **Key expressions or variables used:** Likely includes:
  - issue summary
  - description with shot/pass info
  - assignee or project key
  - links to outputs and Nuke template
- **Input and output connections:**
  - Input: `Generate Nuke Roto Template`
  - Output: `Aggregate All Roto Passes`
- **Version-specific requirements:** Jira node version `1`.
- **Edge cases or potential failure types:**
  - Jira auth/project permission issue
  - invalid issue type or project key
  - field schema mismatch in Jira instance
- **Sub-workflow reference:** None.

#### Aggregate All Roto Passes
- **Type and technical role:** `n8n-nodes-base.code`; consolidates results, likely after Jira creation, into a final summary object.
- **Configuration choices:** No code visible. Despite its name, the current wiring suggests it aggregates the successful review-task branch output rather than explicitly joining four independent branches with a Merge node.
- **Key expressions or variables used:** Possibly compiles:
  - all pass statuses
  - review task ID
  - Nuke template references
  - final delivery summary
- **Input and output connections:**
  - Input: `Jira: Create Review Task`
  - Output: `Send Telegram1`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - name implies full aggregation but topology does not enforce waiting for all four passes
  - code may rely on execution data assumptions not guaranteed by current structure
  - missing pass records if items are processed independently
- **Sub-workflow reference:** None.

#### Send Telegram1
- **Type and technical role:** `n8n-nodes-base.telegram`; sends a Telegram notification with final summary or delivery confirmation.
- **Configuration choices:** Not visible. Likely posts to a team/channel/chat.
- **Key expressions or variables used:** Should include task ID, shot summary, and delivery status.
- **Input and output connections:**
  - Input: `Aggregate All Roto Passes`
  - Output: none
- **Version-specific requirements:** Telegram node version `1.1`.
- **Edge cases or potential failure types:**
  - bot token/chat ID issues
  - message length/formatting errors
  - missing aggregated values
- **Sub-workflow reference:** None.

---

## 2.6 Team Notifications

### Overview
Notification logic is distributed rather than centralized. Different destinations are used depending on the event type: Slack for QC failures and workflow errors, Gmail for asset delivery, and Telegram for final summary distribution.

### Nodes Involved
- Slack: QC Failed — Manual Roto Required
- Send a message
- Send Telegram1
- Slack: Error Alert

### Node Details
These nodes are already described in their primary functional blocks. Their notification roles are:
- **Slack: QC Failed — Manual Roto Required:** pass-level operational escalation.
- **Send a message:** direct email delivery/review communication.
- **Send Telegram1:** summary notification after aggregation.
- **Slack: Error Alert:** workflow-level failure notification.

---

## 2.7 Error Handling

### Overview
This independent branch catches execution-level errors and alerts Slack so operators can intervene.

### Nodes Involved
- On Workflow Error
- Slack: Error Alert

### Node Details

#### On Workflow Error
- **Type and technical role:** `n8n-nodes-base.errorTrigger`; special trigger that fires when the workflow errors.
- **Configuration choices:** Standard error-trigger behavior; no parameters visible.
- **Key expressions or variables used:** Typically exposes workflow name, execution ID, error message, failing node, and stack/context.
- **Input and output connections:**
  - Input: none, trigger node.
  - Output: `Slack: Error Alert`
- **Version-specific requirements:** Error Trigger version `1`.
- **Edge cases or potential failure types:**
  - may not catch failures in all operational contexts depending on n8n deployment/config
  - recursive alerting risk if Slack node itself fails and error workflows are globally chained
- **Sub-workflow reference:** None.

#### Slack: Error Alert
- **Type and technical role:** `n8n-nodes-base.slack`; sends a workflow error alert.
- **Configuration choices:** Likely posts execution diagnostics to an operations channel.
- **Key expressions or variables used:** Should include error details from the trigger payload.
- **Input and output connections:**
  - Input: `On Workflow Error`
  - Output: none
- **Version-specific requirements:** Slack node version `2.3`.
- **Edge cases or potential failure types:**
  - Slack auth problems
  - missing trigger fields if message template assumes specific error payload shape
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | Google Sheets Trigger | Entry point from spreadsheet event |  | Validate & Extract Roto Brief | Section: Trigger & Validation |
| Validate & Extract Roto Brief | Code | Validate sheet row and normalize roto brief | Google Sheets Trigger | Fan-Out: 4 Roto Passes | Section: Trigger & Validation |
| Fan-Out: 4 Roto Passes | Code | Expand one request into four roto pass items | Validate & Extract Roto Brief | Build Roto Request Body | Section: Pass Generation & AI Render |
| Build Roto Request Body | Code | Construct Seedance API request payload | Fan-Out: 4 Roto Passes | Seedance: Generate Roto Pass | Section: Pass Generation & AI Render |
| Seedance: Generate Roto Pass | HTTP Request | Submit roto generation request to Seedance AI | Build Roto Request Body | Merge Roto Job ID + Metadata | Section: Pass Generation & AI Render |
| Merge Roto Job ID + Metadata | Code | Attach returned job ID to pass metadata | Seedance: Generate Roto Pass | Poll: Check Roto Job Status | Section: Polling & Completion |
| Poll: Check Roto Job Status | HTTP Request | Poll Seedance job status | Merge Roto Job ID + Metadata; Wait 20s | Roto Render Complete? | Section: Polling & Completion |
| Roto Render Complete? | If | Branch on whether render is complete | Poll: Check Roto Job Status | Build Roto Metadata + QC Score; Wait 20s | Section: Polling & Completion |
| Wait 20s | Wait | Delay before next polling attempt | Roto Render Complete? | Poll: Check Roto Job Status | Section: Polling & Completion |
| Build Roto Metadata + QC Score | Code | Prepare completed render metadata and QC score | Roto Render Complete? | QC Gate: Passes Threshold? | Section: QC Gate & Nuke Script |
| QC Gate: Passes Threshold? | If | Route pass based on QC threshold | Build Roto Metadata + QC Score | Generate Nuke Roto Template; Slack: QC Failed — Manual Roto Required | Section: QC Gate & Nuke Script |
| Generate Nuke Roto Template | Code | Build Nuke handoff template/metadata | QC Gate: Passes Threshold? | Download Roto Pass Video; Jira: Create Review Task | Section: QC Gate & Nuke Script |
| Slack: QC Failed — Manual Roto Required | Slack | Notify team that manual roto is required | QC Gate: Passes Threshold? |  | Section: QC Gate & Nuke Script |
| Download Roto Pass Video | HTTP Request | Download completed roto media | Generate Nuke Roto Template | Send a message | Section: Delivery & Task Creation |
| Send a message | Gmail | Email roto output/review information | Download Roto Pass Video |  | Section: Delivery & Task Creation |
| Jira: Create Review Task | Jira | Create review task for artists/supervisors | Generate Nuke Roto Template | Aggregate All Roto Passes | Section: Delivery & Task Creation |
| Aggregate All Roto Passes | Code | Consolidate pass/task results into final summary | Jira: Create Review Task | Send Telegram1 | Section: Delivery & Task Creation |
| Send Telegram1 | Telegram | Send final team notification | Aggregate All Roto Passes |  | Section: Team Notifications |
| On Workflow Error | Error Trigger | Catch workflow execution failures |  | Slack: Error Alert | Section: Error Handler |
| Slack: Error Alert | Slack | Notify operations of workflow errors | On Workflow Error |  | Section: Error Handler |
| 📋 Overview | Sticky Note | Canvas annotation |  |  |  |
| Section: Trigger & Validation | Sticky Note | Canvas annotation |  |  |  |
| Section: Pass Generation & AI Render | Sticky Note | Canvas annotation |  |  |  |
| Section: Polling & Completion | Sticky Note | Canvas annotation |  |  |  |
| Section: QC Gate & Nuke Script | Sticky Note | Canvas annotation |  |  |  |
| Section: Delivery & Task Creation | Sticky Note | Canvas annotation |  |  |  |
| Section: Team Notifications | Sticky Note | Canvas annotation |  |  |  |
| Security Notes | Sticky Note | Canvas annotation |  |  |  |
| Section: Error Handler | Sticky Note | Canvas annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild plan based on the topology and naming present in the workflow.

## 4.1 Create the trigger and validation block

1. **Create a `Google Sheets Trigger` node**
   - Node type: **Google Sheets Trigger**
   - Configure Google credentials using OAuth2.
   - Select the spreadsheet and worksheet that will hold incoming roto requests.
   - Configure the trigger event appropriate to your intake design, typically:
     - new row added, or
     - row updated.
   - Ensure the sheet contains columns for at least:
     - shot identifier
     - source media URL or asset path
     - roto brief / prompt
     - QC threshold
     - recipients / reviewer info
     - project or Jira metadata

2. **Create a `Code` node named `Validate & Extract Roto Brief`**
   - Connect `Google Sheets Trigger` → `Validate & Extract Roto Brief`.
   - Implement JavaScript to:
     - verify required fields exist,
     - trim and normalize strings,
     - parse any numeric thresholds,
     - reject empty source URL/prompt values,
     - output a single normalized item.
   - Recommended output fields:
     - `shot_id`
     - `source_url`
     - `roto_brief`
     - `qc_threshold`
     - `project_key`
     - `email_to`
     - `telegram_chat_id`
     - optional review metadata

## 4.2 Create the multi-pass generation branch

3. **Create a `Code` node named `Fan-Out: 4 Roto Passes`**
   - Connect `Validate & Extract Roto Brief` → `Fan-Out: 4 Roto Passes`.
   - Configure code to emit **four items** from one validated brief.
   - For each output item, include:
     - all original brief fields,
     - `pass_index` from 1 to 4,
     - `pass_name`,
     - any pass-specific prompt or mask target.

4. **Create a `Code` node named `Build Roto Request Body`**
   - Connect `Fan-Out: 4 Roto Passes` → `Build Roto Request Body`.
   - Build the Seedance API payload for each pass.
   - Typical structure should include:
     - source clip reference
     - prompt/instructions
     - pass metadata
     - desired output type
     - optional callback/job metadata
   - Store the body in a field like `request_body`.

5. **Create an `HTTP Request` node named `Seedance: Generate Roto Pass`**
   - Connect `Build Roto Request Body` → `Seedance: Generate Roto Pass`.
   - Configure:
     - Method: usually `POST`
     - URL: Seedance generation endpoint
     - Authentication: header or credential-supported auth
     - Content-Type: `application/json`
     - Request body: use the field prepared in the previous node
   - Save response data, especially:
     - job ID / render ID
     - status endpoint reference if provided

6. **Create a `Code` node named `Merge Roto Job ID + Metadata`**
   - Connect `Seedance: Generate Roto Pass` → `Merge Roto Job ID + Metadata`.
   - Merge the API response with original item data.
   - Ensure the output contains:
     - `job_id`
     - `shot_id`
     - `pass_name`
     - `source_url`
     - `qc_threshold`
     - any other original context needed later

## 4.3 Create the polling loop

7. **Create an `HTTP Request` node named `Poll: Check Roto Job Status`**
   - Connect `Merge Roto Job ID + Metadata` → `Poll: Check Roto Job Status`.
   - This node will also later receive input from the wait loop.
   - Configure:
     - Method: usually `GET`
     - URL: Seedance status endpoint, using `job_id`
     - Authentication: same Seedance auth mechanism
   - Response should include a render state and output URL when done.

8. **Create an `If` node named `Roto Render Complete?`**
   - Connect `Poll: Check Roto Job Status` → `Roto Render Complete?`.
   - Configure condition such as:
     - status equals `completed`, `succeeded`, or equivalent.
   - If possible, explicitly handle terminal failed states in your code or by extending this logic.

9. **Create a `Wait` node named `Wait 20s`**
   - Connect the **false** output of `Roto Render Complete?` → `Wait 20s`.
   - Configure the wait duration to **20 seconds**.
   - Connect `Wait 20s` → `Poll: Check Roto Job Status`.
   - This creates the polling loop.

## 4.4 Create QC and success/failure routing

10. **Create a `Code` node named `Build Roto Metadata + QC Score`**
    - Connect the **true** output of `Roto Render Complete?` → `Build Roto Metadata + QC Score`.
    - In code:
      - extract final render URL,
      - preserve pass metadata,
      - compute or assign a `qc_score`,
      - carry forward `qc_threshold`.
    - Recommended output:
      - `output_url`
      - `qc_score`
      - `qc_threshold`
      - `shot_id`
      - `pass_name`

11. **Create an `If` node named `QC Gate: Passes Threshold?`**
    - Connect `Build Roto Metadata + QC Score` → `QC Gate: Passes Threshold?`.
    - Configure condition:
      - `qc_score >= qc_threshold`

12. **Create a `Slack` node named `Slack: QC Failed — Manual Roto Required`**
    - Connect the **false** branch of `QC Gate: Passes Threshold?` → this Slack node.
    - Configure Slack credentials.
    - Set target channel for manual roto escalation.
    - Message should include:
      - shot ID
      - pass name
      - QC score
      - threshold
      - output or diagnostic link if available

## 4.5 Create Nuke handoff and delivery nodes

13. **Create a `Code` node named `Generate Nuke Roto Template`**
    - Connect the **true** branch of `QC Gate: Passes Threshold?` → `Generate Nuke Roto Template`.
    - Build a Nuke handoff payload or text template containing:
      - shot metadata
      - pass labels
      - output media link(s)
      - path conventions
      - artist instructions if needed

14. **Create an `HTTP Request` node named `Download Roto Pass Video`**
    - Connect `Generate Nuke Roto Template` → `Download Roto Pass Video`.
    - Configure:
      - Method: `GET`
      - URL: final render/output URL
      - Response format: file/binary
    - Store the result in a binary property for email delivery.

15. **Create a `Gmail` node named `Send a message`**
    - Connect `Download Roto Pass Video` → `Send a message`.
    - Configure Gmail OAuth2 credentials.
    - Set:
      - recipient from incoming data
      - subject referencing shot ID/pass
      - body with review details
      - optional binary attachment from the download node
    - Be aware of Gmail attachment size limits.

16. **Create a `Jira` node named `Jira: Create Review Task`**
    - Connect `Generate Nuke Roto Template` → `Jira: Create Review Task`.
    - Configure Jira credentials.
    - Set operation to create an issue.
    - Populate:
      - project key
      - issue type
      - summary
      - description with links to render output and Nuke handoff info
      - assignee if required

17. **Create a `Code` node named `Aggregate All Roto Passes`**
    - Connect `Jira: Create Review Task` → `Aggregate All Roto Passes`.
    - Implement logic to summarize pass outputs and created review task data.
    - Important design note: if you truly need synchronization across all four passes, add a proper merge/aggregation design rather than relying on item flow alone.
    - Recommended summary fields:
      - `shot_id`
      - list of completed passes
      - Jira issue key
      - final delivery links

18. **Create a `Telegram` node named `Send Telegram1`**
    - Connect `Aggregate All Roto Passes` → `Send Telegram1`.
    - Configure Telegram bot credentials and chat ID.
    - Send final status summary.

## 4.6 Create workflow-level error handling

19. **Create an `Error Trigger` node named `On Workflow Error`**
    - This is an independent entry point.

20. **Create a `Slack` node named `Slack: Error Alert`**
    - Connect `On Workflow Error` → `Slack: Error Alert`.
    - Configure Slack credentials.
    - Message should include:
      - workflow name
      - failing node
      - execution ID
      - error text

## 4.7 Add canvas documentation notes

21. **Add Sticky Notes** matching the existing sections:
   - `📋 Overview`
   - `Section: Trigger & Validation`
   - `Section: Pass Generation & AI Render`
   - `Section: Polling & Completion`
   - `Section: QC Gate & Nuke Script`
   - `Section: Delivery & Task Creation`
   - `Section: Team Notifications`
   - `Security Notes`
   - `Section: Error Handler`

## 4.8 Credential setup checklist

22. Configure these credentials before activation:
   - **Google Sheets OAuth2** for `Google Sheets Trigger`
   - **Seedance API authentication** for both HTTP Request nodes
   - **Slack credentials** for both Slack nodes
   - **Gmail OAuth2** for `Send a message`
   - **Jira credentials** for `Jira: Create Review Task`
   - **Telegram bot credentials** for `Send Telegram1`

## 4.9 Important implementation constraints

23. Ensure the workflow handles:
   - failed Seedance jobs as terminal states,
   - max polling attempts or timeout ceilings,
   - proper aggregation of all four passes if sequence-level completion matters,
   - attachment/file-size limits on email delivery,
   - idempotency if Google Sheets triggers duplicate events.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| No sticky note bodies were populated in the provided workflow JSON. Only sticky note titles are available. | Canvas documentation |
| The workflow title supplied by the user differs from the internal workflow name in JSON. Internal name: “Seedance AI Roto Matte Generator with Multi-Pass Extraction, QC Automation, and Artist Review Pipeline”. | Naming/reference |
| The node `Aggregate All Roto Passes` suggests full four-pass consolidation, but the current topology does not include an explicit Merge/Combine synchronization node. Review this if all passes must finish before Jira/Telegram summary. | Design caution |
| The wait-loop design is suitable for asynchronous render polling, but a max-attempt or timeout guard is strongly recommended to avoid infinite polling. | Reliability note |
| Security/authentication details are not visible in the JSON parameters and must be supplied manually during implementation. | Deployment note |