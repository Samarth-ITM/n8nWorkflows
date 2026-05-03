Generate Real Estate Research Reports With Exa AI, PandaDoc and Instantly AI

https://n8nworkflows.xyz/workflows/generate-real-estate-research-reports-with-exa-ai--pandadoc-and-instantly-ai-14053


# Generate Real Estate Research Reports With Exa AI, PandaDoc and Instantly AI

# 1. Workflow Overview

This workflow automates the creation and distribution of real estate research reports. It runs on a schedule, generates market research through Exa AI, turns that research into a branded report through PandaDoc, sends the generated report for human review in Slack, then distributes it to a contact list from Google Sheets through Instantly AI, while writing status updates back to the sheet and sending a final Slack confirmation.

Typical use cases include:

- Recurring investor market updates
- Buyer-facing area snapshots
- Listing context reports
- Periodic client research packs with a fixed report structure

## 1.1 Trigger and Research Task Creation

The workflow starts on a schedule and creates an Exa AI research task. It then repeatedly checks Exa until the task is completed.

## 1.2 Report Generation and Status Polling

Once Exa research is complete, the workflow submits the structured result to PandaDoc to generate a document. It then polls PandaDoc until the document is ready.

## 1.3 Approval and Contact Retrieval

After PandaDoc finishes, the workflow retrieves the generated document via URL, posts it to Slack for approval, and then fetches contact records from Google Sheets.

## 1.4 Batch Distribution via Instantly AI

Contacts are split into batches, prepared for upload, sent to Instantly AI, and throttled through a wait loop to stay within API limits.

## 1.5 Status Logging and Completion Notification

After batch preparation, the workflow updates send status in Google Sheets, aggregates final run results, and posts a completion message in Slack.

---

# 2. Block-by-Block Analysis

## Block 1 — Workflow Start and Exa Research Polling

### Overview

This block initiates the workflow and manages asynchronous research generation through Exa AI. It uses a polling loop with an If + Wait pattern so downstream processing only starts after the Exa task has fully completed.

### Nodes Involved

- Schedule Trigger
- Create a research task
- Get a research task
- If1
- Wait1

### Node Details

#### 1. Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry point for recurring execution.
- **Configuration choices:** Configured with an interval rule. The JSON shows a default interval structure, so the exact cadence must be set manually in n8n.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Create a research task**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failure types:**
  - Workflow may run too frequently, causing unnecessary API cost.
  - Time-based execution drift or timezone misunderstanding.
- **Sub-workflow reference:** None.

#### 2. Create a research task
- **Type and role:** `n8n-nodes-exa-official.exa`; creates an Exa research job.
- **Configuration choices:** Resource = `research`, Operation = `createTask`.
- **Key expressions or variables used:** None visible in the exported JSON; task payload details are not included and must be configured manually.
- **Input and output connections:** Input from **Schedule Trigger**; output to **Get a research task**.
- **Version-specific requirements:** Exa node type version `1`.
- **Edge cases / failure types:**
  - Missing Exa credentials.
  - Invalid task definition or unsupported query format.
  - Exa usage limits or billing issues.
- **Sub-workflow reference:** None.

#### 3. Get a research task
- **Type and role:** `n8n-nodes-exa-official.exa`; fetches the current status and result of the previously created Exa task.
- **Configuration choices:** Resource = `research`, Operation = `getTask`.
- **Key expressions or variables used:** Implicitly depends on task identifier from the prior Exa node.
- **Input and output connections:** Input from **Create a research task** and **Wait1**; output to **If1**.
- **Version-specific requirements:** Exa node type version `1`.
- **Edge cases / failure types:**
  - Missing or malformed task ID.
  - Task not found.
  - Temporary Exa API failure.
- **Sub-workflow reference:** None.

#### 4. If1
- **Type and role:** `n8n-nodes-base.if`; checks whether the Exa task is complete.
- **Configuration choices:** Condition is `{{$json.status}} equals "completed"`.
- **Key expressions or variables used:** `{{$json.status}}`
- **Input and output connections:** Input from **Get a research task**; true branch to **PandaDoc (Generate Presentation)**, false branch to **Wait1**.
- **Version-specific requirements:** Type version `2.3`, condition engine version 3.
- **Edge cases / failure types:**
  - If Exa returns statuses other than `completed`/`pending`, they fall into the false branch.
  - Missing `status` field can cause logic errors.
- **Sub-workflow reference:** None.

#### 5. Wait1
- **Type and role:** `n8n-nodes-base.wait`; pauses before polling Exa again.
- **Configuration choices:** Wait node present but no explicit timing appears in the JSON export; delay settings must be configured in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from false branch of **If1**; output back to **Get a research task**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failure types:**
  - No meaningful wait duration could create aggressive polling.
  - Long waits increase total run time.
- **Sub-workflow reference:** None.

---

## Block 2 — PandaDoc Report Generation and Polling

### Overview

This block takes completed research and submits it to PandaDoc for document generation. Because PandaDoc generation is asynchronous, a second If + Wait loop repeatedly checks document status until the final report is available.

### Nodes Involved

- PandaDoc (Generate Presentation)
- PandaDoc (checkPresentationStatus)
- If
- Wait

### Node Details

#### 6. PandaDoc (Generate Presentation)
- **Type and role:** `n8n-nodes-base.httpRequest`; sends a POST request to PandaDoc to generate the report document from a template.
- **Configuration choices:** HTTP method is `POST`. The exact URL, headers, authentication, and request body are not visible in the JSON and must be supplied manually.
- **Key expressions or variables used:** Not exposed in the export, but this node is expected to map Exa research fields into PandaDoc template variables.
- **Input and output connections:** Input from true branch of **If1**; output to **PandaDoc (checkPresentationStatus)**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failure types:**
  - PandaDoc authentication errors.
  - Invalid template ID.
  - Missing variables required by the template.
  - Payload schema mismatch.
- **Sub-workflow reference:** None.

#### 7. PandaDoc (checkPresentationStatus)
- **Type and role:** `n8n-nodes-base.httpRequest`; checks the generation status of the PandaDoc document.
- **Configuration choices:** HTTP request with unspecified method in export, likely GET. Must be configured to query PandaDoc document status endpoint.
- **Key expressions or variables used:** Expected to reference the document ID returned by **PandaDoc (Generate Presentation)**.
- **Input and output connections:** Input from **PandaDoc (Generate Presentation)** and **Wait**; output to **If**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failure types:**
  - Missing document ID.
  - PandaDoc returns transient status values not handled explicitly.
  - API throttling or timeout.
- **Sub-workflow reference:** None.

#### 8. If
- **Type and role:** `n8n-nodes-base.if`; checks whether the PandaDoc document is no longer pending.
- **Configuration choices:** Condition is `{{$json.status}} notEquals "pending"`.
- **Key expressions or variables used:** `{{$json.status}}`
- **Input and output connections:** Input from **PandaDoc (checkPresentationStatus)**; true branch to **HTTP Request**, false branch to **Wait**.
- **Version-specific requirements:** Type version `2.3`, condition engine version 3.
- **Edge cases / failure types:**
  - Any non-`pending` status passes, including possible failure statuses such as `error` or `failed`.
  - This is an important logic caveat: the node does not explicitly require `completed`.
- **Sub-workflow reference:** None.

#### 9. Wait
- **Type and role:** `n8n-nodes-base.wait`; pauses before rechecking PandaDoc generation status.
- **Configuration choices:** Wait node exists but no explicit delay appears in the export.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from false branch of **If**; output to **PandaDoc (checkPresentationStatus)**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failure types:**
  - Missing wait duration can create rapid polling.
  - A long wait adds latency to report delivery.
- **Sub-workflow reference:** None.

---

## Block 3 — Report Retrieval and Slack Approval

### Overview

Once PandaDoc indicates the document is no longer pending, this block retrieves the generated artifact from a path returned in the previous response and sends it to Slack for review. In the current implementation, the Slack node is followed immediately by contact retrieval, so approval appears to be a notification step rather than an enforced interactive approval gate.

### Nodes Involved

- HTTP Request
- Pending Approval

### Node Details

#### 10. HTTP Request
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches the final document or related payload from a URL supplied by PandaDoc output.
- **Configuration choices:** URL is dynamically set to `{{$json.data.path}}`.
- **Key expressions or variables used:** `{{$json.data.path}}`
- **Input and output connections:** Input from true branch of **If**; output to **Pending Approval**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases / failure types:**
  - `data.path` may not exist if PandaDoc returns a different schema.
  - URL may require auth headers not included here.
  - File retrieval may fail due to permissions or expired links.
- **Sub-workflow reference:** None.

#### 11. Pending Approval
- **Type and role:** `n8n-nodes-base.slack`; posts a Slack message to notify reviewers.
- **Configuration choices:** Slack node is set with `select = user`, but the selected user value is empty in the export. The node must be configured with a Slack user, channel, or equivalent destination depending on the operation chosen in the UI.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** Input from **HTTP Request**; output to **Fetch contact**.
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases / failure types:**
  - Missing Slack credentials.
  - Empty recipient configuration.
  - Message posts successfully but does not actually pause for approval.
- **Sub-workflow reference:** None.

**Important behavior note:**  
Despite the node name “Pending Approval,” there is no interactive Slack approval branch, wait-for-response node, or conditional approval check in the workflow JSON. After posting to Slack, the workflow proceeds automatically to contact fetching.

---

## Block 4 — Contact Retrieval and Instantly Batch Upload Loop

### Overview

This block reads recipients from Google Sheets, batches them, uploads them to Instantly AI, and applies a wait loop to reduce the risk of hitting Instantly rate limits.

### Nodes Involved

- Fetch contact
- Batch Instantly Upload
- Instantly Add Lead
- Wait for Instantly Rate Limit

### Node Details

#### 12. Fetch contact
- **Type and role:** `n8n-nodes-base.googleSheets`; reads the recipient list from Google Sheets.
- **Configuration choices:** Uses selected `documentId` and `sheetName`, both blank in the export and requiring manual selection. Google Sheets OAuth2 credential is attached.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** Input from **Pending Approval**; output to **Batch Instantly Upload**.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failure types:**
  - Google OAuth misconfiguration.
  - Wrong spreadsheet or sheet selection.
  - Missing expected columns such as email, name, or campaign fields.
- **Sub-workflow reference:** None.

#### 13. Batch Instantly Upload
- **Type and role:** `n8n-nodes-base.splitInBatches`; iterates through contacts in batches.
- **Configuration choices:** Batch options are empty in the export, so the batch size must be set in the editor if needed.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Fetch contact** and loop-back from **Wait for Instantly Rate Limit**; first output to **Prep Batch Upload Data**, second output to **Instantly Add Lead**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failure types:**
  - If batch size is not configured as intended, performance and rate limiting may suffer.
  - Empty input set results in no distribution.
- **Sub-workflow reference:** None.

#### 14. Instantly Add Lead
- **Type and role:** `n8n-nodes-base.httpRequest`; sends contact data to Instantly AI.
- **Configuration choices:** HTTP Request node with unspecified details in the export; must be configured with Instantly API endpoint, auth headers, and payload mapping.
- **Key expressions or variables used:** Not visible, but should map recipient and report data into Instantly’s lead or campaign endpoint.
- **Input and output connections:** Input from **Batch Instantly Upload**; output to **Wait for Instantly Rate Limit**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failure types:**
  - Invalid Instantly API key.
  - Lead payload schema mismatch.
  - Campaign or sequence not configured.
  - Rate limit or duplicate lead rejection.
- **Sub-workflow reference:** None.

#### 15. Wait for Instantly Rate Limit
- **Type and role:** `n8n-nodes-base.wait`; delays between batches to comply with Instantly limits.
- **Configuration choices:** Wait duration is not visible in export and must be configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Instantly Add Lead**; output loops back to **Batch Instantly Upload**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failure types:**
  - Too short a delay may still hit rate limits.
  - Too long a delay may make large runs slow.
- **Sub-workflow reference:** None.

---

## Block 5 — Batch Preparation, Sheet Updates, Aggregation, and Final Slack Notification

### Overview

This block prepares upload metadata for each batch, writes send status back to Google Sheets, aggregates batch outcomes, and posts a final Slack completion message.

### Nodes Involved

- Prep Batch Upload Data
- Update Sheet Status
- Aggregate Final Results
- Send a message1

### Node Details

#### 16. Prep Batch Upload Data
- **Type and role:** `n8n-nodes-base.code`; transforms batch data into a structure suitable for sheet updates or downstream summary logic.
- **Configuration choices:** No code content is included in the export.
- **Key expressions or variables used:** Unknown from export; likely consumes current batch items and possibly report metadata.
- **Input and output connections:** Input from **Batch Instantly Upload**; output to **Update Sheet Status**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failure types:**
  - Empty or invalid JavaScript code.
  - Referencing fields that do not exist in Google Sheets output.
  - Mismatch between batch data and update target columns.
- **Sub-workflow reference:** None.

#### 17. Update Sheet Status
- **Type and role:** `n8n-nodes-base.googleSheets`; updates contact rows with send status.
- **Configuration choices:** Uses selected `documentId` and `sheetName`, blank in export. Google Sheets OAuth2 credential is attached.
- **Key expressions or variables used:** Not visible, but should map row identifiers and status values.
- **Input and output connections:** Input from **Prep Batch Upload Data**; output to **Aggregate Final Results**.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failure types:**
  - No row key to match records.
  - Partial updates if sheet schema changed.
  - Permission issues on spreadsheet.
- **Sub-workflow reference:** None.

#### 18. Aggregate Final Results
- **Type and role:** `n8n-nodes-base.code`; compiles overall distribution results.
- **Configuration choices:** No code content is included in the export.
- **Key expressions or variables used:** Unknown from export; likely counts sent, failed, or processed records.
- **Input and output connections:** Input from **Update Sheet Status**; output to **Send a message1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failure types:**
  - Aggregation logic may be incorrect if input structure changes.
  - No items to aggregate on empty runs.
- **Sub-workflow reference:** None.

#### 19. Send a message1
- **Type and role:** `n8n-nodes-base.slack`; posts final completion notification.
- **Configuration choices:** Slack node with generic other options; exact target and message must be configured manually.
- **Key expressions or variables used:** Not visible, but should typically include summary fields from **Aggregate Final Results**.
- **Input and output connections:** Input from **Aggregate Final Results**; no downstream nodes.
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases / failure types:**
  - Missing Slack credentials or destination.
  - Summary message may fail if aggregate fields are absent.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring cadence |  | Create a research task | ## Phase 1: Research Layer<br>* Schedule Trigger: kicks off the workflow on your defined cadence, daily, weekly, or whatever fits your reporting cycle.<br>* Create a Research Task: spins up a research job via Exa AI, querying the web for market data, listings context, and pricing signals for your target area.<br>* Get a Research Task — polls Exa AI to retrieve the completed research output once it's ready.<br>* If1 + Wait1 — checks whether the task is done. If not, it waits and loops back to poll again. Once confirmed, it passes the structured data forward to Phase 2.<br>## Real Estate Research Report Generation + Instantly<br>### This n8n template automates real estate research reports — pulling live market data, structuring it, and filling it into a branded presentation template.<br>Use cases include recurring client updates, market snapshots, and listing context reports where the structure stays fixed but the data changes every time.<br>## How it works<br>* Exa AI handles the research layer — pulling market information, listings context, and related web data automatically. The raw output is routed into n8n for light processing. It gets organized into sections: overview, pricing, listings, and key points.<br>* The structured data is then sent to PandaDoc, which has a custom real estate research template with fixed layouts and brand rules already set up. Instead of generating random slides, the data fills into the template. The result is a consistent, clean report every time — no manual cleanup needed.<br>* The report document generated. Will pend for approval in Slack, or email.<br>* Onced approved, it will automatically fetch your client/investor list in google sheets, or your preferred CRMs, and send them a personalised report via instantly ai, instead of sending your client/investor individually.<br>### How to use<br>* The schedule trigger node is used as a starting point — swap it with a webhook or form trigger to fit your workflow.<br>* Set up your own PandaDoc template with the layout and branding you want before connecting it. Works best for recurring reports where the structure stays the same but the underlying data changes each time.<br>* Set up your Google client Oauth in Google Console. Create project in Google console and apply the Client ID and secret.<br>* Set up your template your emails within Instantly AI<br>* Set Up Slack chat/channel<br>### Requirements<br>* Exa AI account for web research<br>* PandaDoc API with a custom report template configured<br>* Google Oauth<br>* Slack API<br>* Instantly API<br>* n8n instance (self-hosted or cloud)<br>This for anyone taking too much time with research and presentation, great when you're sending updates and listings to your investors and etc. |
| Create a research task | Exa | Creates asynchronous web research task in Exa | Schedule Trigger | Get a research task | ## Phase 1: Research Layer<br>* Schedule Trigger: kicks off the workflow on your defined cadence, daily, weekly, or whatever fits your reporting cycle.<br>* Create a Research Task: spins up a research job via Exa AI, querying the web for market data, listings context, and pricing signals for your target area.<br>* Get a Research Task — polls Exa AI to retrieve the completed research output once it's ready.<br>* If1 + Wait1 — checks whether the task is done. If not, it waits and loops back to poll again. Once confirmed, it passes the structured data forward to Phase 2. |
| Get a research task | Exa | Polls Exa for task status and result | Create a research task, Wait1 | If1 | ## Phase 1: Research Layer<br>* Schedule Trigger: kicks off the workflow on your defined cadence, daily, weekly, or whatever fits your reporting cycle.<br>* Create a Research Task: spins up a research job via Exa AI, querying the web for market data, listings context, and pricing signals for your target area.<br>* Get a Research Task — polls Exa AI to retrieve the completed research output once it's ready.<br>* If1 + Wait1 — checks whether the task is done. If not, it waits and loops back to poll again. Once confirmed, it passes the structured data forward to Phase 2. |
| If1 | If | Checks whether Exa research is completed | Get a research task | PandaDoc (Generate Presentation), Wait1 | ## Phase 1: Research Layer<br>* Schedule Trigger: kicks off the workflow on your defined cadence, daily, weekly, or whatever fits your reporting cycle.<br>* Create a Research Task: spins up a research job via Exa AI, querying the web for market data, listings context, and pricing signals for your target area.<br>* Get a Research Task — polls Exa AI to retrieve the completed research output once it's ready.<br>* If1 + Wait1 — checks whether the task is done. If not, it waits and loops back to poll again. Once confirmed, it passes the structured data forward to Phase 2. |
| Wait1 | Wait | Delays before repolling Exa | If1 | Get a research task | ## Phase 1: Research Layer<br>* Schedule Trigger: kicks off the workflow on your defined cadence, daily, weekly, or whatever fits your reporting cycle.<br>* Create a Research Task: spins up a research job via Exa AI, querying the web for market data, listings context, and pricing signals for your target area.<br>* Get a Research Task — polls Exa AI to retrieve the completed research output once it's ready.<br>* If1 + Wait1 — checks whether the task is done. If not, it waits and loops back to poll again. Once confirmed, it passes the structured data forward to Phase 2. |
| PandaDoc (Generate Presentation) | HTTP Request | Creates PandaDoc document from research data | If1 | PandaDoc (checkPresentationStatus) | ## Report Generation<br>* PandaDoc (Generate Presentation): — POST — sends the structured research data into your PandaDoc template via API. The data fills into your fixed layout and brand rules automatically.<br>* PandaDoc (checkPresentationStatus): — SET — polls PandaDoc to confirm the document has finished generating.<br>* If + Wait: same pattern as Phase 1. If the document isn't ready yet, it waits and re-checks. Once the report is confirmed, it moves to Phase 3. |
| PandaDoc (checkPresentationStatus) | HTTP Request | Polls PandaDoc for document generation status | PandaDoc (Generate Presentation), Wait | If | ## Report Generation<br>* PandaDoc (Generate Presentation): — POST — sends the structured research data into your PandaDoc template via API. The data fills into your fixed layout and brand rules automatically.<br>* PandaDoc (checkPresentationStatus): — SET — polls PandaDoc to confirm the document has finished generating.<br>* If + Wait: same pattern as Phase 1. If the document isn't ready yet, it waits and re-checks. Once the report is confirmed, it moves to Phase 3. |
| If | If | Checks whether PandaDoc status is no longer pending | PandaDoc (checkPresentationStatus) | HTTP Request, Wait | ## Report Generation<br>* PandaDoc (Generate Presentation): — POST — sends the structured research data into your PandaDoc template via API. The data fills into your fixed layout and brand rules automatically.<br>* PandaDoc (checkPresentationStatus): — SET — polls PandaDoc to confirm the document has finished generating.<br>* If + Wait: same pattern as Phase 1. If the document isn't ready yet, it waits and re-checks. Once the report is confirmed, it moves to Phase 3. |
| Wait | Wait | Delays before repolling PandaDoc | If | PandaDoc (checkPresentationStatus) | ## Report Generation<br>* PandaDoc (Generate Presentation): — POST — sends the structured research data into your PandaDoc template via API. The data fills into your fixed layout and brand rules automatically.<br>* PandaDoc (checkPresentationStatus): — SET — polls PandaDoc to confirm the document has finished generating.<br>* If + Wait: same pattern as Phase 1. If the document isn't ready yet, it waits and re-checks. Once the report is confirmed, it moves to Phase 3. |
| HTTP Request | HTTP Request | Fetches final report URL or payload from PandaDoc result | If | Pending Approval | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Pending Approval | Slack | Sends Slack notification for review | HTTP Request | Fetch contact | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Fetch contact | Google Sheets | Reads client/investor contacts from spreadsheet | Pending Approval | Batch Instantly Upload | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Batch Instantly Upload | Split In Batches | Iterates through recipients in controlled batches | Fetch contact, Wait for Instantly Rate Limit | Prep Batch Upload Data, Instantly Add Lead | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Instantly Add Lead | HTTP Request | Uploads each contact or batch payload to Instantly AI | Batch Instantly Upload | Wait for Instantly Rate Limit | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed.<br>Phase 5: Instantly Upload + Reporting<br>HTTP Request to Instantly API |
| Wait for Instantly Rate Limit | Wait | Delays between Instantly uploads to reduce rate-limit risk | Instantly Add Lead | Batch Instantly Upload | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Prep Batch Upload Data | Code | Prepares sheet-update data or batch metadata | Batch Instantly Upload | Update Sheet Status | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Update Sheet Status | Google Sheets | Writes send status back to source sheet | Prep Batch Upload Data | Aggregate Final Results | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Aggregate Final Results | Code | Builds final delivery summary | Update Sheet Status | Send a message1 | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Send a message1 | Slack | Sends final completion notice to Slack | Aggregate Final Results |  | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Sticky Note | Sticky Note | Documentation / visual annotation |  |  | ## Phase 1: Research Layer<br>* Schedule Trigger: kicks off the workflow on your defined cadence, daily, weekly, or whatever fits your reporting cycle.<br>* Create a Research Task: spins up a research job via Exa AI, querying the web for market data, listings context, and pricing signals for your target area.<br>* Get a Research Task — polls Exa AI to retrieve the completed research output once it's ready.<br>* If1 + Wait1 — checks whether the task is done. If not, it waits and loops back to poll again. Once confirmed, it passes the structured data forward to Phase 2. |
| Sticky Note1 | Sticky Note | Documentation / visual annotation |  |  | ## Report Generation<br>* PandaDoc (Generate Presentation): — POST — sends the structured research data into your PandaDoc template via API. The data fills into your fixed layout and brand rules automatically.<br>* PandaDoc (checkPresentationStatus): — SET — polls PandaDoc to confirm the document has finished generating.<br>* If + Wait: same pattern as Phase 1. If the document isn't ready yet, it waits and re-checks. Once the report is confirmed, it moves to Phase 3. |
| Sticky Note2 | Sticky Note | Documentation / visual annotation |  |  | ### Send via Instantly AI<br>* HTTP Request — fetches the finalised PandaDoc report URL or payload to pass into the distribution flow.<br>* Pending Approval (Slack — post message) — sends the report to your Slack channel or DM for human review before anything goes out to clients.<br>* Fetch Contact (Google Sheets — read sheet) — pulls your client or investor list from Google Sheets once the report is approved.<br>* Batch Instantly Upload — groups contacts into a batch ready for upload to Instantly AI.<br>* Instantly Add Lead — SET — pushes each contact into your Instantly AI campaign with the personalised report attached.<br>* Wait for Instantly Rate Limit — adds a delay between batches to stay within Instantly AI's API rate limits and avoid dropped leads.<br>* Prep Batch Upload Data + Update Sheet Status — formats the next batch and writes back the send status to your Google Sheet so you always know who's been contacted.<br>* Aggregate Final Results — consolidates the full send summary across all batches.<br>* Send a Message (Slack — post message) — notifies your Slack once the full distribution is complete. Closed loop, no manual follow-up needed. |
| Sticky Note3 | Sticky Note | Documentation / visual annotation |  |  | ## Real Estate Research Report Generation + Instantly<br>### This n8n template automates real estate research reports — pulling live market data, structuring it, and filling it into a branded presentation template.<br>Use cases include recurring client updates, market snapshots, and listing context reports where the structure stays fixed but the data changes every time.<br>## How it works<br>* Exa AI handles the research layer — pulling market information, listings context, and related web data automatically. The raw output is routed into n8n for light processing. It gets organized into sections: overview, pricing, listings, and key points.<br>* The structured data is then sent to PandaDoc, which has a custom real estate research template with fixed layouts and brand rules already set up. Instead of generating random slides, the data fills into the template. The result is a consistent, clean report every time — no manual cleanup needed.<br>* The report document generated. Will pend for approval in Slack, or email.<br>* Onced approved, it will automatically fetch your client/investor list in google sheets, or your preferred CRMs, and send them a personalised report via instantly ai, instead of sending your client/investor individually.<br>### How to use<br>* The schedule trigger node is used as a starting point — swap it with a webhook or form trigger to fit your workflow.<br>* Set up your own PandaDoc template with the layout and branding you want before connecting it. Works best for recurring reports where the structure stays the same but the underlying data changes each time.<br>* Set up your Google client Oauth in Google Console. Create project in Google console and apply the Client ID and secret.<br>* Set up your template your emails within Instantly AI<br>* Set Up Slack chat/channel<br>### Requirements<br>* Exa AI account for web research<br>* PandaDoc API with a custom report template configured<br>* Google Oauth<br>* Slack API<br>* Instantly API<br>* n8n instance (self-hosted or cloud)<br>This for anyone taking too much time with research and presentation, great when you're sending updates and listings to your investors and etc. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence for n8n.

## Preparation

1. **Create credentials in n8n**
   - Exa AI credential for the Exa nodes.
   - Google Sheets OAuth2 credential.
   - Slack credential.
   - PandaDoc authentication method:
     - either generic HTTP header auth in HTTP Request nodes,
     - or preconfigured credentials if your environment supports it.
   - Instantly API authentication:
     - usually API key in headers on HTTP Request nodes.

2. **Prepare external systems**
   - Build a PandaDoc template with variables matching your research fields.
   - Set up your Google Sheet with recipient columns such as:
     - `name`
     - `email`
     - `status`
     - optional `campaign_id`, `investor_type`, `region`
   - Configure your Instantly campaign/sequence and sender accounts.
   - Create a Slack channel or DM destination for review and final notices.

3. **Decide your schedule**
   - Daily, weekly, or monthly depending on reporting frequency.

---

## Build Steps

### Phase 1 — Exa research

1. **Add a Schedule Trigger node**
   - Set the rule to your desired cadence.
   - Example: every Monday morning.

2. **Add an Exa node named `Create a research task`**
   - Resource: `Research`
   - Operation: `Create Task`
   - Attach Exa credentials.
   - Configure the research prompt/task definition to gather:
     - market overview
     - pricing trends
     - listing context
     - local signals relevant to your area
   - Connect `Schedule Trigger -> Create a research task`.

3. **Add another Exa node named `Get a research task`**
   - Resource: `Research`
   - Operation: `Get Task`
   - Use the task ID from the previous node.
   - Connect `Create a research task -> Get a research task`.

4. **Add an If node named `If1`**
   - Condition:
     - Left value: `{{$json.status}}`
     - Operation: `equals`
     - Right value: `completed`
   - Connect `Get a research task -> If1`.

5. **Add a Wait node named `Wait1`**
   - Configure a polling delay such as 30 seconds, 1 minute, or longer depending on Exa speed and cost tolerance.
   - Connect the **false** output of `If1 -> Wait1`.
   - Connect `Wait1 -> Get a research task`.

### Phase 2 — PandaDoc generation

6. **Add an HTTP Request node named `PandaDoc (Generate Presentation)`**
   - Method: `POST`
   - URL: PandaDoc document generation endpoint.
   - Add authentication headers, for example bearer token.
   - Add request body with:
     - template ID
     - document name
     - recipient metadata if needed
     - PandaDoc variables populated from the Exa response
   - Connect the **true** output of `If1 -> PandaDoc (Generate Presentation)`.

7. **Add an HTTP Request node named `PandaDoc (checkPresentationStatus)`**
   - Method: typically `GET`
   - URL: PandaDoc document status endpoint using the document ID returned by generation.
   - Example logic: `/documents/{{document_id}}`
   - Connect `PandaDoc (Generate Presentation) -> PandaDoc (checkPresentationStatus)`.

8. **Add an If node named `If`**
   - Condition:
     - Left value: `{{$json.status}}`
     - Operation: `not equals`
     - Right value: `pending`
   - Connect `PandaDoc (checkPresentationStatus) -> If`.

9. **Add a Wait node named `Wait`**
   - Configure a delay such as 20–60 seconds.
   - Connect false branch `If -> Wait`.
   - Connect `Wait -> PandaDoc (checkPresentationStatus)`.

10. **Recommended improvement**
   - Instead of `status != pending`, use a stricter condition such as:
     - `status == completed`
   - Or add a second branch for explicit error statuses.

### Phase 3 — Retrieve report and notify Slack

11. **Add an HTTP Request node named `HTTP Request`**
   - Configure URL as:
     - `{{$json.data.path}}`
   - Use this only if PandaDoc actually returns a downloadable path at `data.path`.
   - If PandaDoc returns a different schema, adapt accordingly.
   - Connect true branch `If -> HTTP Request`.

12. **Add a Slack node named `Pending Approval`**
   - Configure it to send a message to a Slack channel or user.
   - Include:
     - report title
     - report link or attachment
     - summary text
   - Connect `HTTP Request -> Pending Approval`.

13. **If you want real approval, add missing logic**
   - The provided workflow does **not** pause for a Slack approval action.
   - To make approval real, add:
     - a Slack interactive workflow,
     - a Webhook,
     - or a Wait for Webhook / approval callback pattern.
   - Only after approval should contacts be fetched.

### Phase 4 — Fetch contacts and send through Instantly

14. **Add a Google Sheets node named `Fetch contact`**
   - Operation: read rows.
   - Select your spreadsheet document.
   - Select your sheet tab.
   - Return all rows or filtered rows as needed.
   - Connect `Pending Approval -> Fetch contact`.

15. **Add a Split In Batches node named `Batch Instantly Upload`**
   - Set a batch size appropriate for your Instantly plan.
   - Example: 10, 25, or 50 rows per batch.
   - Connect `Fetch contact -> Batch Instantly Upload`.

16. **Add an HTTP Request node named `Instantly Add Lead`**
   - Method: usually `POST`
   - URL: Instantly lead/campaign endpoint.
   - Add API auth header.
   - Build request body from each contact item:
     - email
     - first name / full name
     - campaign ID
     - custom fields such as report URL
   - Connect main batch output `Batch Instantly Upload -> Instantly Add Lead`.

17. **Add a Wait node named `Wait for Instantly Rate Limit`**
   - Set a delay between batches according to Instantly limits.
   - Example: 5–30 seconds depending on volume and plan.
   - Connect `Instantly Add Lead -> Wait for Instantly Rate Limit`.
   - Connect `Wait for Instantly Rate Limit -> Batch Instantly Upload` to continue the loop.

### Phase 5 — Update spreadsheet and final summary

18. **Add a Code node named `Prep Batch Upload Data`**
   - Connect another output from `Batch Instantly Upload -> Prep Batch Upload Data`.
   - Use it to prepare records for update, for example:
     - row identifier
     - send status
     - timestamp
     - report URL
   - Because code was not included in the export, define your own schema clearly.

19. **Add a Google Sheets node named `Update Sheet Status`**
   - Operation: update row(s).
   - Use spreadsheet and sheet from your source file.
   - Match each row by an ID, email, or row number.
   - Update fields such as:
     - `status = sent`
     - `sent_at = now`
     - optional `report_url`
   - Connect `Prep Batch Upload Data -> Update Sheet Status`.

20. **Add a Code node named `Aggregate Final Results`**
   - Connect `Update Sheet Status -> Aggregate Final Results`.
   - Build a summary such as:
     - total contacts processed
     - sent count
     - failed count
     - timestamp
     - campaign name

21. **Add a Slack node named `Send a message1`**
   - Configure final Slack notification.
   - Include the aggregate summary from the previous code node.
   - Connect `Aggregate Final Results -> Send a message1`.

---

## Suggested field design

To make the workflow reproducible, define a stable data contract:

### Exa output fields
- `overview`
- `pricing_signals`
- `listing_context`
- `key_points`
- `region_name`
- `report_period`

### PandaDoc variables
- `{{overview}}`
- `{{pricing_signals}}`
- `{{listing_context}}`
- `{{key_points}}`
- `{{region_name}}`
- `{{report_date}}`

### Google Sheets columns
- `row_id`
- `name`
- `email`
- `campaign_id`
- `status`
- `sent_at`
- `report_url`

### Instantly payload fields
- email
- name
- campaign identifier
- custom variable with report URL

---

## Credential configuration notes

1. **Exa**
   - Add Exa API key in the Exa credential.
   - Test both create-task and get-task operations.

2. **Google Sheets**
   - Create Google OAuth app in Google Cloud Console.
   - Enable Google Sheets API.
   - Add redirect URI from n8n.
   - Authorize spreadsheet access.

3. **Slack**
   - Use a bot token or app connection supported by your n8n Slack node version.
   - Ensure the app can post to the target channel or DM.

4. **PandaDoc**
   - Use PandaDoc API key or bearer token in HTTP headers.
   - Confirm endpoint URLs and template permissions.

5. **Instantly**
   - Add the API key in the HTTP Request header.
   - Verify the exact endpoint for adding leads to campaigns.

---

## Missing implementation details you must supply

The workflow export omits several practical settings. You will need to define:

- Exa research prompt/task payload
- PandaDoc API URL, template ID, body, and headers
- PandaDoc status endpoint and field mapping
- Slack message content and destination
- Google Sheets read/update operations and match keys
- Instantly endpoint, auth, and payload
- Code node scripts for:
  - batch preparation
  - final result aggregation
- Wait durations
- Batch size

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Exa AI charges per search query. Validate pricing before running large scheduled jobs. | Exa pricing page |
| Instantly AI has lead upload rate limits. This workflow uses a wait loop between batches, but plan-specific limits should still be checked. | Instantly API / plan limits |
| PandaDoc generation is asynchronous. Document creation time varies based on template complexity and must be polled. | PandaDoc API documentation |
| The workflow is positioned for recurring market updates, listing context, investor updates, and fixed-structure research reports. | Template use case |
| The default entry point is a Schedule Trigger, but it can be replaced by a webhook or form trigger for on-demand execution. | Customization guidance |
| Google Sheets can be replaced by another CRM or data source such as HubSpot, Airtable, or GoHighLevel. | Customization guidance |
| Slack approval in the current JSON is only a notification step; it does not enforce an approval pause. | Important implementation note |
| Suggested enhancement: add an AI summarization node before PandaDoc to condense Exa output further before template injection. | Customization guidance |