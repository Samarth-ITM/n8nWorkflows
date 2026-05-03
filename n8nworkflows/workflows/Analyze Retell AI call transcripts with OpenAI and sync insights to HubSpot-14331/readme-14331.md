Analyze Retell AI call transcripts with OpenAI and sync insights to HubSpot

https://n8nworkflows.xyz/workflows/analyze-retell-ai-call-transcripts-with-openai-and-sync-insights-to-hubspot-14331


# Analyze Retell AI call transcripts with OpenAI and sync insights to HubSpot

# 1. Workflow Overview

This workflow receives a **Retell AI `call_analyzed` webhook** when a voice call ends, extracts transcript and call metadata, sends the transcript to **OpenAI** for structured sales analysis, then syncs the results into **HubSpot**, optionally alerts the sales team in **Slack**, creates a **HubSpot follow-up task** for qualified leads, and appends every analyzed call to **Google Sheets**.

Its primary use cases are:

- automated post-call sales qualification
- CRM enrichment after AI voice calls
- lead routing based on AI-derived intent and score
- centralized logging for reporting and follow-up

## 1.1 Trigger & Configuration

The workflow starts from an HTTP webhook that Retell AI calls after call analysis. A Set node injects editable runtime configuration such as Slack channel, qualification threshold, and Google Sheets destination.

## 1.2 Transcript Parsing & Normalization

The incoming webhook payload is normalized into a consistent internal structure. This block extracts identifiers, phone numbers, duration, status, transcript text, and carries forward the user config values.

## 1.3 AI Analysis

The normalized transcript is sent to OpenAI with a prompt requesting **strict JSON output** containing sentiment, lead score, summary, action items, buying signals, objections, and next-step guidance. A downstream Code node parses that result and computes whether the lead qualifies based on the configured threshold.

## 1.4 CRM Update & Webhook Response

The analyzed call is written back to HubSpot as contact properties. After that update, the workflow sends a JSON response back to the originating webhook request.

## 1.5 Lead Routing

If the AI-derived score passes the qualification check, the workflow branches to send a Slack alert and create a HubSpot follow-up task.

## 1.6 Logging

Regardless of qualification, the workflow appends a record of the analyzed call to Google Sheets for tracking and reporting.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Configuration

### Overview

This block receives the Retell AI event and applies editable workflow-level configuration values. It is the entry point for the entire process and determines key routing behavior later in the flow.

### Nodes Involved

- Receive Retell call webhook
- Set user config variables

### Node Details

#### Receive Retell call webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Receives inbound HTTP POST requests from Retell AI.
- **Configuration choices:**
  - Path: `retell-call-analyzed`
  - HTTP method: `POST`
  - Response mode: `responseNode`
- **Key expressions or variables used:**
  - No expressions in configuration
  - The payload is expected in the webhook request body
- **Input and output connections:**
  - Entry point node
  - Outputs to **Set user config variables**
- **Version-specific requirements:**
  - Uses webhook node type version 2
  - Because response mode is `responseNode`, the workflow must eventually reach a **Respond to Webhook** node to complete the HTTP request properly
- **Edge cases or potential failure types:**
  - Retell sending unexpected payload structure
  - Wrong webhook path configured in Retell
  - Workflow not active when using production webhook URL
  - Request timeout if downstream processing stalls before response
- **Sub-workflow reference:** None

#### Set user config variables

- **Type and technical role:** `n8n-nodes-base.set`  
  Adds editable configuration fields into the current item while preserving the incoming webhook payload.
- **Configuration choices:**
  - Manual mode
  - `includeOtherFields: true`, so the original webhook content remains available
  - Adds:
    - `slackChannel = #sales-alerts`
    - `leadScoreThreshold = 7`
    - `googleSheetId = YOUR_GOOGLE_SHEET_ID_HERE`
    - `sheetName = Call Log`
- **Key expressions or variables used:**
  - Static values only
- **Input and output connections:**
  - Input from **Receive Retell call webhook**
  - Output to **Parse call transcript and metadata**
- **Version-specific requirements:**
  - Uses Set node version 3.4
- **Edge cases or potential failure types:**
  - Placeholder `googleSheetId` left unchanged
  - Invalid Slack channel name later causing Slack send failure
  - Threshold value not aligned with expected 1–10 lead score range
- **Sub-workflow reference:** None

---

## 2.2 Transcript Parsing & Normalization

### Overview

This block converts the raw Retell webhook payload into a stable internal schema used by all downstream integrations. It also handles multiple possible transcript locations in the request body.

### Nodes Involved

- Parse call transcript and metadata

### Node Details

#### Parse call transcript and metadata

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that extracts fields from the webhook payload and prepares normalized output.
- **Configuration choices:**
  - Reads first input item
  - Uses `input.body || input` so it works whether the payload is nested under `body` or already flattened
  - Extracts:
    - `callId`
    - `agentId`
    - `fromNumber`
    - `toNumber`
    - `callDurationSeconds`
    - `callStatus`
    - `disconnectReason`
    - `transcript`
    - config values
    - `analyzedAt`
  - Handles transcript from:
    - `body.transcript`
    - `body.call.transcript`
    - `body.transcript_object[]` mapped into `role: content` lines
- **Key expressions or variables used:**
  - `$input.first().json`
  - Falls back to defaults like `'unknown'`, `0`, `'completed'`
  - `new Date().toISOString()` for analysis timestamp
- **Input and output connections:**
  - Input from **Set user config variables**
  - Output to **Analyze call with OpenAI**
- **Version-specific requirements:**
  - Uses Code node version 2
- **Edge cases or potential failure types:**
  - Missing or empty transcript leading to poor AI output
  - Unexpected Retell field names not covered by the fallback logic
  - `transcript_object` items missing `role` or `content`
  - Non-numeric duration resulting in `NaN` risk if payload shape changes
- **Sub-workflow reference:** None

---

## 2.3 AI Analysis

### Overview

This block sends the transcript to OpenAI and enforces a structured interpretation of the call. It then parses the model output into normalized fields and computes a qualification boolean.

### Nodes Involved

- Analyze call with OpenAI
- Extract and structure AI analysis

### Node Details

#### Analyze call with OpenAI

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Chat model invocation that asks OpenAI to return JSON-only call analysis.
- **Configuration choices:**
  - Resource: `chat`
  - Model: `gpt-4o-mini`
  - Temperature: `0.2`
  - Max tokens: `1500`
  - Prompt requests JSON with:
    - `sentiment`
    - `leadScore`
    - `summary`
    - `actionItems`
    - `keyTopics`
    - `buyingSignals`
    - `objections`
    - `recommendedNextStep`
  - Includes transcript, call duration, and status in the prompt
  - Explicitly instructs: “Return ONLY valid JSON, no markdown formatting.”
- **Key expressions or variables used:**
  - `{{ $json.transcript }}`
  - `{{ $json.callDurationSeconds }}`
  - `{{ $json.callStatus }}`
- **Input and output connections:**
  - Input from **Parse call transcript and metadata**
  - Output to **Extract and structure AI analysis**
- **Version-specific requirements:**
  - Uses LangChain OpenAI node version 1.8
  - Requires valid OpenAI credentials in the node
- **Edge cases or potential failure types:**
  - Authentication failure with OpenAI
  - Model name unavailable in account or n8n version
  - Model returning non-JSON despite prompt instruction
  - Token overflow for very long transcripts
  - Rate limiting or API timeout
- **Sub-workflow reference:** None

#### Extract and structure AI analysis

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the OpenAI response, strips possible Markdown code fences, applies fallback defaults on parse failure, and merges the AI analysis with the earlier parsed call metadata.
- **Configuration choices:**
  - Reads AI output from `input.message?.content || input.text || input.content || JSON.stringify(input)`
  - Removes ```json and ``` wrappers before `JSON.parse`
  - On parse failure, returns a safe fallback analysis:
    - neutral sentiment
    - lead score 5
    - manual review summary
    - one fallback action item
  - Computes `isQualified` as:
    - `analysis.leadScore >= (prev.leadScoreThreshold || 7)`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Parse call transcript and metadata').first().json`
- **Input and output connections:**
  - Input from **Analyze call with OpenAI**
  - Outputs to:
    - **Update HubSpot contact with insights**
    - **Check if lead is qualified**
    - **Log call analysis to tracking sheet**
- **Version-specific requirements:**
  - Uses Code node version 2
- **Edge cases or potential failure types:**
  - If AI output shape changes, extraction may fail
  - If `leadScore` comes back as string instead of integer, qualification comparison may behave unexpectedly
  - If prompt result omits arrays, code handles with `|| []`
  - Named-node reference breaks if the upstream parse node is renamed without updating expressions
- **Sub-workflow reference:** None

---

## 2.4 CRM Update & Webhook Response

### Overview

This block updates the HubSpot contact record with AI-derived insights and then completes the original webhook request. It is the only branch directly tied to the webhook HTTP response.

### Nodes Involved

- Update HubSpot contact with insights
- Respond to Retell webhook

### Node Details

#### Update HubSpot contact with insights

- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Updates an existing HubSpot contact record with custom properties derived from the AI analysis.
- **Configuration choices:**
  - Resource: `contact`
  - Operation: `update`
  - Contact identifier: `={{ $json.fromNumber }}`
  - Properties updated:
    - `last_call_summary`
    - `lead_score`
    - `last_call_sentiment`
    - `last_call_date`
- **Key expressions or variables used:**
  - `{{ $json.summary }}`
  - `{{ $json.leadScore }}`
  - `{{ $json.sentiment }}`
  - `{{ $json.analyzedAt }}`
- **Input and output connections:**
  - Input from **Extract and structure AI analysis**
  - Output to **Respond to Retell webhook**
- **Version-specific requirements:**
  - Uses HubSpot node version 2
  - Requires HubSpot credentials
  - Assumes custom contact properties already exist in HubSpot
- **Edge cases or potential failure types:**
  - Contact update may fail if `fromNumber` is not a valid contact ID in HubSpot
  - In many HubSpot setups, phone number is not the internal contact ID; this is a major implementation risk
  - Property names may not exist in HubSpot and cause API errors
  - Auth, permission, or rate limit issues
- **Sub-workflow reference:** None

#### Respond to Retell webhook

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the HTTP response back to the original webhook caller.
- **Configuration choices:**
  - Respond with JSON
  - Body:
    - `status: "processed"`
    - `callId` from the parse node
- **Key expressions or variables used:**
  - `{{ $('Parse call transcript and metadata').first().json.callId }}`
- **Input and output connections:**
  - Input from **Update HubSpot contact with insights**
  - No downstream nodes
- **Version-specific requirements:**
  - Uses Respond to Webhook node version 1.1
- **Edge cases or potential failure types:**
  - If the HubSpot update fails, this response node is never reached
  - That means Retell may receive no response or a timeout even though some side branches may still execute
  - Named-node reference depends on upstream node name stability
- **Sub-workflow reference:** None

---

## 2.5 Lead Routing

### Overview

This block determines whether a lead is qualified and, if so, triggers high-priority downstream actions. Qualified leads generate both a Slack alert and a HubSpot follow-up task.

### Nodes Involved

- Check if lead is qualified
- Alert sales team about hot lead
- Create follow-up task in HubSpot

### Node Details

#### Check if lead is qualified

- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates the boolean field `isQualified`.
- **Configuration choices:**
  - Condition requires `={{ $json.isQualified }}` to be `true`
  - True branch connects to Slack and HubSpot task creation
  - False branch is unused
- **Key expressions or variables used:**
  - `{{ $json.isQualified }}`
- **Input and output connections:**
  - Input from **Extract and structure AI analysis**
  - True outputs to:
    - **Alert sales team about hot lead**
    - **Create follow-up task in HubSpot**
- **Version-specific requirements:**
  - Uses If node version 2
- **Edge cases or potential failure types:**
  - If `isQualified` is missing or not boolean, strict validation may cause unexpected routing
  - Since false branch is unused, no explicit non-qualified handling occurs here
- **Sub-workflow reference:** None

#### Alert sales team about hot lead

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack message for qualified leads.
- **Configuration choices:**
  - Resource: `message`
  - Channel is dynamic from config
  - Message includes:
    - phone number
    - lead score
    - sentiment
    - summary
    - buying signals
    - recommended next step
- **Key expressions or variables used:**
  - `{{ $('Extract and structure AI analysis').first().json.slackChannel }}`
  - Multiple fields pulled from **Extract and structure AI analysis**
  - Buying signals rendered using `.join('\n• ')`
- **Input and output connections:**
  - Input from true branch of **Check if lead is qualified**
  - No downstream nodes
- **Version-specific requirements:**
  - Uses Slack node version 2.2
  - Requires Slack credentials
- **Edge cases or potential failure types:**
  - Invalid or inaccessible channel
  - Buying signals array empty, resulting in visually sparse output
  - Slack formatting quirks if summary contains special characters
  - Named-node references depend on stable node naming
- **Sub-workflow reference:** None

#### Create follow-up task in HubSpot

- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Creates a task in HubSpot for qualified leads.
- **Configuration choices:**
  - Resource: `task`
  - Operation: `create`
  - Subject contains caller phone number
  - Body includes recommended next step and action item list
  - Status: `NOT_STARTED`
  - Priority: `HIGH`
- **Key expressions or variables used:**
  - `{{ $('Extract and structure AI analysis').first().json.recommendedNextStep }}`
  - `{{ $('Extract and structure AI analysis').first().json.actionItems.join('\n- ') }}`
  - `{{ $('Extract and structure AI analysis').first().json.fromNumber }}`
- **Input and output connections:**
  - Input from true branch of **Check if lead is qualified**
  - No downstream nodes
- **Version-specific requirements:**
  - Uses HubSpot node version 2
  - Requires HubSpot credentials
- **Edge cases or potential failure types:**
  - Task may be created without association to a specific contact/company/deal
  - If action items are empty, body will contain minimal content
  - Auth/permission issues in HubSpot
- **Sub-workflow reference:** None

---

## 2.6 Logging

### Overview

This block writes every analyzed call to Google Sheets for operational reporting. It is independent from qualification routing and runs for all calls that reach the structured analysis stage.

### Nodes Involved

- Log call analysis to tracking sheet

### Node Details

#### Log call analysis to tracking sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row to a target Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Document ID comes from config
  - Sheet name comes from config
  - Mapping mode: `defineBelow`
  - Columns written:
    - Date
    - Call ID
    - Summary
    - Qualified
    - Sentiment
    - Lead Score
    - From Number
    - Action Items
    - Duration (sec)
- **Key expressions or variables used:**
  - All fields are referenced from **Extract and structure AI analysis**
  - `actionItems.join('; ')`
- **Input and output connections:**
  - Input from **Extract and structure AI analysis**
  - No downstream nodes
- **Version-specific requirements:**
  - Uses Google Sheets node version 4.5
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Placeholder sheet ID not replaced
  - Sheet tab name not found
  - Column names in target sheet not matching expected mapping behavior
  - Permissions denied on spreadsheet
  - Concurrency issues if many rows are appended simultaneously
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Retell call webhook | Webhook | Receives Retell `call_analyzed` POST events |  | Set user config variables | ## 1. Trigger & Config<br>Retell sends a webhook when a call finishes. The config node holds your editable settings. |
| Set user config variables | Set | Injects editable workflow settings while preserving incoming payload | Receive Retell call webhook | Parse call transcript and metadata | ## 1. Trigger & Config<br>Retell sends a webhook when a call finishes. The config node holds your editable settings. |
| Parse call transcript and metadata | Code | Normalizes payload fields and transcript text for downstream use | Set user config variables | Analyze call with OpenAI |  |
| Analyze call with OpenAI | OpenAI (LangChain Chat) | Generates structured sales analysis from transcript | Parse call transcript and metadata | Extract and structure AI analysis | ## 2. AI Analysis<br>OpenAI reads the full transcript and returns structured JSON: sentiment, lead score, action items, and a summary. |
| Extract and structure AI analysis | Code | Parses model output, merges metadata, and computes qualification | Analyze call with OpenAI | Update HubSpot contact with insights; Check if lead is qualified; Log call analysis to tracking sheet | ## 2. AI Analysis<br>OpenAI reads the full transcript and returns structured JSON: sentiment, lead score, action items, and a summary. |
| Update HubSpot contact with insights | HubSpot | Updates a HubSpot contact with AI-generated call properties | Extract and structure AI analysis | Respond to Retell webhook | ## 3. CRM & Routing<br>Updates the HubSpot contact. Qualified leads trigger a Slack alert and a HubSpot follow-up task. |
| Check if lead is qualified | If | Routes only qualified leads to alerting and follow-up | Extract and structure AI analysis | Alert sales team about hot lead; Create follow-up task in HubSpot | ## 3. CRM & Routing<br>Updates the HubSpot contact. Qualified leads trigger a Slack alert and a HubSpot follow-up task. |
| Alert sales team about hot lead | Slack | Sends a Slack alert for hot leads | Check if lead is qualified |  | ## 3. CRM & Routing<br>Updates the HubSpot contact. Qualified leads trigger a Slack alert and a HubSpot follow-up task. |
| Create follow-up task in HubSpot | HubSpot | Creates a high-priority HubSpot follow-up task for qualified leads | Check if lead is qualified |  | ## 3. CRM & Routing<br>Updates the HubSpot contact. Qualified leads trigger a Slack alert and a HubSpot follow-up task. |
| Log call analysis to tracking sheet | Google Sheets | Appends each analyzed call to a reporting sheet | Extract and structure AI analysis |  | ## 4. Logging<br>Every call is appended to Google Sheets for reporting and tracking. |
| Respond to Retell webhook | Respond to Webhook | Returns a JSON acknowledgement to Retell | Update HubSpot contact with insights |  | ## 3. CRM & Routing<br>Updates the HubSpot contact. Qualified leads trigger a Slack alert and a HubSpot follow-up task. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Analyze Retell AI voice call transcripts and sync insights to HubSpot**

2. **Add a Webhook node** named **Receive Retell call webhook**.
   - Set **HTTP Method** to `POST`
   - Set **Path** to `retell-call-analyzed`
   - Set **Response Mode** to `Using Respond to Webhook Node`
   - Save the node
   - Later, copy its production URL into Retell AI webhook settings for the `call_analyzed` event

3. **Add a Set node** named **Set user config variables** and connect it after the webhook.
   - Enable keeping incoming fields (`Include Other Fields`)
   - Add these fields:
     - `slackChannel` → string → `#sales-alerts`
     - `leadScoreThreshold` → number → `7`
     - `googleSheetId` → string → `YOUR_GOOGLE_SHEET_ID_HERE`
     - `sheetName` → string → `Call Log`

4. **Add a Code node** named **Parse call transcript and metadata** and connect it after the Set node.
   - Paste logic equivalent to:
     - read input JSON
     - use `input.body || input`
     - extract:
       - `call_id`
       - `agent_id`
       - `from_number`
       - `to_number`
       - duration/status/disconnect reason
     - normalize transcript from:
       - `body.transcript`
       - `body.call.transcript`
       - `body.transcript_object[]`
     - carry forward:
       - `slackChannel`
       - `leadScoreThreshold`
       - `googleSheetId`
       - `sheetName`
     - generate `analyzedAt` as ISO timestamp
   - Ensure the output contains at least:
     - `callId`
     - `agentId`
     - `fromNumber`
     - `toNumber`
     - `callDurationSeconds`
     - `callStatus`
     - `disconnectReason`
     - `transcript`
     - `slackChannel`
     - `leadScoreThreshold`
     - `googleSheetId`
     - `sheetName`
     - `analyzedAt`

5. **Add an OpenAI node** named **Analyze call with OpenAI** and connect it after the parse node.
   - Use the **OpenAI Chat / LangChain OpenAI** node
   - Select model: `gpt-4o-mini`
   - Set:
     - **Temperature**: `0.2`
     - **Max Tokens**: `1500`
   - Add one user message instructing the model to return strict JSON with fields:
     - `sentiment`
     - `leadScore`
     - `summary`
     - `actionItems`
     - `keyTopics`
     - `buyingSignals`
     - `objections`
     - `recommendedNextStep`
   - Include expressions for:
     - transcript: `{{ $json.transcript }}`
     - duration: `{{ $json.callDurationSeconds }}`
     - status: `{{ $json.callStatus }}`
   - End the prompt with a strict instruction such as:
     - `Return ONLY valid JSON, no markdown formatting.`
   - Attach valid **OpenAI credentials**

6. **Add a Code node** named **Extract and structure AI analysis** and connect it after the OpenAI node.
   - Implement logic to:
     - retrieve AI response text from possible fields like `message.content`
     - strip code fences such as ```json
     - `JSON.parse()` the result
     - on failure, create fallback values:
       - neutral sentiment
       - lead score 5
       - manual review summary
       - fallback action item
     - merge parsed analysis with the output of **Parse call transcript and metadata**
     - compute:
       - `isQualified = leadScore >= leadScoreThreshold`
   - Output at least:
     - original parsed call data
     - `sentiment`
     - `leadScore`
     - `summary`
     - `actionItems`
     - `keyTopics`
     - `buyingSignals`
     - `objections`
     - `recommendedNextStep`
     - `isQualified`

7. **Add a HubSpot node** named **Update HubSpot contact with insights** and connect it from **Extract and structure AI analysis**.
   - Resource: `Contact`
   - Operation: `Update`
   - Set the contact identifier to the caller phone number expression:
     - `{{ $json.fromNumber }}`
   - Add properties:
     - `last_call_summary` → `{{ $json.summary }}`
     - `lead_score` → `{{ $json.leadScore }}`
     - `last_call_sentiment` → `{{ $json.sentiment }}`
     - `last_call_date` → `{{ $json.analyzedAt }}`
   - Attach valid **HubSpot credentials**
   - Important: create these custom properties in HubSpot first if they do not already exist
   - Important: verify whether your HubSpot node can truly update by phone number; many setups require an internal contact ID or a prior lookup step

8. **Add a Respond to Webhook node** named **Respond to Retell webhook** and connect it after the HubSpot contact update node.
   - Set response type to JSON
   - Return a body similar to:
     - `{"status":"processed","callId":"..."}`
   - Build `callId` using the parsed data from the earlier node

9. **Add an If node** named **Check if lead is qualified** and connect it from **Extract and structure AI analysis**.
   - Condition:
     - left value: `{{ $json.isQualified }}`
     - operation: `is true`
   - Leave false branch unused unless you want to add handling for non-qualified leads

10. **Add a Slack node** named **Alert sales team about hot lead** and connect it to the **true** branch of the If node.
    - Resource: `Message`
    - Channel: expression using `{{ $('Extract and structure AI analysis').first().json.slackChannel }}`
    - Compose a message containing:
      - phone number
      - lead score
      - sentiment
      - summary
      - buying signals
      - recommended next step
    - Attach valid **Slack credentials**
    - Ensure the connected Slack app has permission to post in the target channel

11. **Add a second HubSpot node** named **Create follow-up task in HubSpot** and connect it to the **true** branch of the If node.
    - Resource: `Task`
    - Operation: `Create`
    - Subject example:
      - `Follow up with hot lead: {{ fromNumber }}`
    - Body should include:
      - recommended next step
      - action items joined as a list
    - Set:
      - status = `NOT_STARTED`
      - priority = `HIGH`
    - Attach valid **HubSpot credentials**
    - Optionally enhance by associating the task with the relevant contact or deal

12. **Add a Google Sheets node** named **Log call analysis to tracking sheet** and connect it from **Extract and structure AI analysis**.
    - Operation: `Append`
    - Set **Document ID** from expression:
      - `{{ $('Extract and structure AI analysis').first().json.googleSheetId }}`
    - Set **Sheet Name** from expression:
      - `{{ $('Extract and structure AI analysis').first().json.sheetName }}`
    - Use explicit column mapping
    - Map these columns:
      - `Date` → analyzed timestamp
      - `Call ID` → call ID
      - `Summary` → summary
      - `Qualified` → boolean qualification result
      - `Sentiment` → sentiment
      - `Lead Score` → numeric lead score
      - `From Number` → caller number
      - `Action Items` → action items joined by semicolon
      - `Duration (sec)` → call duration
    - Attach valid **Google Sheets OAuth2 credentials**
    - Create the spreadsheet and sheet tab in advance

13. **Configure credentials** in all service nodes.
    - **OpenAI**
      - API key with access to the chosen model
    - **HubSpot**
      - OAuth or private app credentials with contact/task write access
    - **Slack**
      - OAuth credentials with permission to post messages
    - **Google Sheets**
      - OAuth2 credentials with edit access to the target spreadsheet

14. **Prepare HubSpot before activation**.
    - Ensure the contact properties exist:
      - `last_call_summary`
      - `lead_score`
      - `last_call_sentiment`
      - `last_call_date`
    - Confirm the update strategy:
      - if phone number cannot be used directly as the update identifier, insert a HubSpot search step before update

15. **Prepare Google Sheets before activation**.
    - Create a spreadsheet
    - Copy its document ID into `googleSheetId`
    - Create tab `Call Log` or adjust the config node to match your chosen sheet name
    - Ensure headers align with the mapped columns

16. **Prepare Retell AI webhook delivery**.
    - Activate the workflow
    - Copy the production webhook URL from **Receive Retell call webhook**
    - In Retell AI, configure webhook settings for event:
      - `call_analyzed`
    - Paste the n8n production URL

17. **Test the workflow**.
    - Send a sample Retell payload or complete a real test call
    - Validate:
      - transcript extraction works
      - OpenAI returns parsable JSON
      - HubSpot contact update succeeds
      - qualified calls create Slack alerts and HubSpot tasks
      - every call appends to Google Sheets
      - webhook caller receives the JSON response

18. **Recommended hardening improvements** if rebuilding for production:
    - add HubSpot contact lookup by phone before updating
    - add explicit validation for empty transcripts
    - add retry/error handling for API failures
    - move webhook response earlier if Retell requires fast acknowledgment
    - validate AI output types, especially `leadScore`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Retell AI Call Analyzer → HubSpot. Automatically analyze voice call transcripts and sync AI-powered insights to your CRM. | Workflow-level purpose |
| How it works: 1. Webhook receives transcript from Retell AI when a call ends. 2. Parse extracts transcript text, caller phone number, and metadata. 3. OpenAI analyzes sentiment, lead score, and action items. 4. HubSpot updates the contact record. 5. Hot leads trigger Slack alert + HubSpot follow-up task. Every call is logged to Sheets. | Workflow-level logic summary |
| Setup steps: 1. Open “Set user config variables” and enter Slack channel, score threshold, and Sheet ID. 2. Connect OpenAI, HubSpot, Slack, and Google Sheets credentials in each node. 3. Copy the webhook URL into Retell dashboard → Webhook Settings → `call_analyzed`. 4. Activate and test with a call. | Operational setup guidance |

## Additional implementation notes

- The workflow has **one entry point**: **Receive Retell call webhook**.
- The workflow includes **no sub-workflows** and **no Execute Workflow nodes**.
- A notable design risk is that the **Respond to Webhook** node is only reached through the HubSpot contact update branch. If that HubSpot update fails, the webhook response may never be sent.
- Another important risk is the use of **caller phone number as HubSpot contact ID**. In many HubSpot integrations, this will not work without first searching for the contact and then updating by internal ID.
- The Google Sheets and Slack destinations are intentionally centralized in the config node for easier modification without editing multiple downstream nodes.