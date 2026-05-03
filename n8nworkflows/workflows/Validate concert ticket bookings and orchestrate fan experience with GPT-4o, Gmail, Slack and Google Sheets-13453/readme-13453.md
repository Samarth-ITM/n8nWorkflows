Validate concert ticket bookings and orchestrate fan experience with GPT-4o, Gmail, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/validate-concert-ticket-bookings-and-orchestrate-fan-experience-with-gpt-4o--gmail--slack-and-google-sheets-13453


# Validate concert ticket bookings and orchestrate fan experience with GPT-4o, Gmail, Slack and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI-powered concert ticket validation and fan experience orchestration  
**Title (provided):** Validate concert ticket bookings and orchestrate fan experience with GPT-4o, Gmail, Slack and Google Sheets

**Purpose:**  
Automate end-to-end validation of concert ticket booking requests (fraud, payment, inventory, policy compliance) using an AI validation agent, then orchestrate downstream fan experience actions (ticketing updates, confirmations, escalation, audit logging) using a second AI agent. It reduces manual review by routing only high-risk cases to operations.

**Primary use cases:** Ticketing platforms, event operators, venue ops teams handling duplicate/bot purchases, payment authorization failures, overbooking, and standardized refund/escalation behavior.

### 1.1 Booking Intake & Configuration
Receives booking requests via webhook and loads configurable thresholds/endpoints.

### 1.2 Inventory Fetch & Validation Input Normalization
Calls an inventory API and packages booking + inventory + policy constraints into a single payload.

### 1.3 AI Ticket Validation (Risk Classification)
GPT-4o agent returns structured validation output (risk level/score, fraud indicators, eligibility, etc.).

### 1.4 Risk Routing (Auto-handle vs Human Review)
Switch routes low/medium to orchestration, high/critical (or explicitly flagged) to Slack escalation.

### 1.5 Fan Experience Orchestration (Downstream Actions)
GPT-4o agent produces structured action plan + ticketing system update object.

### 1.6 Execution (Ticketing Update + Email) & Audit Logging
Updates ticketing system, sends Gmail confirmation, merges all branches, and upserts an audit row in Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Booking Intake & Configuration
**Overview:** Accepts incoming booking payloads and defines runtime configuration (API endpoints, venue constraints, thresholds) used across the workflow.  
**Nodes involved:** Ticket Booking Webhook, Workflow Configuration

#### Node: Ticket Booking Webhook
- **Type / role:** `Webhook` — entry point receiving booking requests.
- **Key configuration:**
  - **Path:** `concert-ticket-booking`
  - **Method:** `POST`
  - **Response mode:** `lastNode` (webhook response returns output of the last executed node in the run).
- **Inputs/Outputs:**
  - **Output →** Workflow Configuration
- **Key data expected:**
  - Uses expressions later referencing `$('Ticket Booking Webhook').first().json.body.customerEmail`, `ticketQuantity`, `totalAmount`, `bookingId`.
- **Edge cases / failures:**
  - Missing or malformed `body` fields can break downstream expressions (e.g., null customerEmail).
  - With `responseMode=lastNode`, failures anywhere may produce webhook error responses; consider adding an error workflow or explicit response node if needed.
- **Version notes:** Webhook `typeVersion 2.1`.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central place for configurable constants and endpoints.
- **Key configuration (interpreted):**
  - Defines:
    - `inventoryApiUrl` (placeholder)
    - `ticketingApiUrl` (placeholder)
    - `venueCapacity = 5000`
    - `maxTicketsPerCustomer = 8`
    - `fraudScoreThreshold = 75`
    - `highRiskThreshold = 80`
  - **Include other fields:** enabled (keeps webhook payload available).
- **Inputs/Outputs:**
  - **Input ←** Ticket Booking Webhook
  - **Output →** Fetch Inventory Data
- **Edge cases / failures:**
  - Placeholder URLs must be replaced; otherwise HTTP Request nodes will fail.
  - If you rely on `fraudScoreThreshold/highRiskThreshold`, note they are not directly referenced later (risk routing uses the AI’s `riskLevel` / `requiresHumanReview`).
- **Version notes:** Set `typeVersion 3.4`.

---

### Block 2 — Inventory Fetch & Validation Input Normalization
**Overview:** Fetches inventory context from an external API and constructs a normalized object to feed the validation agent.  
**Nodes involved:** Fetch Inventory Data, Prepare Validation Input

#### Node: Fetch Inventory Data
- **Type / role:** `HTTP Request` — retrieves inventory data for cross-checks.
- **Key configuration:**
  - **URL:** from config: `$('Workflow Configuration').first().json.inventoryApiUrl`
  - Sends `Content-Type: application/json`
  - (Method not specified → defaults to GET in n8n HTTP Request)
- **Inputs/Outputs:**
  - **Input ←** Workflow Configuration
  - **Output →** Prepare Validation Input
- **Edge cases / failures:**
  - Auth not configured (node sends headers but no auth) — inventory API may require API key/OAuth.
  - Non-JSON or unexpected inventory response shape may degrade validation quality (agent still receives it, but may reason poorly).
  - Timeouts/5xx responses should be handled (currently no retry/fallback path).
- **Version notes:** HTTP Request `typeVersion 4.3`.

#### Node: Prepare Validation Input
- **Type / role:** `Set` — creates structured payload for AI validation.
- **Key configuration (fields created):**
  - `bookingData` = entire webhook `body`
  - `inventoryData` = HTTP response JSON from inventory node
  - `venueCapacity` and `maxTicketsPerCustomer` from config
  - **Include other fields:** enabled
- **Inputs/Outputs:**
  - **Input ←** Fetch Inventory Data
  - **Output →** Ticket Validation Agent
- **Edge cases / failures:**
  - If webhook `body` is absent or inventory call fails and returns HTML/string, `inventoryData` may not be an object.
- **Version notes:** Set `typeVersion 3.4`.

---

### Block 3 — AI Ticket Validation (Risk Classification)
**Overview:** Runs GPT-4o as a “Ticket Validation Agent” to produce a structured risk and eligibility assessment (including fraud indicators and review requirements).  
**Nodes involved:** OpenAI Model - Validation, Validation Output Parser, Ticket Validation Agent

#### Node: OpenAI Model - Validation
- **Type / role:** `lmChatOpenAi` (LangChain) — language model provider for the agent.
- **Key configuration:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.1` (low randomness; more deterministic classifications)
- **Inputs/Outputs:**
  - Connected to **Ticket Validation Agent** via `ai_languageModel`.
- **Credentials:** OpenAI API credential required.
- **Edge cases / failures:**
  - Invalid/expired API key, quota exceeded, regional restrictions.
  - Model name availability depends on your OpenAI account and n8n node version.
- **Version notes:** `typeVersion 1.3`.

#### Node: Validation Output Parser
- **Type / role:** `outputParserStructured` — enforces structured JSON output from the agent.
- **Key configuration:**
  - Provides a **schema example** containing fields like:
    - `validationStatus`, `riskLevel`, `riskScore`, `requiresHumanReview`, `fraudIndicators`, etc.
- **Inputs/Outputs:**
  - Connected to **Ticket Validation Agent** via `ai_outputParser`.
- **Edge cases / failures:**
  - If the model produces non-conforming JSON, parsing can fail and stop the workflow.
  - The schema is an example; strictness depends on node behavior/version—test with adversarial inputs.
- **Version notes:** `typeVersion 1.3`.

#### Node: Ticket Validation Agent
- **Type / role:** `agent` (LangChain) — orchestrates prompt + model + structured output parsing.
- **Key configuration:**
  - **Input text:** `={{ $json }}` (the entire normalized object from “Prepare Validation Input”)
  - **System message:** detailed policy/validation checklist + risk classification bands.
  - **Has output parser:** enabled (uses “Validation Output Parser”)
- **Inputs/Outputs:**
  - **Main input ←** Prepare Validation Input
  - **Main output →** Route by Risk Level
  - **AI connections:**
    - `ai_languageModel` ← OpenAI Model - Validation
    - `ai_outputParser` ← Validation Output Parser
- **Output shape (important for downstream):**
  - Downstream references assume `Ticket Validation Agent` outputs an object under `json.output` (e.g., `$json.output.riskLevel`).
- **Edge cases / failures:**
  - Large payloads (inventory + booking) risk token limits; trim inventory data if necessary.
  - If `$json` contains circular/very large objects, agent may fail or become slow.
- **Version notes:** `typeVersion 3.1`.

---

### Block 4 — Risk Routing (Auto-handle vs Human Review)
**Overview:** Uses the AI risk classification to route to either orchestration (low/medium) or Slack escalation (high/critical or human review required).  
**Nodes involved:** Route by Risk Level, Alert Operations Team

#### Node: Route by Risk Level
- **Type / role:** `Switch` — branching based on conditions.
- **Key configuration:**
  - Evaluates:
    - **Low/Medium Risk output:** `$json.output.riskLevel` equals `LOW` or `MEDIUM`
    - **High Risk - Human Review output:** riskLevel equals `HIGH` or `CRITICAL` OR `$json.output.requiresHumanReview` is true
  - Fallback output renamed to **Unclassified**
- **Inputs/Outputs:**
  - **Input ←** Ticket Validation Agent
  - **Output 1 (“Low/Medium Risk”) →** Fan Experience Orchestration Agent
  - **Output 2 (“High Risk - Human Review”) →** Alert Operations Team
  - **Fallback (“Unclassified”) →** (not connected; items would stop unless connected)
- **Edge cases / failures:**
  - If `riskLevel` missing or not one of expected strings, item goes to **Unclassified** (currently dropped).
  - Boolean condition uses `leftValue` expression for `requiresHumanReview`; if missing, evaluates false → may misroute.
- **Version notes:** `typeVersion 3.4`.

#### Node: Alert Operations Team
- **Type / role:** `Slack` — sends escalation to an operations channel.
- **Key configuration:**
  - OAuth2 auth
  - Channel ID is a placeholder.
  - Message includes booking details from the webhook and risk details from the validation agent.
  - Fraud indicators rendering: `join('\n• ')` with fallback “None”.
- **Inputs/Outputs:**
  - **Input ←** Route by Risk Level (High Risk branch)
  - **Output →** Merge All Paths (branch index 1)
- **Edge cases / failures:**
  - Slack OAuth token scopes missing (chat:write), invalid channel ID, app not in channel.
  - If `fraudIndicators` is not an array, `.join()` may error; the expression uses optional chaining but not type coercion.
- **Version notes:** Slack `typeVersion 2.4`.

---

### Block 5 — Fan Experience Orchestration (Downstream Actions)
**Overview:** For low/medium risk bookings, a second GPT-4o agent generates a structured plan of actions and a ticketing system update payload (confirmation, waitlist, refunds, SLA checks, audit summary).  
**Nodes involved:** OpenAI Model - Orchestration, Orchestration Output Parser, Fan Experience Orchestration Agent

#### Node: OpenAI Model - Orchestration
- **Type / role:** `lmChatOpenAi` — model provider for orchestration agent.
- **Key configuration:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.2` (slightly more flexible action planning)
- **Inputs/Outputs:**
  - Connected to **Fan Experience Orchestration Agent** via `ai_languageModel`.
- **Credentials:** OpenAI API credential required.
- **Edge cases / failures:** same as validation model node (quota/auth/model availability).
- **Version notes:** `typeVersion 1.3`.

#### Node: Orchestration Output Parser
- **Type / role:** `outputParserStructured` — structured output enforcement for orchestration.
- **Key configuration:**
  - Schema example includes:
    - `actions[]` (e.g., SEND_CONFIRMATION)
    - `ticketingSystemUpdate` object (bookingId/status/seats/paymentStatus)
    - flags: `waitlistActivation`, `refundRequired`, `escalationRequired`, `slaCompliance`
    - `auditLogSummary`
- **Inputs/Outputs:**
  - Connected to **Fan Experience Orchestration Agent** via `ai_outputParser`.
- **Edge cases / failures:** non-conforming JSON from model causes parser failure.
- **Version notes:** `typeVersion 1.3`.

#### Node: Fan Experience Orchestration Agent
- **Type / role:** `agent` — decides downstream actions and produces structured results.
- **Key configuration:**
  - **Input text:** `={{ $json }}` (this receives the validation agent’s output item as-is from Switch)
  - **System message:** policies for capacity, resale transfers, refund windows, overbooking compensation, SLA thresholds, audit trail generation.
  - **Has output parser:** enabled
- **Inputs/Outputs:**
  - **Main input ←** Route by Risk Level (Low/Medium branch)
  - **Main output →** Update Ticketing System
  - **AI connections:**
    - `ai_languageModel` ← OpenAI Model - Orchestration
    - `ai_outputParser` ← Orchestration Output Parser
- **Edge cases / failures:**
  - The orchestration agent is only executed for low/medium risk; high risk path never produces `ticketingSystemUpdate`, so downstream nodes must not depend on it (they don’t; high risk goes straight to merge).
  - If the orchestration output lacks `ticketingSystemUpdate`, the HTTP request node will send null/invalid body.
- **Version notes:** `typeVersion 3.1`.

---

### Block 6 — Execution (Ticketing Update + Email) & Audit Logging
**Overview:** Applies ticketing system updates, sends a confirmation email, merges both branches (auto vs escalation), and logs an audit record to Google Sheets (append or update by bookingId).  
**Nodes involved:** Update Ticketing System, Send Confirmation Email, Merge All Paths, Log to Audit Trail

#### Node: Update Ticketing System
- **Type / role:** `HTTP Request` — posts orchestration update to ticketing backend.
- **Key configuration:**
  - **URL:** `$('Workflow Configuration').first().json.ticketingApiUrl`
  - **Method:** POST
  - **Body:** JSON = `$json.output.ticketingSystemUpdate`
  - Sets `Content-Type: application/json`
- **Inputs/Outputs:**
  - **Input ←** Fan Experience Orchestration Agent
  - **Output →** Send Confirmation Email
- **Edge cases / failures:**
  - Auth not configured (may be required).
  - If `ticketingSystemUpdate` is missing/empty, request may fail or corrupt records.
  - Consider idempotency: repeated webhook calls could double-confirm unless ticketing API is idempotent by bookingId.
- **Version notes:** HTTP Request `typeVersion 4.3`.

#### Node: Send Confirmation Email
- **Type / role:** `Gmail` — sends booking confirmation email.
- **Key configuration:**
  - **To:** prefers orchestration action recipient:
    - `Fan Experience Orchestration Agent` → `output.actions.find(actionType==='SEND_CONFIRMATION').details.recipientEmail`
    - fallback: webhook `body.customerEmail`
  - **Subject:** from orchestration action subject or default `Concert Ticket Confirmation`
  - **Message:** HTML template including bookingId/status/seats from orchestration `ticketingSystemUpdate`
- **Inputs/Outputs:**
  - **Input ←** Update Ticketing System
  - **Output →** Merge All Paths (branch index 0)
- **Edge cases / failures:**
  - If the orchestration agent did not include SEND_CONFIRMATION action but you still want an email, fallback recipient works, but subject may default; body still references `ticketingSystemUpdate` fields and could render “undefined”.
  - Gmail OAuth scopes or “From” restrictions can block sending.
- **Version notes:** Gmail `typeVersion 2.2`.

#### Node: Merge All Paths
- **Type / role:** `Merge` — recombines low/medium (email path) and high risk (slack path).
- **Key configuration:**
  - **Mode:** `chooseBranch` (takes data from one branch; in practice each execution follows one route)
- **Inputs/Outputs:**
  - **Input 0 ←** Send Confirmation Email
  - **Input 1 ←** Alert Operations Team
  - **Output →** Log to Audit Trail
- **Edge cases / failures:**
  - Because execution chooses a branch, ensure the chosen branch carries all fields needed for audit logging via cross-node references (this workflow uses cross-node references heavily, so it’s mostly safe).
- **Version notes:** Merge `typeVersion 3.2`.

#### Node: Log to Audit Trail
- **Type / role:** `Google Sheets` — writes an audit record (append or update).
- **Key configuration:**
  - **Operation:** `appendOrUpdate`
  - **Matching column:** `bookingId`
  - **Document ID / Sheet name:** placeholders
  - **Columns written (expressions):**
    - bookingId: orchestration bookingId fallback to webhook bookingId
    - riskLevel, riskScore, validationStatus, requiresHumanReview: from validation agent
    - customerEmail: from webhook
    - timestamp: `$now.toISO()`
    - auditSummary: orchestration `auditLogSummary` fallback “Processed”
- **Inputs/Outputs:**
  - **Input ←** Merge All Paths
  - **Output:** end of workflow (webhook responds with this node’s output because `responseMode=lastNode`)
- **Edge cases / failures:**
  - If bookingId is missing, `appendOrUpdate` matching may behave unexpectedly (may append duplicates or fail matching).
  - Permissions: sheet access must allow edit; correct spreadsheet tab name required.
  - Rate limits for Sheets API on high-volume events.
- **Version notes:** Google Sheets `typeVersion 4.7`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Ticket Booking Webhook | Webhook | Entry point: receive booking request | — | Workflow Configuration | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| Workflow Configuration | Set | Store endpoints/thresholds/constants | Ticket Booking Webhook | Fetch Inventory Data | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| Fetch Inventory Data | HTTP Request | Retrieve inventory context | Workflow Configuration | Prepare Validation Input | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| Prepare Validation Input | Set | Normalize booking+inventory payload | Fetch Inventory Data | Ticket Validation Agent | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| OpenAI Model - Validation | OpenAI (LangChain Chat) | LLM backend for validation agent | — (AI connection) | Ticket Validation Agent (ai_languageModel) | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| Validation Output Parser | Structured Output Parser | Enforce JSON structure for validation output | — (AI connection) | Ticket Validation Agent (ai_outputParser) | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| Ticket Validation Agent | LangChain Agent | Produces risk/eligibility assessment | Prepare Validation Input | Route by Risk Level | ## Ingest & Ticket Validation; **What** – AI verifies payment, fraud signals, tier rules. **Why** – Prevent invalid or high-risk transactions. |
| Route by Risk Level | Switch | Branch low/medium vs high/critical | Ticket Validation Agent | Fan Experience Orchestration Agent; Alert Operations Team | ## Risk Routing; **What** – Route by low, medium, or high risk. **Why** – Focus human review only where necessary. |
| Alert Operations Team | Slack | Escalate high-risk bookings | Route by Risk Level | Merge All Paths | ## Risk Routing; **What** – Route by low, medium, or high risk. **Why** – Focus human review only where necessary. |
| OpenAI Model - Orchestration | OpenAI (LangChain Chat) | LLM backend for orchestration agent | — (AI connection) | Fan Experience Orchestration Agent (ai_languageModel) | ## Fan Orchestration; **What** – Confirm, reschedule, refund, escalate. **Why** – Enforce SLAs and protect fan experience. |
| Orchestration Output Parser | Structured Output Parser | Enforce JSON structure for orchestration output | — (AI connection) | Fan Experience Orchestration Agent (ai_outputParser) | ## Fan Orchestration; **What** – Confirm, reschedule, refund, escalate. **Why** – Enforce SLAs and protect fan experience. |
| Fan Experience Orchestration Agent | LangChain Agent | Produces action plan + ticketing update | Route by Risk Level (Low/Medium) | Update Ticketing System | ## Fan Orchestration; **What** – Confirm, reschedule, refund, escalate. **Why** – Enforce SLAs and protect fan experience. |
| Update Ticketing System | HTTP Request | Apply orchestration results to ticketing API | Fan Experience Orchestration Agent | Send Confirmation Email | ## Fan Orchestration; **What** – Confirm, reschedule, refund, escalate. **Why** – Enforce SLAs and protect fan experience. |
| Send Confirmation Email | Gmail | Send booking confirmation | Update Ticketing System | Merge All Paths | ## Audit Logging; **What** – Merge paths and log outcomes. **Why** – Maintain compliance and traceability. |
| Merge All Paths | Merge | Recombine success/escalation paths | Send Confirmation Email; Alert Operations Team | Log to Audit Trail | ## Audit Logging; **What** – Merge paths and log outcomes. **Why** – Maintain compliance and traceability. |
| Log to Audit Trail | Google Sheets | Append/update audit log row | Merge All Paths | — | ## Audit Logging; **What** – Merge paths and log outcomes. **Why** – Maintain compliance and traceability. |
| Sticky Note | Sticky Note | Documentation / context | — | — | ## Prerequisites… / Use Cases / Customization / Benefits (content-only node) |
| Sticky Note1 | Sticky Note | Documentation / setup steps | — | — | ## Setup Steps… (content-only node) |
| Sticky Note2 | Sticky Note | Documentation / how it works | — | — | ## How It Works… (content-only node) |
| Sticky Note3 | Sticky Note | Documentation / block label | — | — | ## Ingest & Ticket Validation… (content-only node) |
| Sticky Note4 | Sticky Note | Documentation / block label | — | — | ## Fan Orchestration… (content-only node) |
| Sticky Note5 | Sticky Note | Documentation / block label | — | — | ## Risk Routing… (content-only node) |
| Sticky Note6 | Sticky Note | Documentation / block label | — | — | ## Audit Logging… (content-only node) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook node** named **“Ticket Booking Webhook”**
   - Method: **POST**
   - Path: **concert-ticket-booking**
   - Response: **Last node**
   - Save to generate the production/test URL.
3. **Add Set node** named **“Workflow Configuration”**
   - Add fields:
     - `inventoryApiUrl` (string) → your inventory endpoint URL
     - `ticketingApiUrl` (string) → your ticketing system endpoint URL
     - `venueCapacity` (number) → e.g. 5000
     - `maxTicketsPerCustomer` (number) → e.g. 8
     - `fraudScoreThreshold` (number) → e.g. 75 (optional; not used directly by routing)
     - `highRiskThreshold` (number) → e.g. 80 (optional; not used directly by routing)
   - Enable **Include Other Fields**.
4. **Connect:** Ticket Booking Webhook → Workflow Configuration

5. **Add HTTP Request node** named **“Fetch Inventory Data”**
   - URL expression: `$('Workflow Configuration').first().json.inventoryApiUrl`
   - (Optional) Add authentication required by your inventory API (API key/OAuth2/etc.).
   - Header: `Content-Type: application/json`
6. **Connect:** Workflow Configuration → Fetch Inventory Data

7. **Add Set node** named **“Prepare Validation Input”**
   - Enable **Include Other Fields**
   - Add fields:
     - `bookingData` (object): `$('Ticket Booking Webhook').first().json.body`
     - `inventoryData` (object): `{{$json}}`
     - `venueCapacity` (number): `$('Workflow Configuration').first().json.venueCapacity`
     - `maxTicketsPerCustomer` (number): `$('Workflow Configuration').first().json.maxTicketsPerCustomer`
8. **Connect:** Fetch Inventory Data → Prepare Validation Input

9. **Add OpenAI Chat Model node (LangChain)** named **“OpenAI Model - Validation”**
   - Model: **gpt-4o**
   - Temperature: **0.1**
   - Set **OpenAI credentials** (API key) in n8n credentials and select it here.
10. **Add Structured Output Parser node** named **“Validation Output Parser”**
    - Paste a schema example matching the expected validation JSON (fields like riskLevel, riskScore, requiresHumanReview, etc.).
11. **Add AI Agent node (LangChain)** named **“Ticket Validation Agent”**
    - Prompt type: **Define**
    - Text input: `{{$json}}`
    - System message: paste the validation policy instructions (tier/seat/payment/fraud/etc. + risk bands).
    - Enable **Has Output Parser**.
    - Connect AI ports:
      - OpenAI Model - Validation → Ticket Validation Agent (**ai_languageModel**)
      - Validation Output Parser → Ticket Validation Agent (**ai_outputParser**)
12. **Connect:** Prepare Validation Input → Ticket Validation Agent

13. **Add Switch node** named **“Route by Risk Level”**
    - Rule set 1 output name: **Low/Medium Risk**
      - Conditions (OR):
        - `{{$json.output.riskLevel}}` equals `LOW`
        - `{{$json.output.riskLevel}}` equals `MEDIUM`
    - Rule set 2 output name: **High Risk - Human Review**
      - Conditions (OR):
        - `{{$json.output.riskLevel}}` equals `HIGH`
        - `{{$json.output.riskLevel}}` equals `CRITICAL`
        - `{{$json.output.requiresHumanReview}}` is true
    - Fallback output name: **Unclassified** (optional: connect to an error handler)
14. **Connect:** Ticket Validation Agent → Route by Risk Level

15. **Add OpenAI Chat Model node (LangChain)** named **“OpenAI Model - Orchestration”**
    - Model: **gpt-4o**
    - Temperature: **0.2**
    - Use same or separate OpenAI credentials.
16. **Add Structured Output Parser node** named **“Orchestration Output Parser”**
    - Paste schema example for orchestration output (actions array, ticketingSystemUpdate, auditLogSummary, etc.).
17. **Add AI Agent node (LangChain)** named **“Fan Experience Orchestration Agent”**
    - Text input: `{{$json}}`
    - System message: paste orchestration policies (refund windows, overbooking compensation, SLA rules, etc.)
    - Enable **Has Output Parser**
    - Connect AI ports:
      - OpenAI Model - Orchestration → Fan Experience Orchestration Agent (**ai_languageModel**)
      - Orchestration Output Parser → Fan Experience Orchestration Agent (**ai_outputParser**)
18. **Connect:** Route by Risk Level (Low/Medium Risk) → Fan Experience Orchestration Agent

19. **Add HTTP Request node** named **“Update Ticketing System”**
    - Method: **POST**
    - URL: `$('Workflow Configuration').first().json.ticketingApiUrl`
    - Body type: **JSON**
    - JSON body expression: `{{$json.output.ticketingSystemUpdate}}`
    - Header: `Content-Type: application/json`
    - Add ticketing API authentication as required.
20. **Connect:** Fan Experience Orchestration Agent → Update Ticketing System

21. **Add Gmail node** named **“Send Confirmation Email”**
    - Configure **Gmail OAuth2** credentials.
    - To:
      - `$('Fan Experience Orchestration Agent').first().json.output.actions.find(a => a.actionType === 'SEND_CONFIRMATION')?.details?.recipientEmail || $('Ticket Booking Webhook').first().json.body.customerEmail`
    - Subject:
      - `$('Fan Experience Orchestration Agent').first().json.output.actions.find(a => a.actionType === 'SEND_CONFIRMATION')?.details?.subject || 'Concert Ticket Confirmation'`
    - Message: HTML body referencing `ticketingSystemUpdate.bookingId/status/seatAssignments`
22. **Connect:** Update Ticketing System → Send Confirmation Email

23. **Add Slack node** named **“Alert Operations Team”**
    - Configure **Slack OAuth2** credentials (ensure chat:write scope).
    - Post to **channel** by **channelId**.
    - Message: include risk level/score, booking details, fraud indicators, reasoning.
24. **Connect:** Route by Risk Level (High Risk - Human Review) → Alert Operations Team

25. **Add Merge node** named **“Merge All Paths”**
    - Mode: **Choose Branch**
26. **Connect:**
    - Send Confirmation Email → Merge All Paths (Input 0)
    - Alert Operations Team → Merge All Paths (Input 1)

27. **Add Google Sheets node** named **“Log to Audit Trail”**
    - Configure **Google Sheets OAuth2** credentials.
    - Operation: **Append or Update**
    - Document ID: your spreadsheet ID
    - Sheet name: your tab name (e.g., `Audit_Log`)
    - Matching column: `bookingId`
    - Map columns (define below):
      - bookingId: orchestration bookingId fallback webhook bookingId
      - riskLevel, riskScore, validationStatus, requiresHumanReview: from validation agent
      - customerEmail: from webhook
      - timestamp: `{{$now.toISO()}}`
      - auditSummary: orchestration auditLogSummary fallback “Processed”
28. **Connect:** Merge All Paths → Log to Audit Trail

29. **Activate the workflow** and test:
    - POST to the webhook with a JSON body containing at least: `bookingId`, `customerEmail`, `ticketQuantity`, `totalAmount` (and any fraud/payment signals you want the agent to consider).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Setup Steps: Add OpenAI/Nvidia API credentials; configure ticketing endpoint; connect Gmail; connect Slack; connect Google Sheets; define risk thresholds and SLA rules. | From workflow sticky note “Setup Steps”. |
| Prerequisites: n8n account, OpenAI API key, ticketing API access, Gmail, Slack, Google Sheets. | From workflow sticky note “Prerequisites”. |
| Use Cases: Concert ticket sales, festival booking systems, venue seat management, VIP allocation handling. | From workflow sticky note “Use Cases”. |
| Customization: Adjust fraud thresholds, add SMS notifications, integrate CRM, extend loyalty logic. | From workflow sticky note “Customization”. |
| Benefits: Reduces fraud, automates fan communication, enforces ticketing policies. | From workflow sticky note “Benefits”. |
| How it works (summary): webhook intake → inventory fetch → validation agent risk classification → routing → orchestration agent actions → merge → audit log. | From workflow sticky note “How It Works”. |