Qualify webhook leads with GPT-4o-mini and create Asana tasks automatically

https://n8nworkflows.xyz/workflows/qualify-webhook-leads-with-gpt-4o-mini-and-create-asana-tasks-automatically-12480


# Qualify webhook leads with GPT-4o-mini and create Asana tasks automatically

## 1. Workflow Overview

**Purpose:**  
This workflow receives inbound lead data via an HTTP webhook, enriches the lead using RocketReach, computes a data-quality confidence score, uses **GPT-4o-mini** to score/qualify the lead, and then:
- **Creates an Asana task** for qualified leads and **notifies Slack**
- **Rejects** low-confidence or low-score leads and **notifies Slack**
- **Sends a Gmail alert** if any workflow error occurs (via Error Trigger)

**Primary use cases:**
- Automating lead qualification from forms / landing pages / outbound tools that can POST to a webhook
- Routing only high-quality leads into Asana while tracking disqualified leads

### 1.1 Data Ingestion & Normalization
Webhook receives raw lead fields and maps them into consistent internal names.

### 1.2 Enrichment (RocketReach) + Data Quality Scoring
Calls RocketReach profile lookup using LinkedIn URL, then derives a **confidence score** from the best available email’s “grade”.

### 1.3 AI Scoring (GPT-4o-mini) + Threshold Filtering
Sends enriched profile to OpenAI, expects strict JSON output `{score, priority, reason}`, then filters on score.

### 1.4 Task Creation + Notifications
Responds to the webhook, then creates an Asana task and posts Slack messages for qualified or disqualified leads.

### 1.5 Error Handling
Any unhandled failure triggers an email alert via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Data Ingestion & Normalization
**Overview:** Receives the lead payload via webhook and maps incoming field names into a normalized schema used by downstream nodes.  
**Nodes involved:** `Webhook Trigger`, `Normalize Lead Data`

#### Node: Webhook Trigger
- **Type / role:** `Webhook` trigger; entry point for inbound HTTP requests
- **Key configuration:**
  - Method: **POST**
  - Path: `lead-enrichment`
  - Response mode: **Respond via “Respond to Webhook” node** (`responseNode`)
- **Input/Output:**
  - **Output →** `Normalize Lead Data`
- **Edge cases / failures:**
  - Caller doesn’t POST JSON or uses different field names → downstream expressions become `undefined`
  - If the workflow doesn’t reach a Respond node (e.g., a fatal error before response), caller may see a timeout

#### Node: Normalize Lead Data
- **Type / role:** `Set` node; standardizes incoming data
- **Configuration choices:**
  - Creates fields:
    - `firstName` = `{{$json.body.first_name}}`
    - `lastName` = `{{$json.body.last_name}}`
    - `company` = `{{$json.body.company}}`
    - `jobTitle` = `{{$json.body.job_title}}`
    - `domain` = `{{$json.body.company_domain}}`
    - `linkedinUrl` = `{{$json.body.linkedin_url}}`
- **Input/Output:**
  - **Input ←** `Webhook Trigger`
  - **Output →** `RocketReach Lookup Profile`
- **Edge cases:**
  - If webhook sender uses different keys (e.g., `firstName` instead of `first_name`), normalized fields will be empty and RocketReach lookup will likely fail/return low data.

---

### Block 2 — Lead Enrichment & Quality Check
**Overview:** Enriches the lead using RocketReach by LinkedIn URL, then calculates a confidence score from the best available email grade and filters out low-confidence leads.  
**Nodes involved:** `RocketReach Lookup Profile`, `Calculate Email Confidence`, `Quality Filter`, `Respond - Low Confidence`

#### Node: RocketReach Lookup Profile
- **Type / role:** `HTTP Request`; calls RocketReach to enrich lead profile
- **Configuration choices:**
  - URL expression (query-based):
    - `https://api.rocketreach.co/v2/api/lookupProfile?api_key=YOUR_API_KEY&linkedin_url={{encodeURIComponent($json.linkedinUrl)}}`
  - Timeout: **45s**
  - Retries: `maxTries=3`, `retryOnFail=true`
  - **onError:** `continueErrorOutput` (workflow continues and provides an error output item)
- **Input/Output:**
  - **Input ←** `Normalize Lead Data`
  - **Output →** `Calculate Email Confidence`
- **Edge cases / failures:**
  - **Hardcoded API key placeholder** (`YOUR_API_KEY`) must be replaced; otherwise you’ll get 401/403 or invalid request
  - Missing/invalid `linkedinUrl` → RocketReach may return error or empty result
  - Rate limiting or RocketReach downtime
  - Because `continueErrorOutput` is enabled, downstream nodes may receive an unexpected structure (error body instead of profile JSON)

#### Node: Calculate Email Confidence
- **Type / role:** `Code` node; derives a normalized email selection and numeric confidence
- **Logic (interpreted):**
  - Maps RocketReach email grade to confidence:
    - A → 0.9, B → 0.7, C → 0.5, D → 0.3
  - Picks `selectedEmail`:
    1) first `emails[].type === "professional"`, else  
    2) first `emails[].type === "personal"`, else `null`
  - Outputs:
    - `selected_email`, `email_type`, `email_grade`, `confidence`
    - plus all original RocketReach fields
- **Input/Output:**
  - **Input ←** `RocketReach Lookup Profile`
  - **Output →** `Quality Filter`
- **Edge cases:**
  - If RocketReach returns no `emails` array, confidence defaults effectively to grade D (0.3)
  - If RocketReach returns an error JSON structure, `data.emails` may not exist; code handles it by defaulting to `[]`

#### Node: Quality Filter
- **Type / role:** `IF`; gates leads by data quality
- **Condition:**
  - `{{$json.confidence ?? 0}} >= 0.6`
- **Input/Output:**
  - **True →** `AI Lead Scoring`
  - **False →** `Respond - Low Confidence`
- **Edge cases:**
  - If `confidence` is missing/not numeric, it becomes `0` via nullish coalescing and fails the filter

#### Node: Respond - Low Confidence
- **Type / role:** `Respond to Webhook`; returns final response for rejected leads
- **Response body:**
  - `{ success: false, message: "Lead did not meet quality threshold", reason: "Confidence score below 0.6" }`
- **Input/Output:**
  - **Input ←** `Quality Filter` (false branch)
  - No downstream connections
- **Edge cases:**
  - This ends the HTTP request; callers will get a clean rejection response.

---

### Block 3 — AI Lead Scoring & Filtering
**Overview:** Sends enriched lead data to GPT-4o-mini and forces JSON-only output, then extracts score/priority/reason and filters out low-score leads.  
**Nodes involved:** `AI Lead Scoring`, `Extract AI Score Fields`, `Score Filter`, `Respond - Low Score`

#### Node: AI Lead Scoring
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi`; chat completion for scoring
- **Model:** `gpt-4o-mini`
- **Options:** Temperature `0.3` (more deterministic)
- **Prompt details (key variables referenced):**
  - Uses RocketReach-like fields such as:
    - `{{$json.name}}`, `{{$json.current_title}}`, `{{$json.current_employer}}`
    - `{{$json.emails?.[0]?.email ?? 'N/A'}}`, `{{$json.phones?.[0]?.number ?? 'N/A'}}`
    - `{{$json.confidence}}`, `{{$json.location || 'unknown'}}`
    - `{{$json.current_employer_id}}`, `{{$json.current_employer_size ?? 'Unknown'}}`
  - Requires output: **ONLY valid JSON** with fields `score`, `priority`, `reason`
- **jsonOutput:** enabled (expects JSON)
- **Credentials:** OpenAI credential `OpenAi account 3`
- **Input/Output:**
  - **Input ←** `Quality Filter` (true branch)
  - **Output →** `Extract AI Score Fields`
- **Edge cases / failures:**
  - If upstream data doesn’t contain the referenced fields (e.g., RocketReach returned different schema or error), the prompt may degrade and/or model may output unexpected JSON
  - Occasional malformed JSON output can still happen; extraction node may fail if structure differs
  - OpenAI auth issues, quota/rate limits, timeouts

#### Node: Extract AI Score Fields
- **Type / role:** `Set`; normalizes AI output into top-level fields used by filters and responses
- **Configuration choices:**
  - `score` = `{{$json.message.content.score}}`
  - `priority` = `{{$json.message.content.priority}}`
  - `reason` = `{{$json.message.content.reason}}`
- **Input/Output:**
  - **Input ←** `AI Lead Scoring`
  - **Output →** `Score Filter`
- **Edge cases:**
  - If the OpenAI node returns a different response shape (e.g., `content` is a string, not parsed JSON), these expressions may become undefined and cause filter issues.

#### Node: Score Filter
- **Type / role:** `IF`; gates leads by AI score
- **Condition:**
  - `{{$json.score}} >= 6`
- **Input/Output:**
  - **True →** `Respond to Webhook`
  - **False →** `Respond - Low Score`
- **Edge cases:**
  - If `score` is missing/not numeric, the condition fails (or may error depending on strict validation).

#### Node: Respond - Low Score
- **Type / role:** `Respond to Webhook`; returns rejection for low AI score
- **Response body:**
  - `{ success: false, message: "Lead did not qualify", reason: "AI score below threshold", score: $json.aiScore }`
- **Input/Output:**
  - **Input ←** `Score Filter` (false branch)
  - **Output →** `Notify Slack - Disqualified`
- **Important issue (data mismatch):**
  - The response uses `{{$json.aiScore}}`, but the workflow created `score` (not `aiScore`). This will likely return `null/undefined` for `score` in the response unless another field exists.
- **Edge cases:**
  - Even though it responds to the webhook, it continues to Slack notification (which is fine in n8n, but note the webhook caller already got a response).

---

### Block 4 — Task Creation & Notifications
**Overview:** For qualified leads, responds to the webhook, creates an Asana task, and notifies Slack. For disqualified leads, posts a Slack message.  
**Nodes involved:** `Respond to Webhook`, `Create Asana Task`, `Notify Slack - Qualified`, `Notify Slack - Disqualified`

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook`; returns success response
- **Response body:**
  - `{ success: true, message: "Lead processed successfully", score, priority, taskCreated: true }`
- **Input/Output:**
  - **Input ←** `Score Filter` (true branch)
  - **Output →** `Create Asana Task`
- **Behavior note:**
  - The HTTP caller receives a success response before the Asana/Slack steps complete. If Asana fails afterward, the caller will still have received success.

#### Node: Create Asana Task
- **Type / role:** `Asana`; creates a task
- **Configuration choices:**
  - Task name: `Lead Added`
  - Workspace: `1205967598025927`
  - Assignee: `1212557659831213`
  - Notes: `{{$json.reason}}` (AI-provided explanation)
  - Auth: OAuth2 (`Asana account 2`)
- **Input/Output:**
  - **Input ←** `Respond to Webhook`
  - **Output →** `Notify Slack - Qualified`
- **Edge cases / failures:**
  - Invalid workspace/assignee IDs, insufficient permissions, OAuth token expiry
  - Missing `reason` field if AI extraction failed

#### Node: Notify Slack - Qualified
- **Type / role:** `Slack`; posts message to a channel
- **Configuration choices:**
  - Channel: `general-information` (ID `C09GNB90TED`)
  - Message includes:
    - Score from `$('Extract AI Score Fields').item.json.score`
    - Asana task `notes` and `assignee.name` from the Asana node output (`$json`)
- **Input/Output:**
  - **Input ←** `Create Asana Task`
  - No downstream connections
- **Edge cases:**
  - Slack credential/token issues, channel access issues
  - If Asana output doesn’t include `assignee.name` (depends on Asana API fields returned), message may show blanks

#### Node: Notify Slack - Disqualified
- **Type / role:** `Slack`; posts disqualification message
- **Configuration choices:**
  - Channel: `general-information` (ID `C09GNB90TED`)
  - Text: `Lead Disqualified !`
- **Input/Output:**
  - **Input ←** `Respond - Low Score`
  - No downstream connections
- **Edge cases:**
  - Slack auth/channel permission issues

---

### Block 5 — Error Handling
**Overview:** If the workflow errors (outside nodes that “continue on fail”), an error trigger sends an alert email with node name, error, and timestamp.  
**Nodes involved:** `Workflow Error Handler`, `Send a message1`

#### Node: Workflow Error Handler
- **Type / role:** `Error Trigger`; runs when the workflow execution fails
- **Input/Output:**
  - **Output →** `Send a message1`
- **Edge cases:**
  - Only triggers on executions that are considered failed; nodes with `onError: continueErrorOutput` may prevent a “failure” state.

#### Node: Send a message1
- **Type / role:** `Gmail`; sends an email alert
- **Configuration choices:**
  - Message includes:
    - Error node name: `{{$json.node.name}}`
    - Error message: `{{$json.error.message}}`
    - Timestamp: `{{$now.toISO()}}`
  - Email type: `text`
  - Credential: Gmail OAuth2 (`jyothi`)
- **Input/Output:**
  - **Input ←** `Workflow Error Handler`
- **Edge cases:**
  - Gmail OAuth token expiry, missing permissions, sending limits

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / workflow description |  |  | ## 🎯 AI-Powered Lead Enrichment & Qualification … (setup steps, thresholds 0.6 and 6) |
| Sticky Note1 | Sticky Note | Documentation: ingestion/normalization |  |  | ## 📥 Data Ingestion & Normalization … |
| Sticky Note2 | Sticky Note | Documentation: enrichment/quality |  |  | ## 🔍 Lead Enrichment & Quality Check … |
| Sticky Note3 | Sticky Note | Documentation: AI scoring/filtering |  |  | ## 🤖 AI Lead Scoring & Filtering … |
| Sticky Note4 | Sticky Note | Documentation: task + notifications |  |  | ## ✅ Task Creation & Notifications … |
| Sticky Note5 | Sticky Note | Documentation: credentials/security |  |  | ## 🔐 Credentials & Security … Replace hardcoded RocketReach API key with a credential. |
| Webhook Trigger | Webhook | Receives lead POST and delegates response to response node | (Entry) | Normalize Lead Data | ## 📥 Data Ingestion & Normalization … |
| Normalize Lead Data | Set | Maps webhook body fields to normalized lead fields | Webhook Trigger | RocketReach Lookup Profile | ## 📥 Data Ingestion & Normalization … |
| RocketReach Lookup Profile | HTTP Request | Enriches lead via RocketReach lookupProfile endpoint | Normalize Lead Data | Calculate Email Confidence | ## 🔍 Lead Enrichment & Quality Check … |
| Calculate Email Confidence | Code | Selects best email, maps grade to confidence | RocketReach Lookup Profile | Quality Filter | ## 🔍 Lead Enrichment & Quality Check … |
| Quality Filter | IF | Filters out leads with confidence < 0.6 | Calculate Email Confidence | AI Lead Scoring (true), Respond - Low Confidence (false) | ## 🔍 Lead Enrichment & Quality Check … |
| Respond - Low Confidence | Respond to Webhook | Returns rejection response for low confidence | Quality Filter (false) |  | ## 🔍 Lead Enrichment & Quality Check … |
| AI Lead Scoring | OpenAI (LangChain) | Scores lead with GPT-4o-mini; JSON output | Quality Filter (true) | Extract AI Score Fields | ## 🤖 AI Lead Scoring & Filtering … |
| Extract AI Score Fields | Set | Extracts score/priority/reason from AI response | AI Lead Scoring | Score Filter | ## 🤖 AI Lead Scoring & Filtering … |
| Score Filter | IF | Filters out leads with score < 6 | Extract AI Score Fields | Respond to Webhook (true), Respond - Low Score (false) | ## 🤖 AI Lead Scoring & Filtering … |
| Respond to Webhook | Respond to Webhook | Returns success response to caller | Score Filter (true) | Create Asana Task | ## ✅ Task Creation & Notifications … |
| Create Asana Task | Asana | Creates task for qualified lead | Respond to Webhook | Notify Slack - Qualified | ## ✅ Task Creation & Notifications … |
| Notify Slack - Qualified | Slack | Notifies team of created Asana task | Create Asana Task |  | ## ✅ Task Creation & Notifications … |
| Respond - Low Score | Respond to Webhook | Returns rejection response for low AI score | Score Filter (false) | Notify Slack - Disqualified | ## ✅ Task Creation & Notifications … |
| Notify Slack - Disqualified | Slack | Notifies team that lead was disqualified | Respond - Low Score |  | ## ✅ Task Creation & Notifications … |
| Workflow Error Handler | Error Trigger | Starts alert flow when execution fails | (Failure event) | Send a message1 | ## 🚨 Error Handling … |
| Send a message1 | Gmail | Sends email alert with error details | Workflow Error Handler |  | ## 🚨 Error Handling … |
| Sticky Note7 | Sticky Note | Documentation: error handling |  |  | ## 🚨 Error Handling … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n. Name it:  
   `Convert inbound lead data from Webhook to AI-qualified Asana tasks`

2. **Add Webhook Trigger (Webhook node)**
   - Method: `POST`
   - Path: `lead-enrichment`
   - Response mode: **Using “Respond to Webhook” node**
   - Copy the **test/production webhook URL** for your lead capture system.

3. **Add “Normalize Lead Data” (Set node)**
   - Add fields:
     - `firstName` → `{{$json.body.first_name}}`
     - `lastName` → `{{$json.body.last_name}}`
     - `company` → `{{$json.body.company}}`
     - `jobTitle` → `{{$json.body.job_title}}`
     - `domain` → `{{$json.body.company_domain}}`
     - `linkedinUrl` → `{{$json.body.linkedin_url}}`
   - Connect: **Webhook Trigger → Normalize Lead Data**

4. **Add “RocketReach Lookup Profile” (HTTP Request node)**
   - Method: GET (implicit by URL-only usage)
   - URL (expression):
     - `{{ 'https://api.rocketreach.co/v2/api/lookupProfile?api_key=YOUR_API_KEY&linkedin_url=' + encodeURIComponent($json.linkedinUrl) }}`
   - Options:
     - Timeout: `45000`
     - Retry on fail: enabled; set max tries to `3`
   - Error handling: **Continue on fail / continue error output**
   - Connect: **Normalize Lead Data → RocketReach Lookup Profile**
   - **Credential/security recommendation:** move the API key into an n8n credential or environment variable and reference it in the URL expression instead of hardcoding.

5. **Add “Calculate Email Confidence” (Code node)**
   - Paste logic that:
     - selects professional/personal email
     - maps grade A/B/C/D to confidence 0.9/0.7/0.5/0.3
     - outputs `selected_email`, `email_type`, `email_grade`, `confidence`
   - Connect: **RocketReach Lookup Profile → Calculate Email Confidence**

6. **Add “Quality Filter” (IF node)**
   - Condition (Number):
     - Left: `{{$json.confidence ?? 0}}`
     - Operation: `>=`
     - Right: `0.6`
   - Connect: **Calculate Email Confidence → Quality Filter**

7. **Add “Respond - Low Confidence” (Respond to Webhook node)**
   - Respond with: JSON
   - Body expression:
     - `{ "success": false, "message": "Lead did not meet quality threshold", "reason": "Confidence score below 0.6" }`
   - Connect: **Quality Filter (false) → Respond - Low Confidence**

8. **Add “AI Lead Scoring” (OpenAI / LangChain OpenAI node)**
   - Model: `gpt-4o-mini`
   - Temperature: `0.3`
   - Enable “JSON output” mode
   - Prompt: include lead fields and require strict JSON:
     - `{ "score": <1-10>, "priority": "<High|Medium|Low>", "reason": "<...>" }`
   - Configure **OpenAI API credential** in n8n.
   - Connect: **Quality Filter (true) → AI Lead Scoring**

9. **Add “Extract AI Score Fields” (Set node)**
   - `score` → `{{$json.message.content.score}}`
   - `priority` → `{{$json.message.content.priority}}`
   - `reason` → `{{$json.message.content.reason}}`
   - Connect: **AI Lead Scoring → Extract AI Score Fields**

10. **Add “Score Filter” (IF node)**
   - Condition (Number):
     - Left: `{{$json.score}}`
     - Operation: `>=`
     - Right: `6`
   - Connect: **Extract AI Score Fields → Score Filter**

11. **Add “Respond - Low Score” (Respond to Webhook node)**
   - Respond with: JSON
   - Body expression (recommended fix):
     - `{ "success": false, "message": "Lead did not qualify", "reason": "AI score below threshold", "score": $json.score }`
   - Connect: **Score Filter (false) → Respond - Low Score**

12. **Add “Notify Slack - Disqualified” (Slack node)**
   - Post message to channel (e.g. `general-information`)
   - Text: `Lead Disqualified !`
   - Configure **Slack API credential/token**
   - Connect: **Respond - Low Score → Notify Slack - Disqualified**

13. **Add “Respond to Webhook” (Respond to Webhook node)**
   - Respond with: JSON
   - Body expression:
     - `{ "success": true, "message": "Lead processed successfully", "score": $json.score, "priority": $json.priority, "taskCreated": true }`
   - Connect: **Score Filter (true) → Respond to Webhook**

14. **Add “Create Asana Task” (Asana node)**
   - Auth: OAuth2; configure **Asana OAuth2** credential
   - Workspace: select your workspace (ID in source workflow: `1205967598025927`)
   - Task name: `Lead Added`
   - Notes: `{{$json.reason}}`
   - Assignee: choose user (ID in source workflow: `1212557659831213`)
   - Connect: **Respond to Webhook → Create Asana Task**

15. **Add “Notify Slack - Qualified” (Slack node)**
   - Channel: e.g. `general-information`
   - Message can reference:
     - Score from the extracted node: `{{ $('Extract AI Score Fields').item.json.score }}`
     - Asana output fields like `{{$json.notes}}`, `{{$json.assignee.name}}` (if present)
   - Connect: **Create Asana Task → Notify Slack - Qualified**

16. **Add error handling**
   - Add `Error Trigger` node named “Workflow Error Handler”
   - Add `Gmail` node “Send a message1”
     - Message body includes: `{{$json.node.name}}`, `{{$json.error.message}}`, `{{$now.toISO()}}`
     - Configure **Gmail OAuth2** credential
   - Connect: **Workflow Error Handler → Send a message1**

17. **Add sticky notes (optional)**
   - Add notes documenting: ingestion, enrichment, AI scoring, task/notifications, credentials warning, and error handling.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace the hardcoded RocketReach API key in the HTTP node with a credential. Never expose API keys in shared workflows. | Security note (from workflow sticky note) |
| Default thresholds: confidence **0.6** and AI score **6**; adjust based on your quality standards. | Operational tuning (from workflow sticky note) |
| Expected webhook sample fields: `first_name`, `last_name`, `company`, `job_title`, `company_domain`, `linkedin_url`. | Integration contract (from workflow sticky note) |
| Disqualified response currently references `$json.aiScore` but the workflow stores `$json.score`. Use `$json.score` to avoid null output. | Consistency/bug note (implementation detail) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.