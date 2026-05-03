Create Bosta shipping orders from Odoo invoices using OpenAI GPT models

https://n8nworkflows.xyz/workflows/create-bosta-shipping-orders-from-odoo-invoices-using-openai-gpt-models-12641


# Create Bosta shipping orders from Odoo invoices using OpenAI GPT models

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create Bosta shipping orders from Odoo invoices using OpenAI GPT models  
**Workflow name (n8n):** Odoo to Bosta - With Documentation  
**Status:** Inactive (active=false)

**Purpose:**  
This workflow receives an external trigger (Webhook), fetches an invoice and its customer + line items from **Odoo**, summarizes the items, enriches the context with **zones** from an n8n **Data Table**, uses **OpenAI GPT chat models (LangChain nodes)** to classify/normalize shipping address information, maps the resulting data into **Bosta** order payload fields, rounds the COD amount, creates a **Bosta shipping order** via HTTP request, and sends a **Telegram alert** if the Bosta API call fails.

### 1.1 Input Reception
Webhook entry point that starts the process, presumably carrying an identifier for the Odoo invoice/customer.

### 1.2 Odoo Data Retrieval
Fetches customer, invoice, splits/normalizes invoice structure (code), then fetches invoice line items.

### 1.3 Item Summarization + Zone Lookup
Summarizes line items into a compact description and fetches zone mappings (likely city/area → zoneId) from an n8n Data Table.

### 1.4 AI Address Classification (OpenAI via LangChain)
Prepares a prompt context, then runs an agent (“Classify Address”) backed by **two** GPT models (two language-model inputs) to classify/standardize the address / zone selection.

### 1.5 Payload Mapping + COD Rounding
Maps all required fields to Bosta’s expected schema and rounds COD before submission.

### 1.6 Bosta Order Creation + Failure Alerting
Creates the Bosta order via HTTP request; on error, routes to Telegram alert.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Receives an inbound HTTP request that triggers the workflow. The payload is expected to include identifiers needed to load Odoo records.

**Nodes involved:**
- Receive Trigger (Webhook)

#### Node: Receive Trigger
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point.
- **Configuration (interpreted):** Parameters are empty in the JSON export (so method/path/response mode are not visible here). A placeholder webhookId exists.
- **Key variables/expressions:** Not shown (no parameters provided).
- **Connections:**
  - **Output →** Get Customer
- **Version specifics:** TypeVersion **2.1** (Webhook node v2+ behavior; response settings differ from v1).
- **Edge cases / failures:**
  - Missing expected fields (e.g., invoice ID) will cascade into Odoo nodes failing or returning empty.
  - If webhook is configured with “Respond immediately” vs “Respond after workflow”, downstream failures may not be reflected to caller.

---

### Block 2 — Odoo Data Retrieval
**Overview:** Pulls customer and invoice information from Odoo, performs a custom “Split Lines” step, then retrieves invoice line items.

**Nodes involved:**
- Get Customer (Odoo)
- Get Invoice (Odoo)
- Split Lines (Code)
- Get Line Items (Odoo)

#### Node: Get Customer
- **Type / role:** `Odoo` — retrieves customer/partner data.
- **Configuration (interpreted):** Parameters are empty in export; credentials and model/operation would be configured in the UI (likely “Contact/Partner: Get” or “Search”).
- **Connections:**
  - **Input ←** Receive Trigger
  - **Output →** Get Invoice
- **Reliability:** `retryOnFail: true`
- **Edge cases / failures:**
  - Odoo auth/credential errors.
  - Incorrect model or missing partner ID in webhook payload.
  - Empty result set causing later expressions to fail.

#### Node: Get Invoice
- **Type / role:** `Odoo` — retrieves invoice record.
- **Configuration:** Not visible (empty parameters). Likely uses an invoice ID from webhook/customer node output.
- **Connections:**
  - **Input ←** Get Customer
  - **Output →** Split Lines
- **Reliability:** `retryOnFail: true`
- **Edge cases / failures:**
  - Invoice not found, not posted, or wrong company context.
  - Permissions/record rules in Odoo.

#### Node: Split Lines
- **Type / role:** `Code` — transforms the invoice payload into a structure suitable for line retrieval.
- **Configuration:** Empty parameters (actual JS not included), so only its role can be inferred from placement and name.
- **Connections:**
  - **Input ←** Get Invoice
  - **Output →** Get Line Items
- **Version specifics:** Code node TypeVersion **2** (newer code node runtime).
- **Edge cases / failures:**
  - If code assumes fields that aren’t present (e.g., `invoice_line_ids`), it can throw and stop execution.
  - If it emits no items, downstream nodes receive nothing.

#### Node: Get Line Items
- **Type / role:** `Odoo` — fetches invoice line items (likely via IDs produced by Split Lines).
- **Configuration:** Empty parameters.
- **Connections:**
  - **Input ←** Split Lines
  - **Output →** Summarize Items
- **Reliability:** `retryOnFail: true`
- **Edge cases / failures:**
  - If line IDs are missing/invalid, returns empty.
  - Odoo access rules for account.move.line.

---

### Block 3 — Item Summarization + Zone Lookup
**Overview:** Summarizes order items for shipping label/description and pulls zone mapping data from an n8n Data Table.

**Nodes involved:**
- Summarize Items (Code)
- Fetch Zones (Data Table)

#### Node: Summarize Items
- **Type / role:** `Code` — reduces multiple invoice lines into a concise summary (e.g., “2x T-shirt, 1x Shoes”).
- **Configuration:** Not visible (empty parameters).
- **Connections:**
  - **Input ←** Get Line Items
  - **Output →** Fetch Zones
- **Version specifics:** TypeVersion **2**
- **Edge cases / failures:**
  - Unexpected line schema (null product names, missing quantities).
  - Producing overly long strings could violate downstream API limits (Bosta notes/description limits).

#### Node: Fetch Zones
- **Type / role:** `Data Table` (n8n-nodes-base.dataTable) — reads zone reference data stored inside n8n.
- **Configuration:** Empty parameters; likely “Read rows” from a configured table (e.g., mapping city/area to Bosta zoneId).
- **Connections:**
  - **Input ←** Summarize Items
  - **Output →** Prep AI Context
- **Edge cases / failures:**
  - Missing/empty table or wrong table selected.
  - Zone data drift (Bosta zone IDs change, table not updated).

---

### Block 4 — AI Address Classification (OpenAI via LangChain)
**Overview:** Builds an LLM context combining customer address + zone mapping and uses a LangChain Agent node (“Classify Address”) with two OpenAI chat models to determine normalized shipping fields (e.g., city, zone, district, formatted address).

**Nodes involved:**
- Prep AI Context (Code)
- GPT Model (LangChain ChatOpenAI)
- GPT Model 2 (LangChain ChatOpenAI)
- Classify Address (LangChain Agent)

#### Node: Prep AI Context
- **Type / role:** `Code` — constructs prompt inputs / context for the agent (e.g., raw address + list of zones).
- **Configuration:** Not visible.
- **Connections:**
  - **Input ←** Fetch Zones
  - **Output →** Classify Address
- **Version specifics:** TypeVersion **2**
- **Edge cases / failures:**
  - Prompt too large if zones table is big (token overflow in LLM).
  - Missing customer address fields yields weak classification.

#### Node: GPT Model
- **Type / role:** `lmChatOpenAi` — provides an OpenAI chat model to the agent.
- **Configuration:** Not visible (model name, temperature, API key are configured in-node).
- **Connections:**
  - **AI output →** Classify Address (as `ai_languageModel`, index 0)
- **Version specifics:** TypeVersion **1.3** (LangChain integration versions matter for parameter names).
- **Edge cases / failures:**
  - OpenAI credential missing/invalid.
  - Rate limits / timeouts.
  - Model mismatch (e.g., selecting a model not available to the key).

#### Node: GPT Model 2
- **Type / role:** `lmChatOpenAi` — second OpenAI chat model for the agent (often used as fallback/alternative reasoning model).
- **Configuration:** Not visible.
- **Connections:**
  - **AI output →** Classify Address (as `ai_languageModel`, index 1)
- **Edge cases / failures:** same as GPT Model.

#### Node: Classify Address
- **Type / role:** `agent` (LangChain) — runs an agent that uses the provided LLM(s) to produce structured classification output used later for shipping creation.
- **Configuration (interpreted):**
  - `maxTries: 5`, `waitBetweenTries: 5000ms` (re-attempt agent execution).
  - `alwaysOutputData: false` (if it fails, downstream may not run).
- **Connections:**
  - **Input ←** Prep AI Context
  - **Input (AI models) ←** GPT Model, GPT Model 2
  - **Output →** Map Fields
- **Edge cases / failures:**
  - Non-deterministic outputs; agent may return unparseable text if downstream expects JSON.
  - Prompt injection risk if raw address contains hostile instructions (mitigate by strict output schema + validation).
  - Token limits and latency; multi-try can increase total runtime.

---

### Block 5 — Payload Mapping + COD Rounding
**Overview:** Converts Odoo + AI-enriched data into the Bosta API payload and rounds COD amount according to business rules.

**Nodes involved:**
- Map Fields (Set)
- Round COD (Code)

#### Node: Map Fields
- **Type / role:** `Set` — maps/renames/selects fields for Bosta order creation.
- **Configuration:** Empty parameters in export; typically this node defines explicit fields (recipient name, phone, address, zoneId, COD, notes, etc.).
- **Connections:**
  - **Input ←** Classify Address
  - **Output →** Round COD
- **Version specifics:** TypeVersion **3.4**
- **Edge cases / failures:**
  - If it references missing properties from agent output, expressions resolve to `null` or error depending on expression usage.
  - Data type mismatches (COD as string vs number).

#### Node: Round COD
- **Type / role:** `Code` — rounds cash-on-delivery amount before sending to Bosta.
- **Configuration:** Not visible.
- **Connections:**
  - **Input ←** Map Fields
  - **Output →** Create Bosta Order
- **Version specifics:** TypeVersion **2**
- **Edge cases / failures:**
  - Currency rounding rules: rounding up/down can cause reconciliation issues with Odoo totals.
  - Floating point precision; should use integer minor units if possible.

---

### Block 6 — Bosta Order Creation + Failure Alerting
**Overview:** Sends the prepared payload to Bosta’s API. If the HTTP request errors, it continues on an error output branch that triggers a Telegram alert.

**Nodes involved:**
- Create Bosta Order (HTTP Request)
- Alert Failure (Telegram)

#### Node: Create Bosta Order
- **Type / role:** `HTTP Request` — calls Bosta API endpoint to create an order.
- **Configuration:** Empty parameters in export (so URL/method/auth/body not visible). Notable runtime settings:
  - `onError: continueErrorOutput` (do not fail workflow; route error to secondary output)
  - `alwaysOutputData: false`
- **Connections:**
  - **Input ←** Round COD
  - **Output (success) →** none
  - **Output (error) →** Alert Failure
- **Version specifics:** TypeVersion **4.3**
- **Edge cases / failures:**
  - 4xx validation errors from Bosta (missing zoneId, invalid phone, invalid COD).
  - 401/403 due to missing/expired token.
  - Network timeouts; idempotency issues (duplicate order creation on retry outside n8n).

#### Node: Alert Failure
- **Type / role:** `Telegram` — sends a message when Bosta order creation fails.
- **Configuration:** Empty parameters in export; chat ID / message template configured in-node.
- **Connections:**
  - **Input ←** Create Bosta Order (error branch)
- **Version specifics:** TypeVersion **1.2**
- **Edge cases / failures:**
  - Telegram bot token invalid, chat not started with bot, blocked bot.
  - Message formatting errors if it interpolates missing error details.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Documentation | Sticky Note | Documentation container | — | — | |
| Section 1 | Sticky Note | Section header/notes | — | — | |
| Section 2 | Sticky Note | Section header/notes | — | — | |
| Section 3 | Sticky Note | Section header/notes | — | — | |
| Section 4 | Sticky Note | Section header/notes | — | — | |
| Section 5 | Sticky Note | Section header/notes | — | — | |
| Receive Trigger | Webhook | Entry point | — | Get Customer | |
| Get Customer | Odoo | Fetch customer/partner | Receive Trigger | Get Invoice | |
| Get Invoice | Odoo | Fetch invoice | Get Customer | Split Lines | |
| Split Lines | Code | Transform invoice structure for line retrieval | Get Invoice | Get Line Items | |
| Get Line Items | Odoo | Fetch invoice line items | Split Lines | Summarize Items | |
| Summarize Items | Code | Build short items description | Get Line Items | Fetch Zones | |
| Fetch Zones | Data Table | Load zone mapping reference | Summarize Items | Prep AI Context | |
| Prep AI Context | Code | Build LLM prompt/context | Fetch Zones | Classify Address | |
| GPT Model | LangChain ChatOpenAI | Primary LLM for agent | — | Classify Address (AI model idx 0) | |
| GPT Model 2 | LangChain ChatOpenAI | Secondary LLM for agent | — | Classify Address (AI model idx 1) | |
| Classify Address | LangChain Agent | Normalize/classify address & zone | Prep AI Context + GPT Model(s) | Map Fields | |
| Map Fields | Set | Map fields to Bosta payload | Classify Address | Round COD | |
| Round COD | Code | Round COD amount | Map Fields | Create Bosta Order | |
| Create Bosta Order | HTTP Request | Create shipping order in Bosta | Round COD | (success) none; (error) Alert Failure | |
| Alert Failure | Telegram | Notify on Bosta API failure | Create Bosta Order (error output) | — | |

*Note:* Sticky note contents are empty in the provided JSON, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: **“Odoo to Bosta - With Documentation”** (optional: keep inactive until tested).

2. **Add Webhook node** named **Receive Trigger**  
   - Node type: **Webhook**  
   - Configure:
     - HTTP Method (commonly POST)
     - Path (e.g., `/odoo/bosta`)
     - Response mode (as desired: immediate or after completion)
   - Ensure the webhook payload includes at minimum an identifier such as `invoice_id` (or any key you will use to query Odoo).

3. **Add Odoo node** named **Get Customer**  
   - Node type: **Odoo**
   - Credentials: configure **Odoo API credentials** (URL, DB, username/password or token depending on node support).
   - Operation: select an operation that retrieves the customer tied to the invoice:
     - Either search partner by a field from webhook payload, or
     - Derive partner from invoice (if webhook sends invoice id, you might move invoice retrieval before customer).
   - Enable **Retry on fail**.

4. **Add Odoo node** named **Get Invoice**  
   - Node type: **Odoo**
   - Operation: fetch the invoice (e.g., by `invoice_id` from webhook or from Get Customer output).
   - Enable **Retry on fail**.
   - Connect: **Get Customer → Get Invoice**.

5. **Add Code node** named **Split Lines**  
   - Node type: **Code**
   - Purpose: output one item containing the list of invoice line IDs (or a format suitable to fetch lines).
   - Example behavior to implement:
     - Read invoice line identifiers from the invoice record (e.g., `invoice_line_ids`)
     - Output them as an array or as separate items depending on how your Odoo “Get Line Items” node expects input.
   - Connect: **Get Invoice → Split Lines**.

6. **Add Odoo node** named **Get Line Items**  
   - Node type: **Odoo**
   - Operation: fetch invoice line items using IDs from Split Lines output.
   - Enable **Retry on fail**.
   - Connect: **Split Lines → Get Line Items**.

7. **Add Code node** named **Summarize Items**  
   - Node type: **Code**
   - Implement summarization (e.g., concatenate product names and quantities, omit shipping lines, ignore zero qty).
   - Connect: **Get Line Items → Summarize Items**.

8. **Add Data Table node** named **Fetch Zones**  
   - Node type: **Data Table**
   - Create/choose a Data Table that contains Bosta zone mapping (e.g., columns like `city`, `area`, `zoneId`, `zoneName`).
   - Operation: “Read” / “Get many rows” (depending on node UI).
   - Connect: **Summarize Items → Fetch Zones**.

9. **Add Code node** named **Prep AI Context**  
   - Node type: **Code**
   - Build a prompt/context object that includes:
     - Raw recipient name/phone/address from Odoo customer/invoice
     - Summarized items string
     - Zone list from Data Table
     - Instruction to produce **strict structured output** (recommended)
   - Connect: **Fetch Zones → Prep AI Context**.

10. **Add LangChain ChatOpenAI node** named **GPT Model**  
    - Node type: **OpenAI Chat Model (LangChain)**
    - Credentials: configure **OpenAI API key** in n8n credentials.
    - Choose model (e.g., GPT-4o-mini / GPT-4.1-mini depending on availability) and temperature appropriate for deterministic classification.
    - This node is not in the main chain; it connects as an AI language model to the agent.

11. **Add LangChain ChatOpenAI node** named **GPT Model 2**  
    - Configure similarly (may use same or different model as fallback).
    - Also connects only as a language model input to the agent.

12. **Add LangChain Agent node** named **Classify Address**  
    - Node type: **Agent**
    - Configure agent instructions to:
      - Normalize address
      - Pick best matching zoneId from the provided zone list
      - Return JSON fields needed by Bosta (e.g., `firstName`, `lastName`, `phone`, `address`, `city`, `zoneId`, `notes`)
    - Set:
      - **Max tries: 5**
      - **Wait between tries: 5000 ms**
    - Connect:
      - Main: **Prep AI Context → Classify Address**
      - AI Model inputs: **GPT Model → Classify Address (AI model input 0)** and **GPT Model 2 → Classify Address (AI model input 1)**

13. **Add Set node** named **Map Fields**  
    - Node type: **Set**
    - Define the outgoing Bosta payload fields using expressions referencing:
      - Odoo invoice totals/COD
      - Customer contact fields
      - Agent output fields (zoneId, formatted address, etc.)
    - Connect: **Classify Address → Map Fields**

14. **Add Code node** named **Round COD**  
    - Node type: **Code**
    - Implement rounding rule (e.g., nearest integer, ceiling, or currency-specific).
    - Output the final COD numeric field in the exact format required by Bosta.
    - Connect: **Map Fields → Round COD**

15. **Add HTTP Request node** named **Create Bosta Order**  
    - Node type: **HTTP Request**
    - Configure:
      - Method: likely **POST**
      - URL: Bosta order creation endpoint (per Bosta API)
      - Authentication: header token / bearer token as required
      - Body: JSON from “Round COD” output (your mapped payload)
    - Set **On Error** to **Continue (send to error output)** (matches `continueErrorOutput`).
    - Connect: **Round COD → Create Bosta Order**

16. **Add Telegram node** named **Alert Failure**  
    - Node type: **Telegram**
    - Credentials: Telegram bot token credential in n8n.
    - Configure chat ID and message. Recommended message content includes:
      - invoice reference
      - Bosta response status/body from the HTTP Request error output
    - Connect **Create Bosta Order (error output) → Alert Failure**.

17. **Test end-to-end**
    - Call the webhook with a sample payload.
    - Confirm Odoo nodes return expected records.
    - Validate agent output is parseable and contains required Bosta fields.
    - Verify Bosta order created; simulate failure to verify Telegram alerts.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist (“Main Documentation”, “Section 1–5”) but their contents are empty in the provided workflow export. | Internal documentation placeholders within the canvas. |