Assign and track Paperclip tasks using GPT-5-mini, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/assign-and-track-paperclip-tasks-using-gpt-5-mini--google-sheets--and-gmail-14976


# Assign and track Paperclip tasks using GPT-5-mini, Google Sheets, and Gmail

Let me now build the full document based on this analysis.

For the sticky note mapping to nodes:
- Sticky Note (id 4e00ed6c): Position [-464,-688], width 1152, height 400 - This is a big overview note above the workflow. It doesn't directly cover specific nodes in the canvas area where nodes are placed (nodes start around y=-128). So this is a general overview note - not directly covering any specific node. I'll leave it as not covering any nodes.

- Sticky Note1 (id 0db41d74): Position [-464,-272], width 432, height 512 - covers x: -464 to -32, y: -272 to 240. This covers Manual Trigger at [-640,-128] (x: -640 is outside the x range), Webhook at [-416,80] (x: -416 is inside, y: 80 is inside), Set data at [-416,-128] (inside). So it covers Webhook and Set data. Manual Trigger is at x=-640 which is outside the -464 to -32 range. But the sticky note title says "STEP 1 - Receives an issue" and mentions "Receives a task with title and issue via webhook or Set Data for testing" - so it's about the input reception nodes. Let me include Manual Trigger too since the content references it conceptually even if the position doesn't perfectly overlap.

Actually, let me re-check: Manual trigger is at [-640, -128]. Sticky Note1 covers x from -464 to -32, y from -272 to 240. -640 is outside the x range. So technically the sticky note doesn't cover the manual trigger node position. But the content clearly describes that step. I'll include it in all three nodes (Manual Trigger, Webhook, Set data) since the content is about receiving an issue.

- Sticky Note2 (id 65dc1fc5): Position [-16,-272], width 1280, height 512 - covers x: -16 to 1264, y: -272 to 240. Nodes in this area: Get Company ID at [-192,-32] (x=-192 is outside), Get Paperclip Agents at [32,-32] (inside), Normalize at [256,-32] (inside), OpenAI Chat Model at [448,48] (inside), Structured Output Parser at [640,48] (inside), Task Assignment at [480,-128] (inside), Create issue at [832,-32] (inside), Add issue to Sheet at [1056,-32] (inside). Get Company ID at x=-192 is outside the range. But conceptually it's part of Step 2. I'll include Get Company ID in this sticky note coverage since the content describes "Retrieves all available Paperclip agents for that company."

- Sticky Note3 (id 0a370693): Position [-16,256], width 1296, height 512 - covers x: -16 to 1280, y: 256 to 768. Nodes in this area: If at [544,432] (inside), Send alert at [848,432] (inside), Update issue to Sheet at [1088,496] (inside).

- Sticky Note4 (id 4a014ff4): Position [-464,256], width 432, height 512 - covers x: -464 to -32, y: 256 to 768. Nodes in this area: Schedule Trigger at [-656,496] (x=-656 is outside), Get open issues at [-304,496] (inside), Loop Over Items at [96,496] (x=96 is outside). I'll include all three since the content says "Fetches all open issues" and describes Step 3.

- Sticky Note9 (id 051cfeb0): Position [720,-1024] - separate area, YouTube promo. No functional nodes covered.

Now let me construct the document.

Let me verify the connections more carefully:

From the JSON connections:
- If → main[0] → Send alert (true branch)
- If → main[1] → Loop Over Items (false branch)
- Webhook → Get Company ID
- Set data → Get Company ID
- Get issue → If
- Normalize → Task Assignment
- Send alert → Update issue to Sheet
- Create issue → Add issue to Sheet
- Get Company ID → Get Paperclip Agents
- Get open issues → Loop Over Items
- Loop Over Items → main[0] (done/first batch) → nothing (empty array)
- Loop Over Items → main[1] (items to process) → Get issue
- Task Assignment → Create issue
- Schedule Trigger → Get open issues
- OpenAI Chat Model → Task Assignment (ai_languageModel)
- Add issue to Sheet → (empty output)
- Get Paperclip Agents → Normalize
- Update issue to Sheet → Loop Over Items
- Structured Output Parser → Task Assignment (ai_outputParser)
- When clicking 'Execute workflow' → Set data

Wait, let me re-read the Loop Over Items connections:
"Loop Over Items":{"main":[[],[{"node":"Get issue","type":"main","index":0}]]}

So main[0] is empty (done branch), main[1] goes to Get issue. That means when there are items to process, it sends to Get issue. When all items are done, the first output is empty (no further action).

And "If":{"main":[[{"node":"Send alert","type":"main","index":0}],[{"node":"Loop Over Items","type":"main","index":0}]]}

So If true → Send alert, If false → Loop Over Items (which means back to the loop to continue processing)

And Update issue to Sheet → Loop Over Items (back to loop)

So the flow is:
1. Schedule Trigger → Get open issues → Loop Over Items
2. Loop Over Items (has items) → Get issue → If
3. If status = completed → Send alert → Update issue to Sheet → Loop Over Items (back to loop)
4. If status ≠ completed → Loop Over Items (back to loop, skip this item)
5. Loop Over Items (done) → (nothing, workflow ends)

This makes sense - it processes each open issue one by one, checking status, and only acts on completed ones.

Now let me finalize the document structure. 1. Workflow Overview

**Paperclip Task Assigner & Completion Tracker** automates the full lifecycle of issue management in a Paperclip instance. It consists of two connected automations:

- **Flow A – Task Intake & AI-Powered Assignment (on-demand):** Receives a task (title + issue description) via webhook or manual trigger, discovers the company and its available Paperclip agents, uses GPT-5-mini to select the most appropriate agent, creates the issue in Paperclip, and logs it to a Google Sheet for centralised tracking.

- **Flow B – Scheduled Completion Monitoring (every 10 minutes):** Reads all open (uncompleted) issues from the Google Sheet, iterates over each one, queries the Paperclip API for the current status, and — when an issue is marked completed — sends a Gmail notification and writes the completion timestamp back into the Sheet.

**Logical Blocks:**

| Block | Purpose | Entry Point |
|---|---|---|
| 1.1 Input Reception | Receive task payload (`title`, `issue`) via webhook or manual trigger with static test data | Webhook / Manual Trigger |
| 1.2 Agent Discovery | Retrieve company ID and list of available Paperclip agents | Get Company ID |
| 1.3 AI Assignment | Use GPT-5-mini with a structured output parser to pick the best agent for the task | Task Assignment |
| 1.4 Issue Creation & Logging | Create the issue in Paperclip with the chosen assignee, then append a tracking row to Google Sheets | Create issue |
| 2.1 Scheduled Polling | Every 10 minutes, fetch all rows from the Sheet where the COMPLETED column is empty | Schedule Trigger |
| 2.2 Per-Issue Status Check | Loop over each open row, call the Paperclip API for the current issue status, branch on completion | Loop Over Items |
| 2.3 Notification & Sheet Update | If completed, send a Gmail alert and write the completion timestamp to the Sheet; if not, continue to the next item | If (true branch) |

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 – Input Reception

**Overview:** This block serves as the entry point for new tasks. It supports two modes: an HTTP POST webhook for production integration and a manual trigger with a Set node for testing. Both converge into the same downstream pipeline.

**Nodes Involved:** When clicking 'Execute workflow', Webhook, Set data

---

**When clicking 'Execute workflow'**  
- **Type:** `n8n-nodes-base.manualTrigger` (v1)  
- **Technical Role:** Manual execution trigger; allows the user to run the workflow from the n8n UI for testing.  
- **Configuration:** No parameters required.  
- **Key Expressions / Variables:** None.  
- **Input Connections:** None (entry node).  
- **Output Connections:** → Set data  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:** None inherent; will only fire on explicit user action.

---

**Webhook**  
- **Type:** `n8n-nodes-base.webhook` (v2.1)  
- **Technical Role:** Exposes an HTTP POST endpoint to receive external task payloads.  
- **Configuration:**  
  - HTTP Method: POST  
  - Path: `939dff15-712c-489a-885f-7912bedf5607` (auto-generated UUID)  
  - Options: default (no special response configuration)  
- **Key Expressions / Variables:** Expects a JSON body with at least `title` and `issue` fields.  
- **Input Connections:** None (entry node).  
- **Output Connections:** → Get Company ID  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - 401 if authentication is later added.  
  - 400 if the POST body is missing `title` or `issue`.  
  - The webhook path must be unique; re-activating the workflow regenerates the endpoint URL.  
- **Pinned Data:** Contains a test payload: `{"issue": "Write an article about AI automation and n8n", "title": "Create a blog article"}`.

---

**Set data**  
- **Type:** `n8n-nodes-base.set` (v3.4)  
- **Technical Role:** Provides hardcoded test values for `title` and `issue` when the workflow is run via the Manual Trigger.  
- **Configuration:**  
  - Assignment `title` (string): `"xxx"` (placeholder — replace with real test value)  
  - Assignment `issue` (string): `"xxx"` (placeholder — replace with real test value)  
- **Key Expressions / Variables:** None; static values.  
- **Input Connections:** ← When clicking 'Execute workflow'  
- **Output Connections:** → Get Company ID  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:** If placeholder `"xxx"` is left unchanged, the downstream AI assignment will receive meaningless input, potentially resulting in a poor or failed agent selection.

---

#### Block 1.2 – Agent Discovery

**Overview:** Retrieves the authenticated user's company ID from the Paperclip API, then fetches all agents belonging to that company. The agent list is normalised into a compact structure for the AI model.

**Nodes Involved:** Get Company ID, Get Paperclip Agents, Normalize

---

**Get Company ID**  
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)  
- **Technical Role:** Calls the Paperclip `/api/agents/me` endpoint to obtain the current user's `companyId`.  
- **Configuration:**  
  - Method: GET (default)  
  - URL: `https://paperclip.xxx.xxx/api/agents/me`  
  - Authentication: Generic Credential Type → HTTP Bearer Auth  
  - Credential: `Paperclip - Acme Corp (Hetzner)`  
- **Key Expressions / Variables:** The returned JSON must contain a `companyId` field.  
- **Input Connections:** ← Webhook OR ← Set data  
- **Output Connections:** → Get Paperclip Agents  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - 401 if the Bearer token is invalid or expired.  
  - 404 if the `/api/agents/me` endpoint doesn't exist on the target Paperclip instance.  
  - Network timeout if the Paperclip server is unreachable.  
  - The placeholder URL `paperclip.xxx.xxx` must be replaced with the real instance domain.

---

**Get Paperclip Agents**  
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)  
- **Technical Role:** Fetches the list of all agents for the discovered company.  
- **Configuration:**  
  - Method: GET (default)  
  - URL: `https://paperclip.xxx.xxx/api/companies/{{ $json.companyId }}/agents`  
  - Authentication: Generic Credential Type → HTTP Bearer Auth  
  - Credential: `Paperclip - Acme Corp (Hetzner)` (same as above)  
- **Key Expressions / Variables:** `{{ $json.companyId }}` — populated from the output of Get Company ID.  
- **Input Connections:** ← Get Company ID  
- **Output Connections:** → Normalize  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - Same auth/network risks as Get Company ID.  
  - If `companyId` is undefined (e.g., upstream node failed), the URL will be malformed, resulting in a request error.  
  - An empty agent list will still pass through; the AI will have no candidates to choose from, likely causing a failed or nonsensical assignment.

---

**Normalize**  
- **Type:** `n8n-nodes-base.code` (v2)  
- **Technical Role:** Transforms the raw agent array into a simplified list containing only `id`, `name`, and `title`, wrapped inside a `result` object for clean AI consumption.  
- **Configuration:**  
  - JavaScript code:
    ```javascript
    const data = items.map(item => item.json);
    const result = data.map(el => ({
      id: el.id,
      name: el.name,
      title: el.title
    }));
    return [{ json: { result: result } }];
    ```
- **Key Expressions / Variables:** Outputs a single item with `json.result` containing the agent array.  
- **Input Connections:** ← Get Paperclip Agents  
- **Output Connections:** → Task Assignment  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - If any agent object is missing `id`, `name`, or `title`, those fields will be `undefined` in the output.  
  - The node outputs exactly one item regardless of input count; the `result` array holds all agents.

---

#### Block 1.3 – AI Assignment

**Overview:** Uses an LLM chain (GPT-5-mini) with a structured output parser to select the single best Paperclip agent for the incoming task. The prompt combines the task details with the normalised agent list, and the parser enforces a response schema of `{id, name}`.

**Nodes Involved:** Task Assignment, OpenAI Chat Model, Structured Output Parser

---

**Task Assignment**  
- **Type:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.9)  
- **Technical Role:** Orchestrates the LLM call: builds the prompt from the task payload and agent list, invokes the connected model, and parses the structured output.  
- **Configuration:**  
  - Prompt Type: `define` (custom prompt)  
  - Prompt Text:
    ```
    Task title: {{ $('Webhook').item.json.title }}
    Task description: {{ $('Webhook').item.json.issue }}
    
    Agents: {{JSON.stringify($json.result)}}
    ```
  - System Message: `"Return only an array with a single agent, assigning it the most appropriate task for a Paperclip agent."`
  - Has Output Parser: true  
  - Batching: default (no batch size specified)  
- **Key Expressions / Variables:**  
  - `$('Webhook').item.json.title` — references the original webhook payload (also works when using Set data, as both feed into the same downstream path via Get Company ID; however, the expression explicitly references the Webhook node, which means it will fail if the workflow was triggered via Manual Trigger unless the Webhook node has pinned data or is executed in the same run).  
  - `$('Webhook').item.json.issue` — same caveat.  
  - `$json.result` — the normalised agent list from the Normalize node.  
- **Input Connections:** ← Normalize (main), ← OpenAI Chat Model (ai_languageModel), ← Structured Output Parser (ai_outputParser)  
- **Output Connections:** → Create issue  
- **Version Requirements:** Requires n8n LangChain nodes (available from n8n 1.0+ with LangChain integration enabled).  
- **Edge Cases / Failure Types:**  
  - If the Webhook node was not executed (Manual Trigger path), referencing `$('Webhook')` will throw a reference error. Workaround: ensure pinned data exists on the Webhook node or change expressions to use a unified source.  
  - If the model returns malformed JSON, the structured output parser will fail.  
  - API rate limits on OpenAI could cause 429 errors.

---

**OpenAI Chat Model**  
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3)  
- **Technical Role:** Provides the LLM backend (GPT-5-mini) for the Task Assignment chain.  
- **Configuration:**  
  - Model: `gpt-5-mini` (selected from list)  
  - Options: default (no temperature, max tokens, etc. overrides)  
  - Built-in Tools: none enabled  
- **Credential:** `OpenAi account (Eure)` — OpenAI API key credential.  
- **Key Expressions / Variables:** None.  
- **Input Connections:** None (sub-node, connected to Task Assignment via `ai_languageModel`).  
- **Output Connections:** → Task Assignment (ai_languageModel)  
- **Version Requirements:** Requires OpenAI API access with `gpt-5-mini` model availability.  
- **Edge Cases / Failure Types:**  
  - Invalid or expired API key → 401.  
  - Model not available in the API plan → 404/model-not-found.  
  - Usage quota exceeded → 429.

---

**Structured Output Parser**  
- **Type:** `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3)  
- **Technical Role:** Enforces that the LLM returns a JSON object matching the defined schema, ensuring the output contains exactly `id` (string) and `name` (string).  
- **Configuration:**  
  - Schema Type: manual  
  - Input Schema:
    ```json
    {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" }
      }
    }
    ```
- **Key Expressions / Variables:** None.  
- **Input Connections:** None (sub-node, connected to Task Assignment via `ai_outputParser`).  
- **Output Connections:** → Task Assignment (ai_outputParser)  
- **Version Requirements:** Requires LangChain output parser node support.  
- **Edge Cases / Failure Types:**  
  - If the model cannot produce valid structured output, the parser will retry (depending on LangChain retry settings) or fail.  
  - Additional properties beyond `id` and `name` will be silently dropped by the schema.

---

#### Block 1.4 – Issue Creation & Logging

**Overview:** Creates the new issue in Paperclip with the AI-selected assignee, then appends a tracking row to Google Sheets with issue metadata.

**Nodes Involved:** Create issue, Add issue to Sheet

---

**Create issue**  
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)  
- **Technical Role:** POSTs a new issue to the Paperclip API with the assigned agent ID.  
- **Configuration:**  
  - Method: POST  
  - URL: `https://paperclip.xxx.xxx/api/companies/{{ $('Get Company ID').item.json.companyId }}/issues`  
  - Authentication: Generic Credential Type → HTTP Bearer Auth  
  - Credential: `Paperclip - Acme Corp (Hetzner)`  
  - Body (JSON):
    ```json
    {
      "title": "Scrivi contenuto: {{ $('Webhook').item.json.title }}",
      "description": "{{ $('Webhook').item.json.issue }}",
      "status": "todo",
      "assigneeAgentId": "{{ $json.output.id }}"
    }
    ```
  - Specify Body: json  
- **Key Expressions / Variables:**  
  - `$('Get Company ID').item.json.companyId` — reuses the company ID fetched earlier.  
  - `$('Webhook').item.json.title` / `.issue` — same webhook reference caveat as Task Assignment.  
  - `$json.output.id` — the agent ID returned by the structured output parser via the Task Assignment node.  
- **Input Connections:** ← Task Assignment  
- **Output Connections:** → Add issue to Sheet  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - Same auth and URL placeholder risks as other Paperclip HTTP requests.  
  - If `$json.output.id` is undefined (parser failure), the `assigneeAgentId` will be empty, potentially creating an unassigned issue or causing a 400 from the API.  
  - The title is prefixed with `"Scrivi contenuto: "` (Italian for "Write content:") — this is hardcoded and may not be appropriate for all use cases.

---

**Add issue to Sheet**  
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)  
- **Technical Role:** Appends a new row to the tracking spreadsheet with issue metadata.  
- **Configuration:**  
  - Operation: append  
  - Document: `Paperclip issuess Schedule` (ID: `1MDJ-SAMiRWdfSpXMXOzH2qiipym-8qhX31ymPMdcz8k`)  
  - Sheet: `Foglio1` (gid=0)  
  - Columns mapped:  
    - DATE: `{{$now.format('dd/LL/yyyy HH:ii')}}` — current timestamp at insertion time  
    - TITLE: `={{ $('Webhook').item.json.title }}`  
    - ISSUE: `={{ $('Webhook').item.json.issue }}`  
    - ASSIGN: `={{ $('Task Assignment').item.json.output.name }}` — the agent name chosen by the AI  
    - ID: `={{ $json.id }}` — the Paperclip issue ID returned by the Create issue call  
    - COMPLETED: (left empty on append — populated later by Flow B)  
  - Matching Columns: none (append always adds a new row)  
- **Credential:** `Google Sheets account` — Google Sheets OAuth2 API.  
- **Key Expressions / Variables:** See column mappings above.  
- **Input Connections:** ← Create issue  
- **Output Connections:** None (terminal node for Flow A).  
- **Version Requirements:** Requires Google Sheets OAuth2 credential with write access to the target spreadsheet.  
- **Edge Cases / Failure Types:**  
  - 403 if the OAuth2 credential lacks access to the spreadsheet.  
  - 404 if the spreadsheet or sheet name has been deleted or renamed.  
  - Date format `dd/LL/yyyy HH:ii` uses Luxon tokens — ensure the n8n timezone setting is correct for expected output.

---

#### Block 2.1 – Scheduled Polling

**Overview:** A schedule trigger fires every 10 minutes, causing the workflow to read all rows from the Google Sheet where the COMPLETED column is empty, then iterate over them one by one.

**Nodes Involved:** Schedule Trigger, Get open issues, Loop Over Items

---

**Schedule Trigger**  
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.3)  
- **Technical Role:** Starts Flow B on a recurring 10-minute interval.  
- **Configuration:**  
  - Rule: Interval → every 10 minutes  
- **Key Expressions / Variables:** None.  
- **Input Connections:** None (entry node).  
- **Output Connections:** → Get open issues  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - If the workflow is deactivated, the schedule won't fire.  
  - Overlapping executions possible if a single run takes more than 10 minutes; consider enabling "Execute Once" or adjusting the interval.

---

**Get open issues**  
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)  
- **Technical Role:** Reads all rows from the tracking spreadsheet, filtering for those where the COMPLETED column is empty.  
- **Configuration:**  
  - Operation: read (default)  
  - Document: `Paperclip issuess Schedule` (same as above)  
  - Sheet: `Foglio1` (gid=0)  
  - Filter: COMPLETED column (lookup column specified, but no filter value explicitly set — relies on empty-cell filtering by n8n's Google Sheets node, which returns rows where the specified column is blank)  
  - Options: default  
- **Credential:** `Google Sheets account` — Google Sheets OAuth2 API.  
- **Key Expressions / Variables:** Each output item contains the row data plus a `row_number` property used later for updates.  
- **Input Connections:** ← Schedule Trigger  
- **Output Connections:** → Loop Over Items  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - Returns zero items if no open issues exist; the Loop Over Items node will immediately exit.  
  - If the COMPLETED column filter doesn't work as expected (e.g., the node returns all rows), the loop may process already-completed issues unnecessarily — this is harmless but inefficient.  
  - Read permission issues → 403.

---

**Loop Over Items**  
- **Type:** `n8n-nodes-base.splitInBatches` (v3)  
- **Technical Role:** Processes open issues one at a time; output 1 fires when all items are done (empty, no connection), output 2 fires for each item to process. After a cycle completes (either via the If false branch or after the Update issue to Sheet node), execution returns here to process the next item.  
- **Configuration:**  
  - Options: default (batch size = 1)  
- **Key Expressions / Variables:** The `item.json.row_number` property is passed through for later use in the Update node.  
- **Input Connections:** ← Get open issues (initial), ← If (false branch — not completed, loop back), ← Update issue to Sheet (after completion processing)  
- **Output Connections:**  
  - Output 1 (done): no connection (workflow ends for Flow B)  
  - Output 2 (has items): → Get issue  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - If an error occurs inside the loop for one item, the entire execution may halt unless error handling is added.  
  - The loop re-invokes itself via two feedback paths (If false and Update Sheet); ensure no infinite loop by verifying that the If false path correctly skips completed issues.

---

#### Block 2.2 – Per-Issue Status Check

**Overview:** For each open issue from the loop, queries the Paperclip API to get the current status, then branches based on whether the status equals "completed".

**Nodes Involved:** Get issue, If

---

**Get issue**  
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)  
- **Technical Role:** Fetches the current state of a single Paperclip issue by its ID.  
- **Configuration:**  
  - Method: GET (default)  
  - URL: `https://paperclip.xxx.xxx/api/issues/{{ $json.ID }}`  
  - Authentication: Generic Credential Type → HTTP Bearer Auth  
  - Credential: `Paperclip - Acme Corp (Hetzner)`  
- **Key Expressions / Variables:** `{{ $json.ID }}` — the issue ID read from the Google Sheet row.  
- **Input Connections:** ← Loop Over Items (output 2)  
- **Output Connections:** → If  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - 404 if the issue has been deleted from Paperclip since it was logged.  
  - Auth errors as with other Paperclip nodes.  
  - If the ID column in the Sheet is empty or invalid, the URL will be malformed.

---

**If**  
- **Type:** `n8n-nodes-base.if` (v2.3)  
- **Technical Role:** Conditional branch — checks whether the issue's `status` field equals `"completed"`.  
- **Configuration:**  
  - Combinator: AND  
  - Condition: `{{ $json.status }}` equals (string, strict type validation) `completed`  
  - Case Sensitive: true  
- **Key Expressions / Variables:** `$json.status` from the Get issue response.  
- **Input Connections:** ← Get issue  
- **Output Connections:**  
  - True (status = completed) → Send alert  
  - False (status ≠ completed) → Loop Over Items (back to loop, process next item)  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - If the Paperclip API returns a status value with different casing (e.g., "Completed"), the strict case-sensitive check will fail and the issue won't be marked as completed in the Sheet.  
  - If the `status` field is missing from the API response, the condition evaluates to false.

---

#### Block 2.3 – Notification & Sheet Update

**Overview:** When an issue is confirmed completed, sends a Gmail notification with issue details and writes the current timestamp into the COMPLETED column of the corresponding Sheet row.

**Nodes Involved:** Send alert, Update issue to Sheet

---

**Send alert**  
- **Type:** `n8n-nodes-base.gmail` (v2.2)  
- **Technical Role:** Sends a completion notification email via Gmail.  
- **Configuration:**  
  - To: `xxx@xxx.xxx` (placeholder — must be replaced with real recipient)  
  - Subject: `[Paperclip] {{ $json.title }} COMPLETED`  
  - Message:
    ```
    Hi,
    This issue is completed
    
    Title: {{ $json.title }}
    Description: {{ $json.description }}
    ```
  - Options: default  
- **Credential:** `Gmail account (n3w.it)` — Gmail OAuth2.  
- **Key Expressions / Variables:** `$json.title` and `$json.description` from the Get issue response.  
- **Input Connections:** ← If (true branch)  
- **Output Connections:** → Update issue to Sheet  
- **Version Requirements:** Requires Gmail OAuth2 credential with sending permissions.  
- **Edge Cases / Failure Types:**  
  - 403 if the Gmail credential lacks send permissions.  
  - 429 if Gmail rate limits are hit (unlikely at 10-min intervals).  
  - If the recipient email is invalid, Gmail may bounce.

---

**Update issue to Sheet**  
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)  
- **Technical Role:** Updates the COMPLETED column of the specific row in the Google Sheet with the current timestamp.  
- **Configuration:**  
  - Operation: update  
  - Document: `Paperclip issuess Schedule` (same spreadsheet)  
  - Sheet: `Foglio1` (gid=0)  
  - Matching Column: `row_number`  
  - Columns mapped:  
    - COMPLETED: `={{$now.format('dd/LL/yyyy HH:ii')}}` — current timestamp  
    - row_number: `={{ $('Loop Over Items').item.json.row_number }}` — identifies which row to update  
- **Credential:** `Google Sheets account` — Google Sheets OAuth2 API.  
- **Key Expressions / Variables:**  
  - `row_number` from the Loop Over Items context identifies the exact row.  
  - Date format same as Add issue to Sheet.  
- **Input Connections:** ← Send alert  
- **Output Connections:** → Loop Over Items (back to loop for next item)  
- **Version Requirements:** None.  
- **Edge Cases / Failure Types:**  
  - If `row_number` is not available (e.g., the Sheet schema changed), the update will fail or update the wrong row.  
  - Write permission issues → 403.  
  - Concurrent modifications to the Sheet could shift row numbers between read and update, causing mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | manualTrigger | Manual test entry point | — | Set data | STEP 1 - Receives an issue: Receives a task with title and issue via webhook or Set Data for testing |
| Webhook | webhook | HTTP POST endpoint for receiving task payloads | — | Get Company ID | STEP 1 - Receives an issue: Receives a task with title and issue via webhook or Set Data for testing |
| Set data | set | Hardcodes test title and issue for manual runs | When clicking 'Execute workflow' | Get Company ID | STEP 1 - Receives an issue: Receives a task with title and issue via webhook or Set Data for testing |
| Get Company ID | httpRequest | Fetches the authenticated user's company ID from Paperclip | Webhook / Set data | Get Paperclip Agents | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Get Paperclip Agents | httpRequest | Fetches all agents for the discovered company | Get Company ID | Normalize | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Normalize | code | Simplifies agent objects to id/name/title and wraps in a result object | Get Paperclip Agents | Task Assignment | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| OpenAI Chat Model | lmChatOpenAi | Provides GPT-5-mini LLM to the Task Assignment chain | — (sub-node) | Task Assignment (ai_languageModel) | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Structured Output Parser | outputParserStructured | Enforces structured JSON output (id + name) from the LLM | — (sub-node) | Task Assignment (ai_outputParser) | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Task Assignment | chainLlm | Uses GPT-5-mini to select the best agent for the task | Normalize, OpenAI Chat Model, Structured Output Parser | Create issue | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Create issue | httpRequest | Creates a new issue in Paperclip with the AI-chosen assignee | Task Assignment | Add issue to Sheet | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Add issue to Sheet | googleSheets | Appends a tracking row with issue metadata to Google Sheets | Create issue | — (terminal) | STEP 2 - Create an issue on Paperclip: Retrieves all available Paperclip agents for that company, Creates a new issue in Paperclip with the assigned agent, Logs the issue to a Google Sheet with metadata (date, ID, title, issue, assigned agent) |
| Schedule Trigger | scheduleTrigger | Fires the monitoring flow every 10 minutes | — | Get open issues | STEP 3 - Fetches all open issues: Fetches all open issues (where COMPLETED column is empty) from Google Sheets |
| Get open issues | googleSheets | Reads all rows from the Sheet where COMPLETED is empty | Schedule Trigger | Loop Over Items | STEP 3 - Fetches all open issues: Fetches all open issues (where COMPLETED column is empty) from Google Sheets |
| Loop Over Items | splitInBatches | Iterates over each open issue one at a time | Get open issues, If (false), Update issue to Sheet | Get issue (output 2) | STEP 3 - Fetches all open issues: Fetches all open issues (where COMPLETED column is empty) from Google Sheets |
| Get issue | httpRequest | Fetches the current status of a single Paperclip issue | Loop Over Items | If | STEP 4 - Checks the current status: Checks the current status of each issue in Paperclip API. If status is "completed", sends a Gmail alert and updates the COMPLETED column in Sheets |
| If | if | Branches on whether issue status equals "completed" | Get issue | Send alert (true), Loop Over Items (false) | STEP 4 - Checks the current status: Checks the current status of each issue in Paperclip API. If status is "completed", sends a Gmail alert and updates the COMPLETED column in Sheets |
| Send alert | gmail | Sends a completion notification email | If (true) | Update issue to Sheet | STEP 4 - Checks the current status: Checks the current status of each issue in Paperclip API. If status is "completed", sends a Gmail alert and updates the COMPLETED column in Sheets |
| Update issue to Sheet | googleSheets | Writes the completion timestamp into the Sheet row | Send alert | Loop Over Items | STEP 4 - Checks the current status: Checks the current status of each issue in Paperclip API. If status is "completed", sends a Gmail alert and updates the COMPLETED column in Sheets |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it `Paperclip Task Assigner & Completion Tracker`.

2. **Add the Manual Trigger node:**
   - Type: Manual Trigger  
   - No configuration needed.

3. **Add the Webhook node:**
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: leave auto-generated or set a memorable value  
   - No authentication (or add as needed)

4. **Add the Set node (named "Set data"):**
   - Type: Set (v3.4)  
   - Add two string assignments: `title` (value: a test title string) and `issue` (value: a test description string)  
   - Connect: Manual Trigger → Set data

5. **Connect both entry points to Get Company ID:**
   - Webhook → Get Company ID  
   - Set data → Get Company ID

6. **Add "Get Company ID" HTTP Request node:**
   - Type: HTTP Request (v4.4)  
   - Method: GET  
   - URL: `https://<your-paperclip-domain>/api/agents/me`  
   - Authentication: Generic Credential Type → HTTP Bearer Auth  
   - Create/Select credential: `Paperclip - <Your Org>` with a valid Bearer token

7. **Add "Get Paperclip Agents" HTTP Request node:**
   - Type: HTTP Request (v4.4)  
   - Method: GET  
   - URL: `https://<your-paperclip-domain>/api/companies/{{ $json.companyId }}/agents`  
   - Authentication: same Bearer Auth credential as step 6  
   - Connect: Get Company ID → Get Paperclip Agents

8. **Add the "Normalize" Code node:**
   - Type: Code (v2)  
   - Language: JavaScript  
   - Code:
     ```javascript
     const data = items.map(item => item.json);
     const result = data.map(el => ({
       id: el.id,
       name: el.name,
       title: el.title
     }));
     return [{ json: { result: result } }];
     ```
   - Connect: Get Paperclip Agents → Normalize

9. **Add the "OpenAI Chat Model" node:**
   - Type: OpenAI Chat Model (LangChain)  
   - Model: `gpt-5-mini`  
   - Create/Select credential: OpenAI API key

10. **Add the "Structured Output Parser" node:**
    - Type: Structured Output Parser (LangChain)  
    - Schema Type: manual  
    - Input Schema:
      ```json
      {
        "type": "object",
        "properties": {
          "id": { "type": "string" },
          "name": { "type": "string" }
        }
      }
      ```

11. **Add the "Task Assignment" LLM Chain node:**
    - Type: LLM Chain (LangChain, v1.9)  
    - Prompt Type: define  
    - Prompt Text:
      ```
      Task title: {{ $('Webhook').item.json.title }}
      Task description: {{ $('Webhook').item.json.issue }}
      
      Agents: {{JSON.stringify($json.result)}}
      ```
    - System Message: `Return only an array with a single agent, assigning it the most appropriate task for a Paperclip agent.`  
    - Has Output Parser: true  
    - Connect sub-nodes: OpenAI Chat Model → Task Assignment (ai_languageModel); Structured Output Parser → Task Assignment (ai_outputParser)  
    - Connect main flow: Normalize → Task Assignment

12. **Add "Create issue" HTTP Request node:**
    - Type: HTTP Request (v4.4)  
    - Method: POST  
    - URL: `https://<your-paperclip-domain>/api/companies/{{ $('Get Company ID').item.json.companyId }}/issues`  
    - Authentication: same Bearer Auth credential  
    - Specify Body: JSON  
    - JSON Body:
      ```json
      {
        "title": "Scrivi contenuto: {{ $('Webhook').item.json.title }}",
        "description": "{{ $('Webhook').item.json.issue }}",
        "status": "todo",
        "assigneeAgentId": "{{ $json.output.id }}"
      }
      ```
    - Connect: Task Assignment → Create issue

13. **Clone the Google Sheet:**
    - Open [the provided template spreadsheet](https://docs.google.com/spreadsheets/d/1MDJ-SAMiRWdfSpXMXOzH2qiipym-8qhX31ymPMdcz8k/edit?usp=sharing) and make a copy.  
    - Note the new spreadsheet ID and sheet name (default: `Foglio1`).  
    - Ensure the sheet has columns: DATE, TITLE, ISSUE, ASSIGN, ID, COMPLETED.

14. **Add "Add issue to Sheet" Google Sheets node:**
    - Type: Google Sheets (v4.7)  
    - Operation: append  
    - Document ID: select your cloned spreadsheet  
    - Sheet: `Foglio1`  
    - Column mapping (define below):  
      - DATE: `{{$now.format('dd/LL/yyyy HH:ii')}}`  
      - TITLE: `={{ $('Webhook').item.json.title }}`  
      - ISSUE: `={{ $('Webhook').item.json.issue }}`  
      - ASSIGN: `={{ $('Task Assignment').item.json.output.name }}`  
      - ID: `={{ $json.id }}`  
      - COMPLETED: (leave blank)  
    - Create/Select credential: Google Sheets OAuth2 API  
    - Connect: Create issue → Add issue to Sheet

15. **Add the Schedule Trigger node:**
    - Type: Schedule Trigger (v1.3)  
    - Interval: every 10 minutes

16. **Add "Get open issues" Google Sheets node:**
    - Type: Google Sheets (v4.7)  
    - Operation: read  
    - Document ID: same spreadsheet  
    - Sheet: `Foglio1`  
    - Filter: COMPLETED column (to return only rows where COMPLETED is empty)  
    - Same Google Sheets OAuth2 credential  
    - Connect: Schedule Trigger → Get open issues

17. **Add "Loop Over Items" Split in Batches node:**
    - Type: Split In Batches (v3)  
    - Default options (batch size 1)  
    - Connect: Get open issues → Loop Over Items (input)

18. **Add "Get issue" HTTP Request node:**
    - Type: HTTP Request (v4.4)  
    - Method: GET  
    - URL: `https://<your-paperclip-domain>/api/issues/{{ $json.ID }}`  
    - Authentication: same Bearer Auth credential  
    - Connect: Loop Over Items (output 2) → Get issue

19. **Add the "If" node:**
    - Type: IF (v2.3)  
    - Condition: `$json.status` equals (string, strict, case-sensitive) `completed`  
    - Connect: Get issue → If  
    - True output → Send alert  
    - False output → Loop Over Items (back to loop)

20. **Add "Send alert" Gmail node:**
    - Type: Gmail (v2.2)  
    - To: your notification email address  
    - Subject: `[Paperclip] {{ $json.title }} COMPLETED`  
    - Message:
      ```
      Hi,
      This issue is completed
      
      Title: {{ $json.title }}
      Description: {{ $json.description }}
      ```
    - Create/Select credential: Gmail OAuth2  
    - Connect: If (true) → Send alert

21. **Add "Update issue to Sheet" Google Sheets node:**
    - Type: Google Sheets (v4.7)  
    - Operation: update  
    - Document ID: same spreadsheet  
    - Sheet: `Foglio1`  
    - Matching Column: `row_number`  
    - Column mapping:  
      - COMPLETED: `={{$now.format('dd/LL/yyyy HH:ii')}}`  
      - row_number: `={{ $('Loop Over Items').item.json.row_number }}`  
    - Same Google Sheets OAuth2 credential  
    - Connect: Send alert → Update issue to Sheet; Update issue to Sheet → Loop Over Items (back to loop)

22. **Replace all placeholder values:**
    - Replace `paperclip.xxx.xxx` in all four HTTP Request nodes with your real Paperclip instance domain.  
    - Replace `xxx@xxx.xxx` in the Gmail node with the actual recipient.  
    - Replace placeholder `"xxx"` values in the Set node with meaningful test data.

23. **Verify credentials:**
    - `httpBearerAuth`: Paperclip API token — ensure the token has access to `/api/agents/me`, `/api/companies/{id}/agents`, `/api/issues`, and `/api/issues/{id}`.  
    - `openAiApi`: OpenAI API key with access to the `gpt-5-mini` model.  
    - `googleSheetsOAuth2Api`: Google account with read/write access to the cloned spreadsheet.  
    - `gmailOAuth2`: Google account with Gmail sending permissions.

24. **Test Flow A:**  
    - Use the Manual Trigger to execute the workflow with the Set node's test data.  
    - Verify that the issue is created in Paperclip and a new row appears in the Google Sheet.

25. **Test Flow B:**  
    - Manually mark a Paperclip issue as completed.  
    - Either wait for the schedule trigger or manually trigger the monitoring section by running the Get open issues node.  
    - Verify that the Gmail alert is sent and the COMPLETED column is updated.

26. **Activate the workflow** to enable the schedule trigger and webhook endpoint.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Clone the provided Google Sheet template before first run. The sheet must have columns: DATE, TITLE, ISSUE, ASSIGN, ID, COMPLETED. | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1MDJ-SAMiRWdfSpXMXOzH2qiipym-8qhX31ymPMdcz8k/edit?usp=sharing) |
| YouTube video tutorial for setting up a Paperclip instance and obtaining an API key (subtitles in English). | [YouTube Tutorial](https://www.youtube.com/watch?v=vdnyiF92qcY) |
| The creator's YouTube channel with n8n videos, Shorts, and free templates. | [YouTube Channel – n3witalia](https://youtube.com/@n3witalia) |
| The `$('Webhook')` expressions in Task Assignment and Create issue nodes reference the Webhook node specifically. When using the Manual Trigger path, these references will fail unless the Webhook node has pinned data or the expressions are adjusted to a unified source (e.g., `$('Set data')`). | General compatibility note |
| The date format `dd/LL/yyyy HH:ii` uses Luxon tokens (`LL` = padded month number, `ii` = minutes). Ensure the n8n instance timezone is configured correctly for the expected output. | Date formatting note |
| The hardcoded Italian prefix `"Scrivi contenuto: "` in the Create issue title may not suit all use cases. Modify as needed. | Internationalisation note |
| The If node uses strict case-sensitive comparison. Paperclip API status values must exactly match `"completed"` (lowercase) for the true branch to fire. | Case sensitivity warning |
| The Loop Over Items node processes one item at a time. If the number of open issues is very large, the 10-minute schedule interval may not be sufficient to process all items before the next trigger fires. | Scalability consideration |