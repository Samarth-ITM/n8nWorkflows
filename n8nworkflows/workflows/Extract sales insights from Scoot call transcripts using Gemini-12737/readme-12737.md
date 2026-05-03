Extract sales insights from Scoot call transcripts using Gemini

https://n8nworkflows.xyz/workflows/extract-sales-insights-from-scoot-call-transcripts-using-gemini-12737


# Extract sales insights from Scoot call transcripts using Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Extract sales insights from Scoot call transcripts using Gemini  
**Workflow name (in JSON):** Extract sales insights from call transcripts with Scoot and Gemini

**Purpose:**  
Automatically ingest a completed call transcription from **Scoot**, fetch the full transcript via Scoot API, use **Gemini** to extract structured sales insights (budget, competitors, objections, next steps, etc.), then:
- update a CRM record (currently a **mock** node), and
- email a formatted post-call summary to the sales rep via **Gmail**.

**Primary use cases:**
- Sales operations automation (post-call notes, pipeline hygiene)
- Standardized sales insight extraction across calls
- Auto-filling CRM custom fields + rep notifications

### 1.1 Trigger & Test Entry Points
Two entry points exist:
- Production: **Webhook** receives Scoot event when transcription completes
- Testing: **Manual Trigger** injects a sample transcript without Scoot

### 1.2 Transcript Fetching & Normalization
Fetch transcript from Scoot API, then merge webhook metadata and API response into one normalized record.

### 1.3 Readiness Gate + Retry Loop
If transcript status is not `COMPLETE`, the workflow increments a retry counter, waits 1 hour, and refetches (up to 6 retries). If still incomplete, logs abandonment.

### 1.4 AI Extraction (Gemini Agent)
Gemini Agent receives the transcript + context and must return **only JSON** with specific fields.

### 1.5 Validation, Formatting, CRM Update, Email Notification
AI output is parsed/validated into safe types, formatted into:
- `crm_update` payload (CRM fields), and
- `email_summary` (human-readable summary),
then sent to the CRM mock update and emailed to the rep.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers (Production + Test)
**Overview:** Starts the workflow either from Scoot’s webhook (production) or from a manual test path that loads a sample transcript.

**Nodes involved:**
- Receive Scoot webhook
- Test with sample data
- Load sample transcript

#### Node: Receive Scoot webhook
- **Type / role:** `Webhook` — entry point for Scoot notifications.
- **Key configuration (interpreted):**
  - Method: `POST`
  - Path: `/scoot-transcription`
  - Webhook ID: `scoot-transcription-webhook` (internal n8n identifier)
- **Inputs / outputs:**
  - No input (trigger).
  - Output → **Fetch transcript from Scoot**
- **Edge cases / failures:**
  - Scoot misconfigured webhook URL or method mismatch.
  - Payload missing `socialTranscriptionId` (required by later HTTP request expression).
  - If webhook authentication/signing is expected by Scoot, it is not implemented here (consider verifying signature in a Code node).
- **Version notes:** Webhook node `typeVersion: 2`.

#### Node: Test with sample data
- **Type / role:** `Manual Trigger` — alternative entry for development/testing.
- **Key configuration:** none.
- **Outputs:** → **Load sample transcript**
- **Edge cases:** none; only runs manually.

#### Node: Load sample transcript
- **Type / role:** `Code` — injects mock call metadata + transcript for testing.
- **Key configuration:**
  - Produces a single JSON object with fields like:
    - `socialTranscriptionId`, `status: "COMPLETE"`, `company_name`, `rep_email`, `transcript`, `is_mock: true`, etc.
- **Outputs:** → **Combine call metadata**
- **Edge cases / failures:**
  - Large transcript may exceed later model/context limits (depends on Gemini model & n8n settings).
- **Version notes:** Code node `typeVersion: 2`.

---

### Block 2.2 — Transcript Fetch + Data Normalization
**Overview:** Retrieves the full transcript from Scoot’s API (production path) and merges it with webhook metadata into a normalized schema used downstream.

**Nodes involved:**
- Fetch transcript from Scoot
- Combine call metadata

#### Node: Fetch transcript from Scoot
- **Type / role:** `HTTP Request` — calls Scoot API to fetch transcription details.
- **Key configuration (interpreted):**
  - URL (expression):  
    `https://api.scoot.app/api/v1/transcription?socialTranscriptionId={{ $json.socialTranscriptionId }}`
  - Authentication: `httpHeaderAuth` via “genericCredentialType”
  - Credential: “Header Auth account” (must contain Scoot API key as header)
- **Inputs / outputs:**
  - Input: from **Receive Scoot webhook** or **Prepare retry request**
  - Output → **Combine call metadata**
- **Edge cases / failures:**
  - Missing/invalid `socialTranscriptionId` → malformed URL / Scoot 4xx.
  - Wrong API key/header name → 401/403.
  - Scoot API downtime / timeouts.
  - Response shape changes (later code expects fields like `status`, `user.emailAddress`, `startDateTime`).
- **Version notes:** HTTP Request node `typeVersion: 4.2`.

#### Node: Combine call metadata
- **Type / role:** `Code` — merges webhook payload and Scoot API response; provides defaults.
- **Key configuration (logic highlights):**
  - If `input.is_mock` is true → passes input through unchanged.
  - Otherwise:
    - `webhookData = $('Receive Scoot webhook').item?.json || {}`
    - `scootData = input`
    - Returns merged object with derived fields:
      - `transcript` fallback if missing
      - `company_name` derived from `scootData.name` by stripping prefixes
      - `rep_email` from `scootData.user.emailAddress` fallback to `user@example.com`
      - `rep_name` from `scootData.user.name`
      - `call_date` from `scootData.startDateTime` (YYYY-MM-DD)
- **Inputs / outputs:**
  - Input: **Fetch transcript from Scoot** OR **Load sample transcript**
  - Output → **Transcript ready?**
- **Edge cases / failures:**
  - The expression `$('Receive Scoot webhook')...` can be fragile if execution path started from manual trigger (but code guards with `is_mock`; for non-mock manual data without webhook, webhookData becomes `{}`).
  - If Scoot returns unexpected types (e.g., `startDateTime` null), fallback date logic covers most cases.
- **Version notes:** Code node `typeVersion: 2`.

---

### Block 2.3 — Transcript Status Gate + Retry Handling
**Overview:** Ensures transcript is complete before AI extraction. If still processing, waits and retries fetching up to 6 times, otherwise logs abandonment.

**Nodes involved:**
- Transcript ready?
- Count retry attempts
- Retries remaining?
- Wait 1 hour
- Prepare retry request
- Log failure

#### Node: Transcript ready?
- **Type / role:** `IF` — status check gate.
- **Key configuration:**
  - Condition: `{{$json.status}} equals "COMPLETE"`
- **Inputs / outputs:**
  - Input: **Combine call metadata**
  - True output → **Extract data with Gemini**
  - False output → **Count retry attempts**
- **Edge cases / failures:**
  - Status values other than `COMPLETE`/`PROCESSING` (e.g. `FAILED`) will go to retry branch; later logic only allows retries when status is `PROCESSING`, so it will fail fast to **Log failure**.
- **Version notes:** IF node `typeVersion: 2`.

#### Node: Count retry attempts
- **Type / role:** `Code` — increments retry counter and decides if retry allowed.
- **Key configuration (logic):**
  - `currentRetryCount = data.retry_count || 0`
  - `maxRetries = 6`
  - `retry_count = currentRetryCount + 1`
  - `can_retry = currentRetryCount < maxRetries && data.status === 'PROCESSING'`
- **Outputs:** → **Retries remaining?**
- **Edge cases / failures:**
  - Off-by-one nuance: with `maxRetries=6`, `can_retry` uses `currentRetryCount < maxRetries` before increment. This yields up to 6 waits/refetches after the initial incomplete result, depending on initial `retry_count` presence.
- **Version notes:** Code node `typeVersion: 2`.

#### Node: Retries remaining?
- **Type / role:** `IF` — routes either to wait+retry or failure.
- **Key configuration:** `{{$json.can_retry}} equals true`
- **Outputs:**
  - True → **Wait 1 hour**
  - False → **Log failure**
- **Edge cases:** If status is `FAILED`, `can_retry` false → immediate **Log failure**.

#### Node: Wait 1 hour
- **Type / role:** `Wait` — delays execution before retrying Scoot fetch.
- **Key configuration:** `amount: 1`, `unit: hours`
- **Outputs:** → **Prepare retry request**
- **Edge cases / failures:**
  - Long waits require n8n to be configured for waiting executions (depends on your n8n mode and queue/worker setup).
- **Version notes:** Wait node `typeVersion: 1.1`.

#### Node: Prepare retry request
- **Type / role:** `Code` — reshapes data for refetch.
- **Key configuration:**
  - Returns `{ socialTranscriptionId, retry_count, original_webhook_data }`
  - `original_webhook_data` preserved if present, else stores current object.
- **Outputs:** → **Fetch transcript from Scoot** (loop)
- **Edge cases:**
  - `original_webhook_data` is not used elsewhere in this workflow; it’s kept for potential auditing/debugging.

#### Node: Log failure
- **Type / role:** `Code` — emits a final “ABANDONED” object when retries exhausted or transcription failed.
- **Key configuration:**
  - `failureReason`:
    - if `status === 'FAILED'` → “Transcription failed in Scoot”
    - else → “Max retries exceeded - still processing after X hours”
- **Outputs:** none (terminal)
- **Edge cases:** Consider also notifying someone or writing to logs/DB; currently this just ends the workflow.

---

### Block 2.4 — AI Extraction with Gemini (LangChain Agent)
**Overview:** Sends transcript + context to a Gemini-backed agent with strict instructions to output JSON only.

**Nodes involved:**
- Extract data with Gemini
- Gemini 1.5 Flash

#### Node: Extract data with Gemini
- **Type / role:** `LangChain Agent` — orchestrates prompt + model call.
- **Key configuration (interpreted):**
  - Prompt text includes:
    - Company, contact, date
    - Full transcript inlined from `{{$json.transcript}}`
  - System message defines:
    - Exact JSON schema to return
    - Rules: “Only extract explicitly stated”, use nulls, short strings, “Return ONLY the JSON object”
  - Prompt mode: `define`
- **Model connection:** Uses the `ai_languageModel` input connected from **Gemini 1.5 Flash**.
- **Outputs:** → **Validate extracted JSON**
- **Edge cases / failures:**
  - Model returns non-JSON (despite instruction) → handled by validation node.
  - Transcript too long → model context limit errors or truncation; may reduce extraction quality.
  - Latency/timeouts from Google API.
- **Version notes:** Agent node `typeVersion: 1.7`.

#### Node: Gemini 1.5 Flash
- **Type / role:** `Google Gemini Chat Model` — language model provider for the agent.
- **Key configuration:**
  - Temperature: `0.3` (more deterministic extraction)
- **Credentials:** Google Gemini/PaLM API credential (“Google Gemini(PaLM) Api account”).
- **Outputs:** Feeds agent via `ai_languageModel` connection.
- **Edge cases / failures:**
  - Invalid API key, billing/quota issues, disabled API → authentication errors.
  - Model availability changes by region/account.
- **Version notes:** `typeVersion: 1`.

---

### Block 2.5 — Validation, Formatting, CRM Update, Email
**Overview:** Parses and validates the AI response into stable types, formats CRM fields and a readable email summary, updates CRM (mock), and emails the rep.

**Nodes involved:**
- Validate extracted JSON
- Format for CRM update
- Update CRM (mock)
- Email summary to rep

#### Node: Validate extracted JSON
- **Type / role:** `Code` — robust parsing + type validation + merge with call metadata.
- **Key configuration (logic highlights):**
  - Pulls previous call context from: `$('Combine call metadata').item.json`
  - Reads AI response from: `input.text || input.content || input.message?.content || ''`
  - Strips Markdown code fences (```json … ``` or ``` … ```)
  - `JSON.parse()`; on error produces a safe default object with `extraction_error`
  - Enforces types and allowed values:
    - booleans, numbers, strings, arrays
    - `call_sentiment` restricted to `positive|neutral|negative`
  - Outputs normalized record including metadata + `extraction_timestamp`
- **Outputs:** → **Format for CRM update**
- **Edge cases / failures:**
  - If the agent output is nested differently than expected and none of `text/content/message.content` exists, parsing will fail → default neutral output is used.
  - If Gemini returns valid JSON but wrong field types, validation coerces to defaults (e.g., arrays become `[]`).
- **Version notes:** Code node `typeVersion: 2`.

#### Node: Format for CRM update
- **Type / role:** `Code` — builds CRM payload and formatted email summary.
- **Key configuration:**
  - Creates `crm_update` object mapping extracted insights into CRM-like fields, e.g.:
    - `budget_confirmed__c`, `competitors__c`, `next_steps__c`, etc.
  - Builds `email_summary` as a plain-text formatted block (sections: call details, budget, competitors, objections, timeline, decision maker, next steps, pain points, buying signals, recommendation).
- **Outputs:** → **Update CRM (mock)**
- **Edge cases:**
  - `budget_amount.toLocaleString()` will throw if `budget_amount` is not a number; upstream validation ensures number or null, and the code checks `data.budget_amount ? ... : ...` (note: `0` would be treated as falsy; if budget can be 0, consider `data.budget_amount !== null`).
- **Version notes:** Code node `typeVersion: 2`.

#### Node: Update CRM (mock)
- **Type / role:** `Code` — placeholder for a real CRM integration.
- **Key configuration:**
  - Returns `crm_response` with:
    - `success: true`
    - `crm_provider: 'MOCK - Replace with real CRM'`
    - `fields_updated` list of non-null/non-empty fields
- **Outputs:** → **Email summary to rep**
- **Edge cases:** No real persistence; must replace with HubSpot/Salesforce/Pipedrive node(s).

#### Node: Email summary to rep
- **Type / role:** `Gmail` — sends the post-call summary to the rep.
- **Key configuration:**
  - To: `{{$json.rep_email}}`
  - Subject: `Call summary: {{ $json.company_name }} ({{ $json.call_sentiment }})`
  - Message body: `{{$json.email_summary}}`
- **Credentials:** Gmail OAuth2 (“Daniel Gmail account”)
- **Edge cases / failures:**
  - Invalid OAuth token / revoked consent.
  - Gmail API quota limits.
  - `rep_email` missing or invalid email format → send failure/bounce.
- **Version notes:** Gmail node `typeVersion: 2.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / explanation |  |  | ## How it works  This workflow automatically extracts structured sales data from call transcripts:  1. **Trigger** → Scoot sends webhook when transcription completes 2. **Fetch** → Retrieves full transcript from Scoot API 3. **Extract** → Gemini AI pulls out budget, competitors, objections, next steps 4. **Update** → Writes extracted fields to your CRM 5. **Notify** → Emails formatted summary to the sales rep  ## Setup steps  1. **Scoot API** → Create Header Auth credential with your API key 2. **Gemini** → Add Google AI API key from aistudio.google.com 3. **Gmail** → Connect OAuth2 for sending notifications 4. **Webhook** → Copy URL to Scoot dashboard → Webhooks 5. **CRM** → Replace mock node with HubSpot/Salesforce/Pipedrive 6. **Test** → Click "Test with sample data" to verify setup |
| Sticky Note1 | Sticky Note | Documentation / explanation |  |  | **Trigger options** Webhook receives Scoot notifications. Manual trigger uses sample data for testing without real calls. |
| Sticky Note2 | Sticky Note | Documentation / explanation |  |  | **Data preparation** Fetches transcript from Scoot, checks if processing is complete, handles retries for incomplete transcripts. |
| Sticky Note3 | Sticky Note | Documentation / explanation |  |  | **AI extraction** AI Agent with Gemini model extracts: budget, competitors, objections, timeline, decision maker, next steps, pain points, buying signals. |
| Sticky Note4 | Sticky Note | Documentation / explanation |  |  | **Output** Replace mock CRM node with your integration. Gmail sends formatted summary to the rep who made the call. |
| Sticky Note5 | Sticky Note | Documentation / explanation |  |  | **Retry logic** If transcript still processing: waits 1 hour, retries up to 6 times. Adjust wait time in "Wait 1 hour" node. |
| Receive Scoot webhook | Webhook | Production trigger (Scoot event intake) | — | Fetch transcript from Scoot | **Trigger options** Webhook receives Scoot notifications. Manual trigger uses sample data for testing without real calls. |
| Test with sample data | Manual Trigger | Test trigger | — | Load sample transcript | **Trigger options** Webhook receives Scoot notifications. Manual trigger uses sample data for testing without real calls. |
| Load sample transcript | Code | Generate mock transcript payload | Test with sample data | Combine call metadata | **Trigger options** Webhook receives Scoot notifications. Manual trigger uses sample data for testing without real calls. |
| Fetch transcript from Scoot | HTTP Request | Fetch transcript details from Scoot API | Receive Scoot webhook; Prepare retry request | Combine call metadata | **Data preparation** Fetches transcript from Scoot, checks if processing is complete, handles retries for incomplete transcripts. |
| Combine call metadata | Code | Merge webhook + Scoot response; normalize fields | Fetch transcript from Scoot; Load sample transcript | Transcript ready? | **Data preparation** Fetches transcript from Scoot, checks if processing is complete, handles retries for incomplete transcripts. |
| Transcript ready? | IF | Gate: proceed only if status COMPLETE | Combine call metadata | Extract data with Gemini; Count retry attempts | **Data preparation** Fetches transcript from Scoot, checks if processing is complete, handles retries for incomplete transcripts. |
| Count retry attempts | Code | Increment retry count; compute can_retry | Transcript ready? (false) | Retries remaining? | **Retry logic** If transcript still processing: waits 1 hour, retries up to 6 times. Adjust wait time in "Wait 1 hour" node. |
| Retries remaining? | IF | Decide whether to retry or abandon | Count retry attempts | Wait 1 hour; Log failure | **Retry logic** If transcript still processing: waits 1 hour, retries up to 6 times. Adjust wait time in "Wait 1 hour" node. |
| Wait 1 hour | Wait | Delay before re-fetching transcript | Retries remaining? (true) | Prepare retry request | **Retry logic** If transcript still processing: waits 1 hour, retries up to 6 times. Adjust wait time in "Wait 1 hour" node. |
| Prepare retry request | Code | Reshape data for refetch loop | Wait 1 hour | Fetch transcript from Scoot | **Retry logic** If transcript still processing: waits 1 hour, retries up to 6 times. Adjust wait time in "Wait 1 hour" node. |
| Log failure | Code | Terminal logging for abandoned/failed transcription | Retries remaining? (false) | — | **Retry logic** If transcript still processing: waits 1 hour, retries up to 6 times. Adjust wait time in "Wait 1 hour" node. |
| Extract data with Gemini | LangChain Agent | AI extraction into structured JSON | Transcript ready? (true) | Validate extracted JSON | **AI extraction** AI Agent with Gemini model extracts: budget, competitors, objections, timeline, decision maker, next steps, pain points, buying signals. |
| Gemini 1.5 Flash | Google Gemini Chat Model | LLM provider for agent | — | Extract data with Gemini (ai_languageModel) | **AI extraction** AI Agent with Gemini model extracts: budget, competitors, objections, timeline, decision maker, next steps, pain points, buying signals. |
| Validate extracted JSON | Code | Parse/clean AI output; enforce schema/types | Extract data with Gemini | Format for CRM update | **AI extraction** AI Agent with Gemini model extracts: budget, competitors, objections, timeline, decision maker, next steps, pain points, buying signals. |
| Format for CRM update | Code | Build CRM payload + formatted email summary | Validate extracted JSON | Update CRM (mock) | **Output** Replace mock CRM node with your integration. Gmail sends formatted summary to the rep who made the call. |
| Update CRM (mock) | Code | Placeholder CRM update | Format for CRM update | Email summary to rep | **Output** Replace mock CRM node with your integration. Gmail sends formatted summary to the rep who made the call. |
| Email summary to rep | Gmail | Send summary email to sales rep | Update CRM (mock) | — | **Output** Replace mock CRM node with your integration. Gmail sends formatted summary to the rep who made the call. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Extract sales insights from call transcripts with Scoot and Gemini** (or your preferred name).
   - (Optional) Add tags: `sales-automation`, `call-transcription`, `scoot-integration`.

2. **Add the production trigger**
   - Add node: **Webhook**
   - Name: **Receive Scoot webhook**
   - Method: `POST`
   - Path: `scoot-transcription`
   - Save the node to generate the webhook URL.
   - In Scoot dashboard → Webhooks, register the **Production** webhook URL.

3. **Add the test trigger path**
   - Add node: **Manual Trigger**
   - Name: **Test with sample data**
   - Add node: **Code**
   - Name: **Load sample transcript**
   - Paste code that returns a single JSON item containing at least:
     - `socialTranscriptionId`, `status`, `company_name`, `rep_email`, `rep_name`, `call_date`, `transcript`, and `is_mock: true`
   - Connect: **Test with sample data → Load sample transcript**

4. **Add Scoot transcript fetch**
   - Add node: **HTTP Request**
   - Name: **Fetch transcript from Scoot**
   - Method: GET (default)
   - URL (expression):  
     `https://api.scoot.app/api/v1/transcription?socialTranscriptionId={{ $json.socialTranscriptionId }}`
   - Authentication: **Header Auth** (HTTP Header Auth credential)
     - Create credential: **HTTP Header Auth**
     - Add Scoot API key header as required by Scoot (commonly `Authorization: Bearer <key>` or `x-api-key: <key>` depending on Scoot).
   - Connect:
     - **Receive Scoot webhook → Fetch transcript from Scoot**
     - (later) **Prepare retry request → Fetch transcript from Scoot**

5. **Add metadata normalization**
   - Add node: **Code**
   - Name: **Combine call metadata**
   - Implement logic:
     - If `is_mock` true, return input as-is.
     - Else merge webhook JSON + Scoot response JSON.
     - Derive `company_name`, `rep_email`, `rep_name`, `call_date` with sensible fallbacks.
   - Connect:
     - **Fetch transcript from Scoot → Combine call metadata**
     - **Load sample transcript → Combine call metadata**

6. **Add readiness check**
   - Add node: **IF**
   - Name: **Transcript ready?**
   - Condition (String):
     - Left: `={{ $json.status }}`
     - Operation: `equals`
     - Right: `COMPLETE`
   - Connect: **Combine call metadata → Transcript ready?**

7. **Add retry loop (false branch)**
   - Add node: **Code** named **Count retry attempts**
     - Increment `retry_count`
     - Set `max_retries = 6`
     - Compute `can_retry` (true only if status is `PROCESSING` and retries remain)
   - Add node: **IF** named **Retries remaining?**
     - Condition (Boolean): `={{ $json.can_retry }}` equals `true`
   - Add node: **Wait** named **Wait 1 hour**
     - Unit: `hours`
     - Amount: `1`
   - Add node: **Code** named **Prepare retry request**
     - Output `{ socialTranscriptionId, retry_count, original_webhook_data }`
   - Add node: **Code** named **Log failure**
     - Output status `ABANDONED` + reason (FAILED vs retries exceeded)
   - Connect (from **Transcript ready?** false output):
     - **Transcript ready? (false) → Count retry attempts → Retries remaining?**
     - **Retries remaining? (true) → Wait 1 hour → Prepare retry request → Fetch transcript from Scoot**
     - **Retries remaining? (false) → Log failure**

8. **Add Gemini model node**
   - Add node: **Google Gemini Chat Model** (n8n LangChain integration)
   - Name: **Gemini 1.5 Flash**
   - Temperature: `0.3`
   - Create credential: **Google Gemini(PaLM) API**
     - Use API key from: `https://aistudio.google.com`
     - Ensure the correct API/plan is enabled for your Google project/account.

9. **Add AI Agent extraction (true branch)**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: **Extract data with Gemini**
   - Provide prompt text that includes company/contact/date and the transcript.
   - Set a strict **system message** that:
     - defines the JSON schema (budget, competitors, objections, etc.)
     - instructs: “Return ONLY the JSON object”
   - Connect:
     - **Transcript ready? (true) → Extract data with Gemini**
     - Connect **Gemini 1.5 Flash** to the agent’s **Language Model** input (`ai_languageModel`).

10. **Add AI output validation**
    - Add node: **Code**
    - Name: **Validate extracted JSON**
    - Implement:
      - Read model output (text/content/message.content)
      - Strip ```json fences
      - `JSON.parse()` with try/catch
      - On failure output defaults + `extraction_error`
      - Type-check each expected field
      - Merge with prior metadata (from **Combine call metadata**)
    - Connect: **Extract data with Gemini → Validate extracted JSON**

11. **Format CRM payload and email**
    - Add node: **Code**
    - Name: **Format for CRM update**
    - Implement:
      - `crm_update` object mapping to your CRM field API names (custom fields like `budget_confirmed__c`, etc.)
      - `email_summary` plaintext block
    - Connect: **Validate extracted JSON → Format for CRM update**

12. **CRM update (replace mock in production)**
    - For parity with this workflow, add node: **Code** named **Update CRM (mock)** that returns a fake success response.
    - In production, replace this node with your CRM node(s), e.g.:
      - **HubSpot**: Update a deal by `deal_id`, set properties.
      - **Salesforce**: Update Opportunity by Id, map custom fields.
      - **Pipedrive**: Update deal fields.
    - Connect: **Format for CRM update → Update CRM (mock/real)**

13. **Email notification**
    - Add node: **Gmail**
    - Name: **Email summary to rep**
    - Configure:
      - To: `={{ $json.rep_email }}`
      - Subject: `Call summary: {{ $json.company_name }} ({{ $json.call_sentiment }})`
      - Message: `={{ $json.email_summary }}`
    - Create credential: **Gmail OAuth2**
      - Connect the Google account authorized to send emails.
    - Connect: **Update CRM (mock/real) → Email summary to rep**

14. **Activate and test**
    - Use **Test with sample data** to validate parsing/formatting/email.
    - Then enable workflow and send a real Scoot webhook event.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Gemini API key setup | https://aistudio.google.com |
| Scoot API auth requirement | Create an n8n **HTTP Header Auth** credential containing Scoot API key header expected by Scoot. |
| CRM integration | Replace “Update CRM (mock)” with HubSpot/Salesforce/Pipedrive nodes and map `crm_update` fields to your CRM properties. |
| Retry timing | “Wait 1 hour” node controls delay; retry logic caps at 6 attempts. |
| Testing approach | Use “Test with sample data” + “Load sample transcript” to validate end-to-end without Scoot. |