Generate Seedance crowd previs passes from chat using Azure OpenAI

https://n8nworkflows.xyz/workflows/generate-seedance-crowd-previs-passes-from-chat-using-azure-openai-14884


# Generate Seedance crowd previs passes from chat using Azure OpenAI

Now I have all the details mapped out. Let me write the comprehensive documentation. 1. Workflow Overview

**Seedance Crowd Simulation Previs with Chat Input and Pass Generation** is a production-oriented workflow that turns a conversational crowd brief into multiple rendered previs passes via the Seedance crowd-simulation service, then packages and delivers the results to a layout team through three parallel channels (Slack, Gmail, Google Sheets).

The workflow is organised into five sequential logical blocks plus an overarching overview note and a security credentials note:

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Trigger & AI Parsing** | Accepts a free-text crowd brief through a chat interface, uses an Azure OpenAI-powered LangChain Agent to parse the brief into structured pass definitions. |
| 2 | **Pass Builder & Submission** | Transforms the AI output into Seedance API request payloads, then submits each pass as an HTTP request to the Seedance rendering service. |
| 3 | **Job Tracking & Polling Loop** | Stores the returned Job ID alongside pass metadata, polls the Seedance API for job completion at 20-second intervals until all passes report as done. |
| 4 | **Package Compilation** | Collects all completed pass results and compiles them into a single previs package (metadata, URLs, layout guidance). |
| 5 | **Multi-Channel Delivery** | Broadcasts the compiled package to the layout team via Slack message, Gmail email, and a Google Sheets row append — all in parallel. |

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & AI Parsing

**Overview:** This block is the entry point. A user interacts with a chat trigger to describe the desired crowd simulation (e.g., "dense medieval marketplace, 500 agents, wide shot"). The AI Agent, backed by an Azure OpenAI chat model, interprets the brief and outputs a structured representation of each previs pass (camera angle, crowd density, behaviour, etc.).

**Nodes Involved:**
- Chat: Receive Crowd Brief
- AI Agent
- Azure OpenAI Chat Model

**Sticky Note:** Section: Trigger & AI Parsing

---

#### Chat: Receive Crowd Brief

| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.chatTrigger` v1.4 |
| **Technical Role** | Webhook-based chat entry point; exposes a conversational UI that captures user messages and forwards them to the connected AI Agent. |
| **Configuration** | No custom parameters set (defaults apply): a webhook ID `crowd-chat-webhook-001` is auto-assigned for the endpoint. |
| **Key Expressions / Variables** | None explicitly configured; the chat message text is implicitly passed to the downstream Agent node. |
| **Input Connections** | None (entry node). |
| **Output Connections** | → AI Agent (main, index 0) |
| **Edge Cases / Failure Types** | — Webhook URL not reachable (firewall / n8n not exposed). — Empty or non-text message payloads. — Concurrent sessions may collide if the same webhook ID is reused. |
| **Version-Specific Notes** | v1.4 of the chatTrigger requires n8n ≥ 1.0 with LangChain nodes enabled. |

---

#### AI Agent

| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.agent` v2.1 |
| **Technical Role** | LangChain Agent that receives the user's crowd brief text, applies a system prompt (expected to be configured at runtime), and returns a structured JSON array of pass definitions. |
| **Configuration** | No custom parameters are hardcoded in the JSON — the agent's system prompt, tools, and output schema are expected to be configured in the n8n editor. The agent is connected to an Azure OpenAI Chat Model as its language model. |
| **Key Expressions / Variables** | Consumes the chat message from the upstream chat trigger; emits a text response that downstream Code nodes will parse. |
| **Input Connections** | ← Chat: Receive Crowd Brief (main, index 0); ← Azure OpenAI Chat Model (ai_languageModel, index 0) |
| **Output Connections** | → Structure Passes as Batch Items (main, index 0) |
| **Edge Cases / Failure Types** | — Azure OpenAI returns an error (quota, key rotation, model unavailable). — Agent output is not valid JSON (parsing failure in downstream Code node). — Rate limiting on Azure OpenAI endpoint. — Timeout if prompt is excessively long. |
| **Version-Specific Notes** | v2.1 of the Agent node supports tool-calling and output parsers. Ensure the system prompt constrains output to a parseable JSON array. |

---

#### Azure OpenAI Chat Model

| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` v1 |
| **Technical Role** | Provides the LLM backend for the AI Agent. Communicates with Azure OpenAI Service (not public OpenAI). |
| **Configuration** | No parameters are hardcoded — requires an Azure OpenAI credential to be configured in n8n (API key, endpoint URL, deployment name, API version). |
| **Key Expressions / Variables** | None in the workflow; the credential object supplies endpoint and model details at runtime. |
| **Input Connections** | None (sub-node attached to AI Agent). |
| **Output Connections** | → AI Agent (ai_languageModel, index 0) |
| **Edge Cases / Failure Types** | — Invalid or expired Azure API key. — Wrong deployment name. — Azure OpenAI resource in a different region causing latency or CORS issues. — Token limit exceeded for the chosen model. |
| **Version-Specific Notes** | v1. Requires the `@n8n/n8n-nodes-langchain` package. Ensure the Azure deployment supports chat completions (e.g., `gpt-4o`, `gpt-35-turbo`). |

---

### Block 2 — Pass Builder & Submission

**Overview:** This block takes the AI Agent's textual output (a JSON description of multiple previs passes), structures each pass into a batch item, constructs the proper Seedance API request payload for each, and fires off the HTTP submissions.

**Nodes Involved:**
- Structure Passes as Batch Items
- Build Seedance Request
- Seedance: Submit Pass

**Sticky Note:** Section: Pass Builder & Submission

---

#### Structure Passes as Batch Items

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` v2 |
| **Technical Role** | Code node that parses the AI Agent's text output into an array of individual items, each representing one previs pass. This creates the batch structure needed for downstream per-pass processing. |
| **Configuration** | No hardcoded parameters — JavaScript code is expected to be authored in the node editor. The likely logic: parse the incoming JSON string, iterate over pass definitions, and return each as a separate n8n item (`return items.map(...)`). |
| **Key Expressions / Variables** | Consumes `$input.first().json` (the agent's output); produces multiple items with pass-specific fields (e.g., `passName`, `cameraAngle`, `agentCount`, `behaviourProfile`). |
| **Input Connections** | ← AI Agent (main, index 0) |
| **Output Connections** | → Build Seedance Request (main, index 0) |
| **Edge Cases / Failure Types** | — AI output is not valid JSON → `JSON.parse` throws. — Empty array returned → downstream nodes receive no items and the workflow halts silently. — Schema mismatch between AI output and expected fields. |
| **Version-Specific Notes** | v2 Code node supports async/await and `$input.all()`. |

---

#### Build Seedance Request

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` v2 |
| **Technical Role** | Code node that transforms each pass item into the exact HTTP request body and headers required by the Seedance API. This includes mapping internal field names to Seedance's schema, injecting authentication tokens, and setting request metadata. |
| **Configuration** | No hardcoded parameters — JavaScript code is expected in the editor. Likely builds an object with `method`, `url`, `headers`, and `body` fields, or directly sets properties on the item that the downstream HTTP Request node will consume. |
| **Key Expressions / Variables** | Reads per-pass fields from the incoming item; may reference environment variables or credentials for the Seedance API key. |
| **Input Connections** | ← Structure Passes as Batch Items (main, index 0) |
| **Output Connections** | → Seedance: Submit Pass (main, index 0) |
| **Edge Cases / Failure Types** | — Missing required Seedance fields → API 400 error. — Stale API key embedded in code (should use n8n credential). — Large payload exceeding Seedance body size limit. |
| **Version-Specific Notes** | v2 Code node; ensure the output item includes the fields the HTTP Request node expects (url, headers, body). |

---

#### Seedance: Submit Pass

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` v4.3 |
| **Technical Role** | Sends an HTTP request to the Seedance crowd-simulation API to start rendering each previs pass. Each item from the previous node triggers a separate request (batch execution). |
| **Configuration** | No parameters are hardcoded in the JSON — must be configured in-editor. Expected settings: HTTP method (likely POST), URL (Seedance API endpoint), authentication header, JSON body constructed by the upstream Code node. |
| **Key Expressions / Variables** | May reference `{{$json.url}}`, `{{$json.body}}` from the upstream Code node, or use static values. |
| **Input Connections** | ← Build Seedance Request (main, index 0) |
| **Output Connections** | → Store Job ID with Pass Data (main, index 0) |
| **Edge Cases / Failure Types** | — Network timeout (Seedance service down). — 401/403 auth error. — 429 rate limiting. — 5xx server error. — Non-JSON response body. — SSL certificate issues. |
| **Version-Specific Notes** | v4.3 HTTP Request node supports pagination, retry on fail, and response format options. Configure "Respond on" appropriately to avoid workflow hanging. |

---

### Block 3 — Job Tracking & Polling Loop

**Overview:** After each pass is submitted, the Seedance API returns a Job ID. This block stores that ID alongside the original pass metadata, then enters a polling loop: it checks the job status every 20 seconds until the job is marked complete (or failed). If the job is still running, the Wait node pauses execution and loops back to polling.

**Nodes Involved:**
- Store Job ID with Pass Data
- Poll: Job Status
- Done?
- Wait 20s1

**Sticky Note:** Section: Polling Loop

---

#### Store Job ID with Pass Data

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` v2 |
| **Technical Role** | Merges the Seedance API response (containing a Job ID) with the original pass metadata (camera, density, etc.) into a single item for downstream tracking. |
| **Configuration** | No hardcoded parameters — JavaScript code is expected to combine `previousNodeResult` data with the current item's JSON. |
| **Key Expressions / Variables** | Likely reads `$input.first().json.jobId` or similar from the Seedance response; merges with pass metadata fields. |
| **Input Connections** | ← Seedance: Submit Pass (main, index 0) |
| **Output Connections** | → Poll: Job Status (main, index 0) |
| **Edge Cases / Failure Types** | — Seedance response doesn't contain a Job ID → downstream poll fails. — Race condition if multiple passes share state. |
| **Version-Specific Notes** | v2 Code node. |

---

#### Poll: Job Status

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` v4.3 |
| **Technical Role** | Sends a GET (or POST) request to the Seedance API to check whether the submitted job has completed. |
| **Configuration** | No parameters hardcoded — must be configured with the Seedance status-check endpoint, authentication, and a reference to the Job ID (likely via an expression like `{{$json.jobId}}`). |
| **Key Expressions / Variables** | `{{$json.jobId}}` interpolated into the URL or query parameters. |
| **Input Connections** | ← Store Job ID with Pass Data (main, index 0); ← Wait 20s1 (main, index 0 — loop-back) |
| **Output Connections** | → Done? (main, index 0) |
| **Edge Cases / Failure Types** | — Same HTTP errors as Seedance: Submit Pass. — Job status field name changes in Seedance API. — Transient network error misinterpreted as "failed" job. — Infinite loop if the "Done?" condition is misconfigured (e.g., always returns false). |
| **Version-Specific Notes** | v4.3 HTTP Request node. Consider enabling "Retry on Fail" for transient errors. |

---

#### Done?

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.if` v2 |
| **Technical Role** | Conditional branch that checks whether the Seedance job status indicates completion. True → proceed to collect results. False → route to the Wait node for another polling cycle. |
| **Configuration** | No parameters hardcoded — must be configured with a condition expression, e.g., `{{$json.status}}` equals `"completed"` or `"done"`. |
| **Key Expressions / Variables** | `{{$json.status}}` — the field returned by the Poll node that indicates job completion. |
| **Input Connections** | ← Poll: Job Status (main, index 0) |
| **Output Connections** | True branch → Collect Completed Pass (main, index 0); False branch → Wait 20s1 (main, index 1) |
| **Edge Cases / Failure Types** | — Condition expression references a non-existent field → always evaluates false → infinite loop. — Seedance returns a "failed" status that is neither "completed" nor "running" — may need a third branch or error handling. — Case-sensitivity mismatch (e.g., "Completed" vs "completed"). |
| **Version-Specific Notes** | v2 IF node supports boolean, string, and number comparisons. |

---

#### Wait 20s1

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.wait` v1.1 |
| **Technical Role** | Pauses workflow execution for 20 seconds before re-polling the Seedance job status. Implements the polling interval. |
| **Configuration** | No parameters hardcoded — must be configured with a time value (20 seconds) and a resume mechanism (webhook ID `crowd-wait-001` is auto-assigned). |
| **Key Expressions / Variables** | None. |
| **Input Connections** | ← Done? (false branch, main, index 1) |
| **Output Connections** | → Poll: Job Status (main, index 0 — loop-back) |
| **Edge Cases / Failure Types** | — Webhook for resumption is not reachable (n8n behind firewall, URL changed). — Wait time too short → Seedance rate-limiting on status endpoint. — Wait time too long → overall latency unacceptable for large pass counts. — Execution TTL exceeded if there are many passes and long render times. |
| **Version-Specific Notes** | v1.1 Wait node uses a webhook to resume. The workflow must be active (or in testing mode with the webhook accessible). |

---

### Block 4 — Package Compilation

**Overview:** Once all passes have completed rendering, this block gathers each pass's output (render URLs, metadata) and compiles them into a unified previs package containing layout guidance, thumbnail links, and a summary.

**Nodes Involved:**
- Collect Completed Pass
- Compile Full Previs Package

**Sticky Note:** Section: Package Compilation

---

#### Collect Completed Pass

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` v2 |
| **Technical Role** | Aggregates the completed pass data from the polling loop. Since each pass runs through its own polling cycle independently, this node likely accumulates results into a single array or merges items that arrive from multiple parallel execution paths. |
| **Configuration** | No hardcoded parameters — JavaScript code is expected. May use `$input.all()` to gather all items and normalise them. |
| **Key Expressions / Variables** | Reads per-pass completed data (job result URL, status, pass name); outputs a combined structure. |
| **Input Connections** | ← Done? (true branch, main, index 0) |
| **Output Connections** | → Compile Full Previs Package (main, index 0) |
| **Edge Cases / Failure Types** | — If only some passes complete and others are still in loop, partial data may reach this node (depends on execution model). — Mismatched field names across passes. — Memory pressure with very large result payloads. |
| **Version-Specific Notes** | v2 Code node. |

---

#### Compile Full Previs Package

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` v2 |
| **Technical Role** | Produces the final previs package: a structured object containing all pass results, a layout guide (textual description of how to use the previs in the scene layout), and summary metadata. This is the payload delivered to the layout team. |
| **Configuration** | No hardcoded parameters — JavaScript code is expected to assemble the package. May generate an HTML or Markdown layout guide, construct a summary table, and attach render URLs. |
| **Key Expressions / Variables** | Reads the aggregated pass data from the previous node; outputs fields like `packageTitle`, `layoutGuide`, `passes` (array), `summaryUrl`, etc. |
| **Input Connections** | ← Collect Completed Pass (main, index 0) |
| **Output Connections** | → Slack: Deliver to Layout Team (main, index 0); → Email: Layout Guide to Team (main, index 0); → Append The Data in the Sheet (main, index 0) |
| **Edge Cases / Failure Types** | — Empty pass array → empty package delivered. — Generated layout guide too long for Slack message limit (~40 000 chars but practical limit lower). — Encoding issues with special characters in the guide. |
| **Version-Specific Notes** | v2 Code node. |

---

### Block 5 — Multi-Channel Delivery

**Overview:** The compiled previs package is broadcast in parallel through three channels: a Slack message to the layout team channel, a Gmail email with the layout guide, and a Google Sheets row append for tracking and archival. This ensures team members receive the information through their preferred medium and a persistent record exists.

**Nodes Involved:**
- Slack: Deliver to Layout Team
- Email: Layout Guide to Team
- Append The Data in the Sheet

**Sticky Notes:** Section: Delivery; Security: Credentials Note

---

#### Slack: Deliver to Layout Team

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.slack` v2.3 |
| **Technical Role** | Sends a message to a designated Slack channel containing the previs package summary, links to rendered passes, and the layout guide. |
| **Configuration** | No parameters hardcoded — must be configured with: Slack credential (Bot User OAuth Token), channel ID or name, message text (likely referencing `{{$json.layoutGuide}}` and pass URLs). |
| **Key Expressions / Variables** | `{{$json.layoutGuide}}`, `{{$json.passes}}`, `{{$json.packageTitle}}` — expected to be interpolated into the message body. |
| **Input Connections** | ← Compile Full Previs Package (main, index 0) |
| **Output Connections** | None (terminal node). |
| **Edge Cases / Failure Types** | — Slack token revoked or expired. — Bot not invited to the target channel. — Message exceeds Slack's block kit or text length limits. — Rate limiting on Slack API. — Channel archived. |
| **Version-Specific Notes** | v2.3 Slack node. Uses Bot Token scope; requires `chat:write` and `chat:write.public` permissions. |

---

#### Email: Layout Guide to Team

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.gmail` v2.2 |
| **Technical Role** | Sends an email via Gmail containing the layout guide and previs package details to the layout team distribution list or individual addresses. |
| **Configuration** | No parameters hardcoded — must be configured with: Gmail OAuth2 credential, recipient email(s), subject line, body (HTML or plain text, likely referencing `{{$json.layoutGuide}}`). |
| **Key Expressions / Variables** | `{{$json.layoutGuide}}`, `{{$json.packageTitle}}` in subject/body. |
| **Input Connections** | ← Compile Full Previs Package (main, index 0) |
| **Output Connections** | None (terminal node). |
| **Edge Cases / Failure Types** | — Gmail OAuth2 token expired or revoked. — Daily sending limit exceeded (Gmail limit: ~500/day for free accounts). — Recipient address invalid. — Email marked as spam. — Attachment size too large (if render files are attached rather than linked). |
| **Version-Specific Notes** | v2.2 Gmail node uses OAuth2. Requires Google Cloud project with Gmail API enabled and appropriate scopes. |

---

#### Append The Data in the Sheet

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.googleSheets` v4.7 |
| **Technical Role** | Appends a row to a Google Sheet for persistent tracking of all previs passes, their status, and results. Serves as the canonical audit log. |
| **Configuration** | No parameters hardcoded — must be configured with: Google Sheets OAuth2 credential, spreadsheet ID or URL, sheet name, column mapping (pass name, job ID, status, render URL, timestamp, etc.). |
| **Key Expressions / Variables** | Maps fields from `{{$json}}` to spreadsheet columns (e.g., `{{$json.passes[0].name}}`, `{{$json.passes[0].renderUrl}}`). |
| **Input Connections** | ← Compile Full Previs Package (main, index 0) |
| **Output Connections** | None (terminal node). |
| **Edge Cases / Failure Types** | — Google Sheets credential expired. — Spreadsheet deleted or permissions revoked. — Column count mismatch → data appended incorrectly. — Sheet is protected or locked. — Rate limiting on Google Sheets API. — Data exceeds cell character limit (50 000 chars). |
| **Version-Specific Notes** | v4.7 Google Sheets node supports append mode with auto-column mapping or explicit column assignment. |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Chat: Receive Crowd Brief | @n8n/n8n-nodes-langchain.chatTrigger v1.4 | Chat webhook entry point for crowd briefs | — | AI Agent | Section: Trigger & AI Parsing |
| AI Agent | @n8n/n8n-nodes-langchain.agent v2.1 | LangChain Agent that parses crowd brief into structured pass definitions | Chat: Receive Crowd Brief; Azure OpenAI Chat Model | Structure Passes as Batch Items | Section: Trigger & AI Parsing |
| Azure OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi v1 | LLM backend (Azure OpenAI) for the AI Agent | — | AI Agent (ai_languageModel) | Section: Trigger & AI Parsing |
| Structure Passes as Batch Items | n8n-nodes-base.code v2 | Parses AI output JSON into individual n8n items (one per pass) | AI Agent | Build Seedance Request | Section: Pass Builder & Submission |
| Build Seedance Request | n8n-nodes-base.code v2 | Constructs Seedance API request payload for each pass | Structure Passes as Batch Items | Seedance: Submit Pass | Section: Pass Builder & Submission |
| Seedance: Submit Pass | n8n-nodes-base.httpRequest v4.3 | Submits render request to Seedance API | Build Seedance Request | Store Job ID with Pass Data | Section: Pass Builder & Submission |
| Store Job ID with Pass Data | n8n-nodes-base.code v2 | Merges Seedance Job ID with pass metadata for tracking | Seedance: Submit Pass | Poll: Job Status | Section: Polling Loop |
| Poll: Job Status | n8n-nodes-base.httpRequest v4.3 | Checks Seedance job completion status | Store Job ID with Pass Data; Wait 20s1 | Done? | Section: Polling Loop |
| Done? | n8n-nodes-base.if v2 | Routes: true → collect results; false → wait and re-poll | Poll: Job Status | Collect Completed Pass (true); Wait 20s1 (false) | Section: Polling Loop |
| Wait 20s1 | n8n-nodes-base.wait v1.1 | 20-second pause between polling attempts | Done? (false) | Poll: Job Status | Section: Polling Loop |
| Collect Completed Pass | n8n-nodes-base.code v2 | Aggregates completed pass results | Done? (true) | Compile Full Previs Package | Section: Package Compilation |
| Compile Full Previs Package | n8n-nodes-base.code v2 | Assembles final previs package with layout guide | Collect Completed Pass | Slack: Deliver to Layout Team; Email: Layout Guide to Team; Append The Data in the Sheet | Section: Package Compilation |
| Slack: Deliver to Layout Team | n8n-nodes-base.slack v2.3 | Sends previs package summary to Slack channel | Compile Full Previs Package | — | Section: Delivery; Security: Credentials Note |
| Email: Layout Guide to Team | n8n-nodes-base.gmail v2.2 | Emails layout guide and previs details to team | Compile Full Previs Package | — | Section: Delivery; Security: Credentials Note |
| Append The Data in the Sheet | n8n-nodes-base.googleSheets v4.7 | Appends pass results row to tracking spreadsheet | Compile Full Previs Package | — | Section: Delivery; Security: Credentials Note |
| Overview: AI Crowd Previs | n8n-nodes-base.stickyNote v1 | Visual overview note | — | — | Overview: AI Crowd Previs (covers all blocks at canvas level) |
| Section: Trigger & AI Parsing | n8n-nodes-base.stickyNote v1 | Visual section label | — | — | — |
| Section: Pass Builder & Submission | n8n-nodes-base.stickyNote v1 | Visual section label | — | — | — |
| Section: Polling Loop | n8n-nodes-base.stickyNote v1 | Visual section label | — | — | — |
| Section: Package Compilation | n8n-nodes-base.stickyNote v1 | Visual section label | — | — | — |
| Section: Delivery | n8n-nodes-base.stickyNote v1 | Visual section label | — | — | — |
| Security: Credentials Note | n8n-nodes-base.stickyNote v1 | Visual reminder about credentials security | — | — | — |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it `Seedance Crowd Simulation Previs with Chat Input and Pass Generation`.

2. **Add sticky notes** for visual organisation (optional but recommended):
   - "Overview: AI Crowd Previs" — place at the top-left of the canvas.
   - "Section: Trigger & AI Parsing" — place around the first three nodes.
   - "Section: Pass Builder & Submission" — place around nodes 4–6.
   - "Section: Polling Loop" — place around nodes 7–10.
   - "Section: Package Compilation" — place around nodes 11–12.
   - "Section: Delivery" — place around nodes 13–15.
   - "Security: Credentials Note" — place near the delivery nodes with a reminder to use n8n credential stores, not hardcoded keys.

3. **Add the Chat Trigger node:**
   - Type: `Chat Trigger` (`@n8n/n8n-nodes-langchain.chatTrigger`), version 1.4.
   - Leave all parameters at defaults. Note the generated webhook ID.
   - This will be the workflow's entry point.

4. **Add the AI Agent node:**
   - Type: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`), version 2.1.
   - Configure the system prompt to instruct the model to return a JSON array of pass definitions. Example prompt constraint: *"You are a crowd-simulation previs assistant. Given a crowd brief, output a JSON array where each element has: passName (string), cameraAngle (string), agentCount (number), behaviourProfile (string), durationSeconds (number). Output ONLY valid JSON, no markdown."*
   - Do not add any tools unless needed.
   - Connect: Chat Trigger → AI Agent (main output → main input).

5. **Add the Azure OpenAI Chat Model node:**
   - Type: `Azure OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`), version 1.
   - Configure:
     - **Credential:** Create a new Azure OpenAI API credential with your Azure endpoint, API key, deployment name (e.g., `gpt-4o`), and API version (e.g., `2024-02-15-preview`).
     - **Model / Deployment:** Select the deployment that matches your Azure resource.
   - Connect: Azure OpenAI Chat Model → AI Agent (ai_languageModel output → ai_languageModel input).

6. **Add the "Structure Passes as Batch Items" Code node:**
   - Type: `Code` (`n8n-nodes-base.code`), version 2.
   - Language: JavaScript.
   - Code logic:
     - Parse the AI Agent's text output: `const passes = JSON.parse($input.first().json.output || $input.first().json.text);`
     - Return each pass as a separate item: `return passes.map(p => ({ json: p }));`
   - Connect: AI Agent → Structure Passes as Batch Items (main → main).

7. **Add the "Build Seedance Request" Code node:**
   - Type: `Code`, version 2.
   - Code logic: For each incoming item, construct a Seedance API request object:
     ```javascript
     return items.map(item => ({
       json: {
         url: 'https://api.seedance.example/v1/renders',
         method: 'POST',
         headers: { 'Authorization': 'Bearer YOUR_SEEDANCE_KEY', 'Content-Type': 'application/json' },
         body: {
           passName: item.json.passName,
           camera: item.json.cameraAngle,
           agentCount: item.json.agentCount,
           behaviour: item.json.behaviourProfile,
           duration: item.json.durationSeconds
         }
       }
     }));
     ```
   - **Important:** Replace `YOUR_SEEDANCE_KEY` with a reference to an n8n credential or environment variable rather than a hardcoded string.
   - Connect: Structure Passes as Batch Items → Build Seedance Request.

8. **Add the "Seedance: Submit Pass" HTTP Request node:**
   - Type: `HTTP Request` (`n8n-nodes-base.httpRequest`), version 4.3.
   - Configuration:
     - **Method:** POST
     - **URL:** `{{$json.url}}` (or the static Seedance endpoint if you prefer)
     - **Authentication:** Header Auth or Generic Credential — set the Authorization header via a credential.
     - **Body Content Type:** JSON
     - **Body:** `{{$json.body}}`
     - **Response Format:** JSON
   - Enable "Retry on Fail" with 3 retries and 1-second interval for resilience.
   - Connect: Build Seedance Request → Seedance: Submit Pass.

9. **Add the "Store Job ID with Pass Data" Code node:**
   - Type: `Code`, version 2.
   - Code logic: Merge the Seedance response (job ID) with the original pass data:
     ```javascript
     return items.map(item => ({
       json: {
         ...item.json,
         jobId: item.json.id || item.json.jobId, // adapt to Seedance response schema
         passName: item.json.passName,
         cameraAngle: item.json.cameraAngle,
         status: 'submitted'
       }
     }));
     ```
   - Connect: Seedance: Submit Pass → Store Job ID with Pass Data.

10. **Add the "Poll: Job Status" HTTP Request node:**
    - Type: `HTTP Request`, version 4.3.
    - Configuration:
      - **Method:** GET
      - **URL:** `https://api.seedance.example/v1/renders/{{$json.jobId}}` (adapt to Seedance's actual status endpoint).
      - **Authentication:** Same Seedance credential as step 8.
      - **Response Format:** JSON
    - Enable "Retry on Fail" with 2 retries.
    - Connect: Store Job ID with Pass Data → Poll: Job Status.

11. **Add the "Done?" IF node:**
    - Type: `IF` (`n8n-nodes-base.if`), version 2.
    - Configuration:
      - **Condition:** Value: `{{$json.status}}`; Operation: `Equal`; Compare to: `completed` (adjust to Seedance's actual completed-state string).
    - Connect:
      - Poll: Job Status → Done? (main input).
      - Done? true output → Collect Completed Pass (step 13).
      - Done? false output → Wait 20s1 (step 12).

12. **Add the "Wait 20s1" Wait node:**
    - Type: `Wait` (`n8n-nodes-base.wait`), version 1.1.
    - Configuration:
      - **Resume:** After a specified time.
      - **Time:** 20 seconds.
    - Connect: Wait 20s1 → Poll: Job Status (creating the loop-back; Poll: Job Status now has two input sources).

13. **Add the "Collect Completed Pass" Code node:**
    - Type: `Code`, version 2.
    - Code logic: Gather all completed pass items into a single consolidated item:
      ```javascript
      const allPasses = $input.all().map(item => item.json);
      return [{ json: { passes: allPasses, totalPasses: allPasses.length, completedAt: new Date().toISOString() } }];
      ```
    - Connect: Done? (true) → Collect Completed Pass.

14. **Add the "Compile Full Previs Package" Code node:**
    - Type: `Code`, version 2.
    - Code logic: Build the full delivery payload:
      ```javascript
      const data = $input.first().json;
      const layoutGuide = data.passes.map(p =>
        `Pass "${p.passName}": ${p.cameraAngle}, ${p.agentCount} agents, behaviour "${p.behaviourProfile}". Render URL: ${p.renderUrl || 'pending'}`
      ).join('\n');
      return [{ json: {
        packageTitle: 'Crowd Previs Package',
        layoutGuide: layoutGuide,
        passes: data.passes,
        totalPasses: data.totalPasses,
        completedAt: data.completedAt
      }}];
      ```
    - Connect: Collect Completed Pass → Compile Full Previs Package.

15. **Add the "Slack: Deliver to Layout Team" node:**
    - Type: `Slack` (`n8n-nodes-base.slack`), version 2.3.
    - Configuration:
      - **Credential:** Create a Slack Bot Token credential (Bot User OAuth Token with `chat:write` scope).
      - **Resource:** Message.
      - **Channel:** Select or enter the layout team channel ID.
      - **Text:** `{{$json.packageTitle}}\n\n{{$json.layoutGuide}}`
    - Connect: Compile Full Previs Package → Slack: Deliver to Layout Team.

16. **Add the "Email: Layout Guide to Team" node:**
    - Type: `Gmail` (`n8n-nodes-base.gmail`), version 2.2.
    - Configuration:
      - **Credential:** Create a Gmail OAuth2 credential (Google Cloud project with Gmail API, scopes: `mail.send`).
      - **To:** Layout team email distribution list.
      - **Subject:** `{{$json.packageTitle}} — {{$json.completedAt}}`
      - **Body (HTML):** Render `{{$json.layoutGuide}}` with appropriate formatting.
    - Connect: Compile Full Previs Package → Email: Layout Guide to Team.

17. **Add the "Append The Data in the Sheet" node:**
    - Type: `Google Sheets` (`n8n-nodes-base.googleSheets`), version 4.7.
    - Configuration:
      - **Credential:** Create a Google Sheets OAuth2 credential.
      - **Operation:** Append Row.
      - **Spreadsheet:** Select the target tracking spreadsheet.
      - **Sheet Name:** Select the target sheet (e.g., "Previs Log").
      - **Column Mapping:** Map fields — Pass Name, Job ID, Camera Angle, Agent Count, Behaviour, Render URL, Status, Timestamp — to the corresponding spreadsheet columns.
    - Connect: Compile Full Previs Package → Append The Data in the Sheet.

18. **Review all connections** to ensure they match:
    - Chat Trigger → AI Agent
    - Azure OpenAI Chat Model → AI Agent (ai_languageModel)
    - AI Agent → Structure Passes as Batch Items
    - Structure Passes as Batch Items → Build Seedance Request
    - Build Seedance Request → Seedance: Submit Pass
    - Seedance: Submit Pass → Store Job ID with Pass Data
    - Store Job ID with Pass Data → Poll: Job Status
    - Poll: Job Status → Done?
    - Done? (true) → Collect Completed Pass
    - Done? (false) → Wait 20s1
    - Wait 20s1 → Poll: Job Status
    - Collect Completed Pass → Compile Full Previs Package
    - Compile Full Previs Package → Slack: Deliver to Layout Team
    - Compile Full Previs Package → Email: Layout Guide to Team
    - Compile Full Previs Package → Append The Data in the Sheet

19. **Activate the workflow** and test by sending a sample crowd brief through the chat trigger URL. Verify:
    - The AI Agent returns valid JSON.
    - Each pass is submitted to Seedance and a Job ID is returned.
    - The polling loop correctly waits and eventually resolves.
    - Slack, Gmail, and Google Sheets all receive the compiled package.

20. **Set the workflow to Active** once testing is complete. Ensure the n8n instance is publicly reachable so the Wait node's resume webhook and the Chat Trigger webhook are accessible.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow relies on a Seedance crowd-simulation API. The exact endpoint URLs, authentication scheme, and response schema must be obtained from the Seedance service documentation or account portal. | Seedance API (vendor-specific, no public URL implied) |
| Azure OpenAI credential requires an active Azure subscription with an OpenAI resource deployed. The deployment name, API version, and region must all align. | Azure OpenAI Service |
| The polling loop uses a 20-second interval. For long renders (many agents, complex behaviours), consider increasing to 60 seconds or adding a max-poll-count safeguard to prevent infinite loops. | Performance tuning |
| The "Security: Credentials Note" sticky reminds that all API keys (Seedance, Azure, Slack, Gmail, Google Sheets) should be stored in n8n's built-in credential store and never hardcoded in Code node bodies. | n8n Credentials Documentation |
| Slack Bot Token requires the `chat:write` and `chat:write.public` OAuth scopes. The bot must be invited to the target channel before first use. | Slack API Scopes |
| Gmail OAuth2 may require a Google Workspace account for higher sending limits (free Gmail caps at ~500 emails/day). | Gmail API Quotas |
| Google Sheets node v4.7 supports `append` with `valueInputOption: USER_ENTERED` by default. Confirm that the target sheet has headers matching the mapped columns. | Google Sheets API |
| The workflow's binary mode is set to `separate` and execution order to `v1`. These are advanced settings that affect how binary data and node execution order are handled. | n8n Execution Settings |
| For production use, consider adding an error-handling branch to the "Done?" node to catch Seedance job failures (status = "failed" or "error") and notify the team via a dedicated Slack alert. | Resilience best practice |
| The Chat Trigger webhook ID (`crowd-chat-webhook-001`) and Wait node webhook ID (`crowd-wait-001`) are auto-generated but noted for reference if debugging webhook delivery issues. | Webhook troubleshooting |