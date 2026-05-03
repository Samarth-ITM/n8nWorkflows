Detect and score refund risk with Webhook, OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/detect-and-score-refund-risk-with-webhook--openai-and-google-sheets-12576


# Detect and score refund risk with Webhook, OpenAI and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Detect and score refund risk with Webhook, OpenAI and Google Sheets

**Purpose:**  
This workflow receives e-commerce order events (single order or bulk array) via a webhook, deduplicates already-processed rows, normalizes/validates the order data, requests a structured refund/chargeback risk evaluation from OpenAI, parses the AI response into strict fields, logs the result to Google Sheets, triggers alerts only for **HIGH** risk orders (Discord + Gmail), and finally updates the source row to mark it as processed.

**Target use cases**
- Refund/chargeback risk triage for finance/ops teams
- Automated screening of incoming orders
- Audit logging to Google Sheets and proactive alerting for high-risk cases

### Logical blocks (based on node-to-node dependencies)

1. **1.1 Ingest & Itemize Payload**: Webhook → split bulk payload → batch loop controller  
2. **1.2 Deduplication Gate**: skip items where `risk_processed` is already TRUE  
3. **1.3 Normalize & Validate Transaction Data**: enforce required fields and normalize types  
4. **1.4 AI Risk Scoring (OpenAI)**: send normalized data to an OpenAI chat model with strict JSON schema requirement  
5. **1.5 Parse & Merge AI Output**: parse JSON, normalize field names, merge with original order metadata  
6. **1.6 Logging**: append result to a Google Sheets “log” sheet  
7. **1.7 High-Risk Decision & Alerts**: if HIGH → Discord alert → finance email  
8. **1.8 Mark Source Row Processed + Continue Loop**: update source sheet row and continue batch loop

---

## 2. Block-by-Block Analysis

### 2.1 Ingest & Itemize Payload

**Overview:**  
Receives POST requests containing either one order object or a bulk array of orders under `body`. Splits array payloads into individual items and prepares looping through them.

**Nodes involved:**
- Webhook1
- Split Out
- Loop Over Items
- Sticky Note12 (comment)
- Sticky Note13 (comment)

#### Webhook1
- **Type / role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point.
- **Key configuration:**
  - Method: **POST**
  - Path: **`refund-risk`**
- **Input / output:**
  - Input: external HTTP POST
  - Output: one item containing request data (notably `body`)
  - Connects to **Split Out**
- **Potential failures / edge cases:**
  - Payload does not contain `body` or has unexpected shape (later nodes may fail).
  - Auth not configured (webhook is public unless additional webhook auth options are enabled).
  - Large bulk payloads may hit request size/time constraints.

#### Split Out
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — converts an array field into multiple items.
- **Key configuration:**
  - Field to split: `body` (expression `=body`)
- **Input / output:**
  - Input from **Webhook1**
  - Output to **Loop Over Items**
- **Edge cases:**
  - If `body` is not an array, behavior depends on node semantics; it may emit one item or fail depending on runtime version and data shape. (The later “Normalize Data” code explicitly supports both single and bulk.)

#### Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — loop controller.
- **Key configuration:**
  - Batch behavior: defaults (no explicit batch size shown)
- **Input / output connections:**
  - Input from **Split Out**
  - Output **(index 1)** goes to **DEDUPE CHECK** (the workflow uses the “loop/continue” branch rather than the first output).
  - Receives loopback input from **Update row in sheet** to continue processing next batch item(s).
- **Edge cases / pitfalls:**
  - Mis-wiring outputs can cause the loop not to iterate as expected. Here, the second output is used to feed the processing chain.
  - If any downstream node throws an error (e.g., invalid input, AI parse fail), the loop stops unless error handling is configured.

#### Sticky Note12 (covers ingest nodes)
- **Type / role:** Sticky Note — documentation.
- **Content:**
  - “Step 1 - Ingest & Prepare Data … Receives order data (single or bulk), splits arrays, and processes each transaction individually.”

#### Sticky Note13 (workflow header note)
- **Type / role:** Sticky Note — workflow description + setup tips.
- **Content highlights:**
  - Explains end-to-end flow: webhook → dedupe → normalize → OpenAI → Sheets log → alerts on HIGH → mark processed.
  - Setup steps and customization tips.

---

### 2.2 Deduplication Gate

**Overview:**  
Prevents reprocessing: only items with `risk_processed = FALSE` (case-normalized) proceed.

**Nodes involved:**
- DEDUPE CHECK
- Sticky Note15 (comment)

#### DEDUPE CHECK
- **Type / role:** IF (`n8n-nodes-base.if`) — filters items.
- **Key configuration:**
  - Condition: `String($json.risk_processed).toUpperCase() == "FALSE"`
- **Input / output:**
  - Input from **Loop Over Items** (output index 1)
  - True branch → **Normalize Data**
  - False branch: not connected (items effectively skipped)
- **Edge cases / pitfalls:**
  - If `risk_processed` is missing/undefined, `String(undefined).toUpperCase()` becomes `"UNDEFINED"` and will **not** match `"FALSE"`, so the item will be skipped (may be unintended).
  - If upstream uses boolean `false` rather than string `"FALSE"`, `String(false).toUpperCase()` becomes `"FALSE"` and will pass (good).

#### Sticky Note15
- **Content:**
  - “Step 2 – DEDUPE CHECK … Orders marked as `risk_processed = TRUE` are skipped.”

---

### 2.3 Normalize & Validate Transaction Data

**Overview:**  
Validates the minimum required field (`order_id`) and normalizes numeric fields and derived features (e.g., country mismatch). Produces a consistent schema for AI scoring and downstream logging.

**Nodes involved:**
- Normalize Data
- Sticky Note16 (comment)

#### Normalize Data
- **Type / role:** Code (`n8n-nodes-base.code`) — transformation + validation.
- **Key configuration choices:**
  - Mode: **Run once for each item**
  - Supports both:
    - bulk payload arrays (uses `$json.body[$itemIndex]`)
    - single payload objects (uses `$json.body ?? $json`)
  - Hard validation: throws error if missing `order_id`.
  - Normalizes:
    - numeric fields with `Number(...)` defaults to 0
    - `country_mismatch` computed from billing vs shipping
    - passes through `rowNumber`, `source`, `triggeredAt`, `risk_processed`
- **Inputs / outputs:**
  - Input from **DEDUPE CHECK**
  - Output to **Message a model2**
- **Edge cases / failure types:**
  - If upstream item does not include `body` but is also not the order object shape, it can throw “Invalid input”.
  - `rowNumber` is cast to `Number(d.rowNumber)`; if missing, becomes `NaN` (later sheet update may fail to match).
  - If currencies/countries are missing, they pass through as undefined; the AI prompt still renders them (could reduce model quality).

#### Sticky Note16
- **Content:**
  - “Step 3 – Normalize Data … Validates required fields and normalizes incoming data.”

---

### 2.4 AI Risk Scoring (OpenAI)

**Overview:**  
Sends normalized transaction data to an OpenAI chat model with strict instructions to return **only** valid JSON with a fixed schema.

**Nodes involved:**
- Message a model2
- Sticky Note10 (comment)

#### Message a model2
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — model inference.
- **Key configuration:**
  - Model: `chatgpt-4o-latest`
  - System prompt: positions the model as a “senior Risk Analytics Specialist”; requires explainable, proportional decisions.
  - User prompt: embeds transaction fields from `$json` (normalized data) and mandates strict JSON output:
    - `risk_score` integer 0–100
    - `risk_level` one of `LOW|MEDIUM|HIGH`
    - `key_risk_drivers` array of strings
    - `recommended_preventive_action` string
  - Additional constraints: “single refund alone must NOT result in HIGH”, “country mismatch alone must NOT result in HIGH”, “HIGH only with multiple strong indicators”.
- **Input / output:**
  - Input from **Normalize Data**
  - Output to **Parse AI Output**
- **Version-specific notes:**
  - Node typeVersion `2.1` indicates the newer LangChain-based OpenAI integration; output structure differs from legacy OpenAI nodes (which affects parsing).
- **Edge cases / failure types:**
  - Model may still return non-JSON or additional text; downstream parse will fail.
  - Credential/auth issues to OpenAI (API key/OAuth depending on setup).
  - Rate limiting / timeouts on large bursts (bulk webhook calls).

#### Sticky Note10
- **Content:**
  - “Step 4 – Message a model (OpenAI) … Sends normalized transaction data to OpenAI.”

---

### 2.5 Parse & Merge AI Output

**Overview:**  
Extracts the AI message text, strips possible code fences, parses JSON, normalizes field names, and merges AI outputs with original transaction identifiers for logging and alerts.

**Nodes involved:**
- Parse AI Output
- Sticky Note9 (comment)

#### Parse AI Output
- **Type / role:** Code (`n8n-nodes-base.code`) — parsing + normalization + merge.
- **Key configuration choices:**
  - Reads AI text from: `$json.output?.[0]?.content?.[0]?.text`
  - Removes potential ```json fences and trims
  - `JSON.parse()` to object
  - Normalizes:
    - `risk_level` sourced from multiple possible keys and uppercased
    - drivers from `key_risk_drivers` or `drivers`
  - Merges with original data using: `$items("Normalize Data")[0].json`
    - Returns `order_id`, `customer_email`, `rowNumber`, `order_value` (formatted string), plus AI fields.
- **Input / output:**
  - Input from **Message a model2**
  - Outputs to **Log Result** and also directly to **IF High Risk**
- **Critical pitfall (pairing/merge correctness):**
  - `$items("Normalize Data")[0]` always picks the **first** item from that node, not the current paired item, which can cause mismatched order IDs/emails when multiple items are processed in a loop. In n8n, correct pairing typically uses `$item` pairing or indexing aligned to current item rather than `[0]`.
- **Other edge cases / failures:**
  - If OpenAI output structure changes (different nesting), `aiText` will be missing → throws “AI response missing”.
  - Any non-JSON character will crash `JSON.parse()`.
  - If drivers are returned as a string instead of array, it becomes empty string due to the array check.

#### Sticky Note9
- **Content:**
  - “Step 5 – Parse AI Output … Cleans and parses the AI response.”

---

### 2.6 Logging (Google Sheets)

**Overview:**  
Appends a record of each scored transaction to a Google Sheet for audit/reporting, including an IST-formatted timestamp.

**Nodes involved:**
- Log Result
- Sticky Note14 (comment)

#### Log Result
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append a row to a log sheet.
- **Key configuration choices:**
  - Operation: **Append**
  - Mapping mode: define below (explicit column mapping)
  - Writes columns:
    - `order_id`, `customer_email`, `risk_score`, `risk_level`, `risk_drivers`, `recommendation_action`
    - `logged_at` generated by an inline JS expression that formats current time in **IST** (UTC+5:30) as `DD-MM-YY HH:mm:ss IST`
  - `ai_model_used` exists in the schema but is marked removed in the node’s internal schema; it is not mapped.
- **Input / output:**
  - Input from **Parse AI Output**
  - Output to **IF High Risk**
- **Credentials / requirements:**
  - Requires Google Sheets OAuth2 (or service account, depending on n8n setup).
  - **Document ID is empty** in the provided configuration; must be set for the node to work.
  - Sheet selection shows an internal numeric ID; ensure it points to the intended logging sheet.
- **Edge cases / failures:**
  - Missing/invalid Document ID → node fails.
  - Column name mismatch or protected sheet → append fails.
  - Timestamp expression is safe, but assumes runtime supports `Date` and template literals (standard in n8n code expressions).

#### Sticky Note14
- **Content:**
  - “Step 6 – Log Result (Google Sheets) … Appends the evaluation result to a logging sheet…”

---

### 2.7 High-Risk Decision & Alerts

**Overview:**  
Checks if `risk_level` is **HIGH**; only then it triggers real-time alerts to Discord and sends an email to finance/ops.

**Nodes involved:**
- IF High Risk
- Discord1
- Email Finance
- Sticky Note11 (comment)

#### IF High Risk
- **Type / role:** IF (`n8n-nodes-base.if`) — decision gate.
- **Key configuration:**
  - Condition: `$json.risk_level == "HIGH"` (case sensitive)
- **Input / output:**
  - Receives input from **Log Result**
  - Also receives input directly from **Parse AI Output** (parallel connection)
  - True branch → **Discord1**
  - False branch → **Update row in sheet** (mark processed even if not high risk)
- **Edge cases / pitfalls:**
  - Because it has **two upstream inputs**, it may run twice per item (once with the object coming from Parse AI Output, once with the object coming out of Log Result). This can cause duplicate Discord/email alerts or duplicate “mark processed” updates depending on execution timing and item pairing.
  - If `risk_level` is missing or not exactly `"HIGH"` (e.g., `"High"`), it won’t alert.
  - Strict type validation is enabled; malformed values can cause condition evaluation errors in some cases.

#### Discord1
- **Type / role:** Discord (`n8n-nodes-base.discord`) — sends a Discord message.
- **Key configuration:**
  - Authentication: **webhook**
  - Message template includes order details, risk score, drivers, recommendation, and ISO timestamp.
- **Input / output:**
  - Input from **IF High Risk** (true)
  - Output to **Email Finance**
- **Failure types:**
  - Invalid Discord webhook URL/credential → HTTP error.
  - Rate limiting for many high-risk events.

#### Email Finance
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends an email alert.
- **Key configuration:**
  - To + CC list: `user@example.com` placeholders
  - Subject includes an emoji and order id: “High Refund / Chargeback Risk – Order …”
  - HTML body references data from multiple nodes via expressions, including:
    - `$('Log Result').item.json.order_id`
    - `$('Normalize Data').item.json.order_value` and `.currency`
    - `$('IF High Risk').item.json.risk_score` etc.
- **Input / output:**
  - Input from **Discord1**
  - Output to **Update row in sheet**
- **Edge cases / pitfalls:**
  - Expressions mixing different node contexts can break if item pairing is off (especially with multi-input to IF node and the Parse AI merge issue).
  - Gmail credentials (OAuth2) required; sending limits may apply.

#### Sticky Note11
- **Content:**
  - “Step 7 - Risk Decision & Alerts … Triggers alerts only when risk level is HIGH…”

---

### 2.8 Mark Source Row Processed + Continue Loop

**Overview:**  
Updates the original/source sheet row for the processed transaction by setting `risk_processed = TRUE` using `row_number` matching, then continues the batch loop.

**Nodes involved:**
- Update row in sheet
- Sticky Note17 (comment)

#### Update row in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — updates an existing row by a matching key.
- **Key configuration choices:**
  - Operation: **Update**
  - Matching column: `row_number`
  - Values written:
    - `row_number` = `$('Normalize Data').item.json.rowNumber`
    - `risk_processed` = `"TRUE"`
  - Sheet name: set to placeholder `"your-sheet-value"`
  - **Document ID is empty** and must be configured
- **Input / output:**
  - Input from **IF High Risk** (false) and from **Email Finance** (after alerts)
  - Output loops back to **Loop Over Items** to continue processing next item(s)
- **Edge cases / failures:**
  - If `rowNumber` is `NaN`/missing, update will not match any row → leaves item unmarked, causing reprocessing later.
  - If the sheet uses a different row index concept than `row_number`, matching fails.
  - Permissions/protected ranges can prevent updates.

#### Sticky Note17
- **Content:**
  - “Step 8 – Mark as Processed … Updates the source row to ensure the transaction is not reprocessed.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook1 | Webhook | Receive incoming order payload(s) | (Trigger) | Split Out | ### Step 1 - Ingest & Prepare Data; Receives order data (single or bulk), splits arrays, and processes each transaction individually. |
| Split Out | Split Out | Split `body` array into items | Webhook1 | Loop Over Items | ### Step 1 - Ingest & Prepare Data; Receives order data (single or bulk), splits arrays, and processes each transaction individually. |
| Loop Over Items | Split In Batches | Batch/loop controller over items | Split Out; Update row in sheet | DEDUPE CHECK (output index 1) | ### Step 1 - Ingest & Prepare Data; Receives order data (single or bulk), splits arrays, and processes each transaction individually. |
| DEDUPE CHECK | IF | Skip already processed items | Loop Over Items | Normalize Data (true) | ### Step 2 – DEDUPE CHECK; Checks whether the transaction has already been processed. Orders marked as `risk_processed = TRUE` are skipped. |
| Normalize Data | Code | Validate + normalize transaction fields | DEDUPE CHECK | Message a model2 | ### Step 3 – Normalize Data; Validates required fields and normalizes incoming data. |
| Message a model2 | OpenAI (LangChain) | Request risk scoring with strict JSON schema | Normalize Data | Parse AI Output | ### Step 4 – Message a model (OpenAI); Sends normalized transaction data to OpenAI. |
| Parse AI Output | Code | Parse AI JSON and merge with original data | Message a model2 | Log Result; IF High Risk | ### Step 5 – Parse AI Output; Cleans and parses the AI response. |
| Log Result | Google Sheets | Append evaluation results to logging sheet | Parse AI Output | IF High Risk | ### Step 6 – Log Result (Google Sheets); Appends the evaluation result to a logging sheet. Includes timestamp, order details, risk level, drivers, and recommended action. |
| IF High Risk | IF | Branch on `risk_level == HIGH` | Log Result; Parse AI Output | Discord1 (true); Update row in sheet (false) | ### Step 7 - Risk Decision & Alerts; Triggers alerts only when risk level is HIGH and notifies relevant teams. |
| Discord1 | Discord | Send HIGH-risk alert to Discord | IF High Risk | Email Finance | ### Step 7 - Risk Decision & Alerts; Triggers alerts only when risk level is HIGH and notifies relevant teams. |
| Email Finance | Gmail | Email HIGH-risk details to finance/ops | Discord1 | Update row in sheet | ### Step 7 - Risk Decision & Alerts; Triggers alerts only when risk level is HIGH and notifies relevant teams. |
| Update row in sheet | Google Sheets | Mark source row as processed | IF High Risk (false); Email Finance | Loop Over Items | ### Step 8 – Mark as Processed; Updates the source row to ensure the transaction is not reprocessed. |
| Sticky Note9 | Sticky Note | Documentation | (none) | (none) | ### Step 5 – Parse AI Output; Cleans and parses the AI response. |
| Sticky Note10 | Sticky Note | Documentation | (none) | (none) | ### Step 4 – Message a model (OpenAI); Sends normalized transaction data to OpenAI. |
| Sticky Note11 | Sticky Note | Documentation | (none) | (none) | ### Step 7 - Risk Decision & Alerts; Triggers alerts only when risk level is HIGH and notifies relevant teams. |
| Sticky Note12 | Sticky Note | Documentation | (none) | (none) | ### Step 1 - Ingest & Prepare Data; Receives order data (single or bulk), splits arrays, and processes each transaction individually. |
| Sticky Note13 | Sticky Note | Documentation | (none) | (none) | ## Smart Refund Risk Detector; Describes how it works, setup steps, and customization tips. |
| Sticky Note14 | Sticky Note | Documentation | (none) | (none) | ### Step 6 – Log Result (Google Sheets); Appends the evaluation result to a logging sheet. |
| Sticky Note15 | Sticky Note | Documentation | (none) | (none) | ### Step 2 – DEDUPE CHECK; Checks whether the transaction has already been processed. |
| Sticky Note16 | Sticky Note | Documentation | (none) | (none) | ### Step 3 – Normalize Data; Validates required fields and normalizes incoming data. |
| Sticky Note17 | Sticky Note | Documentation | (none) | (none) | ### Step 8 – Mark as Processed; Updates the source row to ensure the transaction is not reprocessed. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Webhook” node** named **Webhook1**  
   - HTTP Method: **POST**  
   - Path: **refund-risk**  
   - (Optional) Configure webhook authentication if you don’t want a public endpoint.
3. **Add “Split Out” node** named **Split Out**  
   - Field to split out: **body**  
   - Connect: **Webhook1 → Split Out**
4. **Add “Split In Batches” node** named **Loop Over Items**  
   - Keep default batch options (or set a batch size you want)  
   - Connect: **Split Out → Loop Over Items**
   - Use the loop output that continues item processing (in this workflow, the second output is used).
5. **Add “IF” node** named **DEDUPE CHECK**  
   - Condition (string equals): `String($json.risk_processed).toUpperCase()` equals `FALSE`  
   - Connect: **Loop Over Items (processing output) → DEDUPE CHECK**
6. **Add “Code” node** named **Normalize Data**  
   - Mode: **Run once for each item**  
   - Implement logic to:
     - accept single item or array payloads under `body`
     - require `order_id`
     - normalize numeric fields and compute `country_mismatch`
     - include `rowNumber` for later updates  
   - Connect: **DEDUPE CHECK (true) → Normalize Data**
7. **Add “OpenAI (LangChain) / Message a model” node** named **Message a model2**  
   - Select model: **chatgpt-4o-latest** (or equivalent)  
   - Provide:
     - a System message defining the risk-analyst role
     - a User message that:
       - embeds normalized fields (order id/value/refund counts/account age/countries)
       - demands strict JSON output with keys:
         - `risk_score` (0–100 integer)
         - `risk_level` (LOW/MEDIUM/HIGH)
         - `key_risk_drivers` (array)
         - `recommended_preventive_action` (string)
   - **Credentials:** configure OpenAI credentials in n8n (API key or your platform’s supported method).
   - Connect: **Normalize Data → Message a model2**
8. **Add “Code” node** named **Parse AI Output**  
   - Mode: **Run once for each item**  
   - Steps to implement:
     - extract model text output from the OpenAI node output structure
     - strip any code fences if present
     - `JSON.parse` into an object
     - normalize `risk_level` to uppercase
     - join `key_risk_drivers` into a comma-separated string for alerts/sheets
     - merge with original identifiers (order_id, customer_email, rowNumber, order_value)  
   - Connect: **Message a model2 → Parse AI Output**
9. **Add “Google Sheets” node** named **Log Result** (Append)  
   - Operation: **Append**  
   - Configure **Google Sheets credentials** (OAuth2 / service account).  
   - Set:
     - **Document ID** (required)
     - Logging **Sheet name** (your log sheet tab)
   - Map columns at minimum:
     - `logged_at` (timestamp string)
     - `order_id`, `customer_email`
     - `risk_score`, `risk_level`, `risk_drivers`, `recommendation_action`
   - Connect: **Parse AI Output → Log Result**
10. **Add “IF” node** named **IF High Risk**  
   - Condition: `$json.risk_level` equals `HIGH`  
   - Connect: **Log Result → IF High Risk**
   - (Recommended) Avoid connecting Parse AI Output directly to IF High Risk to prevent double execution; keep a single upstream path.
11. **Add “Discord” node** named **Discord1** (Webhook auth)  
   - Set Discord webhook credentials  
   - Message content: include order id, customer, order value, risk score, drivers, recommendation, timestamp  
   - Connect: **IF High Risk (true) → Discord1**
12. **Add “Gmail” node** named **Email Finance**  
   - Configure Gmail OAuth2 credentials  
   - Set To/CC recipients  
   - Use HTML message body referencing the current item’s fields (`order_id`, `risk_score`, etc.)  
   - Connect: **Discord1 → Email Finance**
13. **Add “Google Sheets” node** named **Update row in sheet** (Update)  
   - Operation: **Update**  
   - Configure **Document ID** (required) and **Sheet name** (the source sheet you want to mark)  
   - Set matching column: **row_number**  
   - Update values:
     - `row_number` from the item’s `rowNumber`
     - `risk_processed` to `TRUE`  
   - Connect:
     - **IF High Risk (false) → Update row in sheet**
     - **Email Finance → Update row in sheet**
14. **Close the loop**  
   - Connect: **Update row in sheet → Loop Over Items** (so it processes the next item).
15. **Add sticky notes (optional)** to document steps (ingest, dedupe, normalize, AI call, parse, log, alerting, mark processed).
16. **Test**
   - Send a POST to the webhook with either:
     - a single order object in `body`, or
     - an array of order objects in `body`
   - Confirm:
     - Log rows are appended
     - HIGH risk triggers Discord + email
     - Source row gets updated to `risk_processed = TRUE`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Smart Refund Risk Detector: webhook ingestion → dedupe → normalization → OpenAI scoring (JSON schema) → Google Sheets logging → HIGH-risk alerts (Discord + email) → mark processed. | From the workflow’s main sticky note (“Smart Refund Risk Detector”). |
| Setup guidance: configure Webhook node; connect OpenAI, Google Sheets, Gmail, Discord credentials; map sheet columns including `row_number` and `risk_processed`; activate and test. | From the workflow’s setup steps sticky note. |
| Customization: adjust alert channels and risk thresholds; extend logging fields for dashboards. | From the workflow’s customization tips sticky note. |