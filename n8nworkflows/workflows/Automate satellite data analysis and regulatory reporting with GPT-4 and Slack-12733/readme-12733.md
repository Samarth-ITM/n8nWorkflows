Automate satellite data analysis and regulatory reporting with GPT-4 and Slack

https://n8nworkflows.xyz/workflows/automate-satellite-data-analysis-and-regulatory-reporting-with-gpt-4-and-slack-12733


# Automate satellite data analysis and regulatory reporting with GPT-4 and Slack

## 1. Workflow Overview

**Workflow name (JSON):** *Clinical Trial Data Integration and Regulatory Compliance Automation*  
**Provided title:** *Automate satellite data analysis and regulatory reporting with GPT-4 and Slack*

This workflow automates ingestion of external clinical trial data (EDC), normalizes and validates it, uses GPT-4-class agents to detect adverse events and generate regulatory-style reports, persists key outputs in Postgres, notifies stakeholders via Slack/email, and logs an audit trail. It supports **two entry points**: scheduled ingestion and event-driven ingestion via webhook.

### 1.1 Trigger & Ingestion Orchestration
Receives execution signals (schedule or webhook), merges them, and sets runtime configuration before pulling data from an external EDC API.

### 1.2 Data Preparation & Protocol Validation
Transforms raw EDC payloads into a normalized structure and validates them against study protocol rules.

### 1.3 AI Safety Processing (AE/SAE Detection)
Uses a LangChain Agent backed by OpenAI to identify and structure adverse events, then routes outcomes by severity.

### 1.4 Regulatory Report Generation & Persistence
For severe cases, enriches data, stores it, generates a structured report via a second AI agent, and stores the report.

### 1.5 Notifications, Submission Packaging & Audit Logging
Notifies the safety team (Slack) and QA (email), checks whether authority submission is required, prepares a submission package, and logs to an audit table.

> Note: Several sticky notes describe *satellite/environmental* use cases (ECC/NASA/ESA, SDQAR, Google Sheets, NVIDIA NIM). Those comments do **not** match the actual node set/configuration (which is clinical/EDC + Postgres + Slack + email + OpenAI). They are documented as-is in the tables/notes.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Ingestion Orchestration

**Overview:** Starts the workflow either on a schedule or via an incoming webhook, merges both trigger sources into a single execution path, applies configuration defaults, then fetches external clinical data via HTTP.

**Nodes involved:**
- Schedule Data Ingestion
- Event Webhook Trigger
- Merge Trigger Sources
- Workflow Configuration
- Fetch EDC Clinical Data

#### 2.1.1 Schedule Data Ingestion
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — time-based entry point.
- **Configuration choices:** Uses an interval rule, but the JSON shows an effectively empty interval object (`interval: [{}]`). In n8n this typically means the schedule is not fully configured (or was exported without UI-resolved values).
- **Inputs/outputs:** No inputs. Output → **Merge Trigger Sources** (input index 0).
- **Version notes:** typeVersion 1.3.
- **Failure modes / edge cases:**
  - Misconfigured interval can prevent triggering.
  - Timezone considerations if later logic assumes local time.

#### 2.1.2 Event Webhook Trigger
- **Type / role:** Webhook Trigger (`n8n-nodes-base.webhook`) — event-driven entry point.
- **Configuration choices:** Path is set to a UUID-like string (`6fb7d5bf-...`), default options.
- **Inputs/outputs:** No inputs. Output → **Merge Trigger Sources** (input index 1).
- **Version notes:** typeVersion 2.1.
- **Failure modes / edge cases:**
  - Webhook not registered (workflow inactive) or URL not exposed (networking/reverse proxy).
  - Unvalidated request payload shape can break downstream Code nodes if expected fields are missing.
  - Authentication is not configured in node options (no header/token validation), so endpoint may be publicly callable unless protected upstream.

#### 2.1.3 Merge Trigger Sources
- **Type / role:** Merge (`n8n-nodes-base.merge`) — unifies two triggers into one flow.
- **Configuration choices:** No merge mode specified in parameters (defaults apply). With triggers, the usual intent is “pass-through” depending on which input fires.
- **Inputs/outputs:** Inputs from **Schedule Data Ingestion** (index 0) and **Event Webhook Trigger** (index 1). Output → **Workflow Configuration**.
- **Version notes:** typeVersion 3.2.
- **Failure modes / edge cases:**
  - Default merge behavior may not be what you want (e.g., “wait for both inputs” vs “append”). Ensure it’s configured to work with independent triggers.

#### 2.1.4 Workflow Configuration
- **Type / role:** Set (`n8n-nodes-base.set`) — intended to define standard fields (environment, run metadata, API base URLs, etc.).
- **Configuration choices:** Currently empty (`options: {}`) with no explicit fields defined in JSON.
- **Inputs/outputs:** Input from **Merge Trigger Sources**. Output → **Fetch EDC Clinical Data**.
- **Version notes:** typeVersion 3.4.
- **Failure modes / edge cases:**
  - If later nodes reference config fields (not visible here), missing Set values will cause expression errors.

#### 2.1.5 Fetch EDC Clinical Data
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — pulls clinical data from an EDC or clinical system API.
- **Configuration choices:** No URL/method/headers visible in exported parameters (only `options:{}`), implying it is not configured or configuration was stripped.
- **Inputs/outputs:** Input from **Workflow Configuration**. Output → **Parse and Normalize Clinical Data**.
- **Version notes:** typeVersion 4.3.
- **Failure modes / edge cases:**
  - Missing URL/method/auth will cause immediate failure.
  - Common API issues: 401/403 auth, 429 rate limiting, 5xx, timeouts, pagination not handled.
  - Large payloads may exceed memory/time limits; consider pagination and streaming patterns.

---

### Block 2.2 — Data Preparation & Protocol Validation

**Overview:** Converts raw EDC API response into a normalized internal schema and validates the normalized data against protocol constraints before AI processing.

**Nodes involved:**
- Parse and Normalize Clinical Data
- Validate Against Study Protocols

#### 2.2.1 Parse and Normalize Clinical Data
- **Type / role:** Code (`n8n-nodes-base.code`) — custom transformation.
- **Configuration choices:** No code included in export; intended to map EDC fields (subject, visit, lab values, AE forms) into a consistent JSON structure.
- **Inputs/outputs:** Input from **Fetch EDC Clinical Data**. Output → **Validate Against Study Protocols**.
- **Version notes:** typeVersion 2.
- **Failure modes / edge cases:**
  - If input structure varies (different EDC endpoints), code may throw (undefined paths).
  - Date parsing/timezone issues, unit normalization, missing identifiers.
  - Multiple items vs single item: code may need to iterate item arrays correctly.

#### 2.2.2 Validate Against Study Protocols
- **Type / role:** Code (`n8n-nodes-base.code`) — rule validation and gating.
- **Configuration choices:** No code included; typical logic would check inclusion/exclusion constraints, required fields, visit windows, allowable ranges, and consistency checks.
- **Inputs/outputs:** Input from **Parse and Normalize Clinical Data**. Output → **Adverse Event Detection Agent**.
- **Version notes:** typeVersion 2.
- **Failure modes / edge cases:**
  - Over-strict validation could block legitimate records; under-strict could pass malformed data to AI.
  - If validation rejects, there is no explicit error/branch handling in the workflow as exported (no IF or error route here).

---

### Block 2.3 — AI Safety Processing (AE/SAE Detection)

**Overview:** Uses an AI agent with an OpenAI chat model and a structured output parser to detect adverse events and return machine-usable structured AE data, then routes downstream based on severity.

**Nodes involved:**
- Adverse Event Detection Agent
- OpenAI GPT-4 Model
- Structured AE Output Parser
- Route by Severity

#### 2.3.1 Adverse Event Detection Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM reasoning/tooling to detect AEs.
- **Configuration choices:** `hasOutputParser: true` and linked to a structured output parser node. Other options are empty in JSON.
- **Key variables/expressions:** Not visible; typically would pass normalized clinical data into the agent prompt/context.
- **Inputs/outputs:**
  - Main input from **Validate Against Study Protocols**
  - AI language model input from **OpenAI GPT-4 Model**
  - AI output parser input from **Structured AE Output Parser**
  - Main output → **Route by Severity**
- **Version notes:** typeVersion 3.1.
- **Failure modes / edge cases:**
  - Model can return non-conforming output; parser will fail if schema is strict.
  - Token limits if you pass full EDC payload; consider summarizing upstream.
  - Prompt injection risk if webhook payload is untrusted.

#### 2.3.2 OpenAI GPT-4 Model
- **Type / role:** OpenAI Chat Model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM to the AE agent.
- **Configuration choices:** Model = `gpt-4.1-mini`. No special options set; built-in tools empty.
- **Credentials:** `OpenAi account` (OpenAI API credential).
- **Inputs/outputs:** Connected via `ai_languageModel` → **Adverse Event Detection Agent**.
- **Version notes:** typeVersion 1.3.
- **Failure modes / edge cases:**
  - 401/429 errors from OpenAI.
  - Regional/data residency constraints for clinical data (PII/PHI compliance).
  - Latency/timeouts for large prompts.

#### 2.3.3 Structured AE Output Parser
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON/schema output.
- **Configuration choices:** Schema not shown in export; must be defined in node UI for reliable parsing.
- **Inputs/outputs:** Connected via `ai_outputParser` → **Adverse Event Detection Agent**.
- **Version notes:** typeVersion 1.3.
- **Failure modes / edge cases:**
  - If schema is missing/incorrect, parsing fails or produces incomplete structures.
  - Strict typing can fail on nulls/empty strings.

#### 2.3.4 Route by Severity
- **Type / role:** Switch (`n8n-nodes-base.switch`) — routes AE cases by severity (e.g., SAE vs non-serious).
- **Configuration choices:** The rule set is effectively blank in JSON (empty left/right values, equals comparison). This indicates it is not configured.
- **Inputs/outputs:** Input from **Adverse Event Detection Agent**. Output (only one connected) → **Enrich SAE Data**.
- **Version notes:** typeVersion 3.4.
- **Failure modes / edge cases:**
  - With empty rules, routing may never match (or may behave unexpectedly), preventing downstream steps.
  - Should explicitly reference something like `{{$json.severity}}` or `{{$json.isSAE}}`.

---

### Block 2.4 — Severe Case Enrichment, Report Generation & Storage

**Overview:** For severe adverse events, enriches the dataset, stores it to a validated database, generates a structured regulatory report via a second AI agent, and stores the final report to a repository (also Postgres here).

**Nodes involved:**
- Enrich SAE Data
- Store SAE in Validated DB
- Generate SUSAR Report
- OpenAI GPT-4 Model for Reports
- Structured Report Output Parser
- Store Report in Document Repository

#### 2.4.1 Enrich SAE Data
- **Type / role:** Set (`n8n-nodes-base.set`) — adds/derives fields needed for storage/reporting (e.g., case ID, MedDRA coding, narrative sections).
- **Configuration choices:** Empty in JSON; no fields defined.
- **Inputs/outputs:** Input from **Route by Severity**. Output → **Store SAE in Validated DB**.
- **Version notes:** typeVersion 3.4.
- **Failure modes / edge cases:**
  - Missing enrichment fields may break DB insert or report prompts.

#### 2.4.2 Store SAE in Validated DB
- **Type / role:** Postgres (`n8n-nodes-base.postgres`) — persists validated SAE case data.
- **Configuration choices:** Schema = `public`; **table is blank** in JSON (not configured).
- **Credentials:** Not shown in export (would be Postgres credential on the node in n8n).
- **Inputs/outputs:** Input from **Enrich SAE Data**. Output → **Generate SUSAR Report**.
- **Version notes:** typeVersion 2.6.
- **Failure modes / edge cases:**
  - Missing table name causes configuration error.
  - Schema mismatch, constraint violations, connection errors.
  - PHI/PII storage compliance requirements (encryption, access control).

#### 2.4.3 Generate SUSAR Report
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — drafts a SUSAR-style report from case data.
- **Configuration choices:** `hasOutputParser: true`; options empty.
- **Inputs/outputs:**
  - Main input from **Store SAE in Validated DB**
  - AI language model from **OpenAI GPT-4 Model for Reports**
  - AI output parser from **Structured Report Output Parser**
  - Main output → **Store Report in Document Repository**
- **Version notes:** typeVersion 3.1.
- **Failure modes / edge cases:**
  - If case data is incomplete, generated report may fail validation or be unsafe to submit.
  - Parser failures if the agent doesn’t comply with schema.
  - Regulatory wording requirements vary by region; prompts must be aligned.

#### 2.4.4 OpenAI GPT-4 Model for Reports
- **Type / role:** OpenAI Chat Model node — LLM backend for report agent.
- **Configuration choices:** Model = `gpt-4.1-mini`.
- **Credentials:** `OpenAi account`.
- **Inputs/outputs:** `ai_languageModel` → **Generate SUSAR Report**.
- **Version notes:** typeVersion 1.3.
- **Failure modes / edge cases:** Same as prior OpenAI model node (auth/rate limits/privacy).

#### 2.4.5 Structured Report Output Parser
- **Type / role:** Structured Output Parser — enforces a schema for the generated report (e.g., sections, metadata, submission identifiers).
- **Configuration choices:** Schema not visible in export; must be configured.
- **Inputs/outputs:** `ai_outputParser` → **Generate SUSAR Report**.
- **Version notes:** typeVersion 1.3.
- **Failure modes / edge cases:** Schema rigidity; null handling; missing required sections.

#### 2.4.6 Store Report in Document Repository
- **Type / role:** Postgres (`n8n-nodes-base.postgres`) — stores report content/metadata.
- **Configuration choices:** Schema = `public`; **table is blank**.
- **Inputs/outputs:** Input from **Generate SUSAR Report**. Output → **Notify Safety Team via Slack**.
- **Version notes:** typeVersion 2.6.
- **Failure modes / edge cases:**
  - Storing large report bodies may require `TEXT`/`JSONB` columns.
  - Missing table configuration.

---

### Block 2.5 — Notifications, Submission Package & Audit Trail

**Overview:** Notifies stakeholders, checks whether authority submission is required, prepares a submission package, and logs an audit trail to Postgres.

**Nodes involved:**
- Notify Safety Team via Slack
- Email QA Team
- Check if Authority Submission Required
- Prepare Authority Submission Package
- Log Audit Trail

#### 2.5.1 Notify Safety Team via Slack
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts messages to Slack (channel/user depends on node UI config).
- **Configuration choices:** OAuth2 authentication; `otherOptions` empty. Operation/channel/message not visible in export.
- **Credentials:** `Slack account` (Slack OAuth2).
- **Inputs/outputs:** Input from **Store Report in Document Repository**. Output → **Email QA Team**.
- **Version notes:** typeVersion 2.4.
- **Failure modes / edge cases:**
  - Slack OAuth scopes missing (e.g., `chat:write`).
  - Posting to a private channel requires bot membership.
  - Message formatting may fail if expressions reference missing report fields.

#### 2.5.2 Email QA Team
- **Type / role:** Email Send (`n8n-nodes-base.emailSend`) — emails QA stakeholders.
- **Configuration choices:** Options empty; SMTP/OAuth configuration not visible.
- **Inputs/outputs:** Input from **Notify Safety Team via Slack**. Output → **Check if Authority Submission Required**.
- **Version notes:** typeVersion 2.1.
- **Failure modes / edge cases:**
  - Missing SMTP credentials / OAuth setup.
  - Recipient/subject/body not configured (not visible in JSON).
  - Attachment handling (if needed) not configured.

#### 2.5.3 Check if Authority Submission Required
- **Type / role:** IF (`n8n-nodes-base.if`) — determines whether to build an authority submission package.
- **Configuration choices:** No conditions shown (options empty), so it is not configured.
- **Inputs/outputs:** Input from **Email QA Team**. Output (only “true” path is connected) → **Prepare Authority Submission Package**.
- **Version notes:** typeVersion 2.3.
- **Failure modes / edge cases:**
  - Without conditions, node may always route to the default branch depending on n8n behavior; must define a boolean condition (e.g., `{{$json.requiresSubmission}} === true`).
  - No “false” path handling is connected (no explicit end/log).

#### 2.5.4 Prepare Authority Submission Package
- **Type / role:** Code (`n8n-nodes-base.code`) — assembles required submission artifacts (PDF/XML/JSON, metadata, packaging).
- **Configuration choices:** Code not included in export.
- **Inputs/outputs:** Input from **Check if Authority Submission Required**. Output → **Log Audit Trail**.
- **Version notes:** typeVersion 2.
- **Failure modes / edge cases:**
  - Packaging formats vary (EMA/EudraVigilance, FDA, etc.); mismatch causes rejection.
  - File handling not represented elsewhere (no binary nodes). If you need files, add nodes for document generation/storage.

#### 2.5.5 Log Audit Trail
- **Type / role:** Postgres (`n8n-nodes-base.postgres`) — writes audit record(s).
- **Configuration choices:** Schema = `public`; **table is blank**.
- **Inputs/outputs:** Input from **Prepare Authority Submission Package**. No outgoing connections (workflow end).
- **Version notes:** typeVersion 2.6.
- **Failure modes / edge cases:**
  - Missing table configuration.
  - Audit completeness: ensure it logs trigger source, timestamps, record IDs, report version, submission decision.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Data Ingestion | Schedule Trigger | Time-based workflow entry | — | Merge Trigger Sources | ## How It Works; This workflow automates satellite data processing by ingesting raw geospatial data, applying AI analysis, and submitting formatted reports to regulatory authorities…; ## Data Ingestion; Fetches satellite imagery and climate sensor data from ECC APIs… |
| Event Webhook Trigger | Webhook | Event-driven workflow entry | — | Merge Trigger Sources | ## How It Works; This workflow automates satellite data processing…; ## Data Ingestion; Fetches satellite imagery and climate sensor data… |
| Merge Trigger Sources | Merge | Unify trigger sources into one path | Schedule Data Ingestion; Event Webhook Trigger | Workflow Configuration | ## How It Works; This workflow automates satellite data processing…; ## Data Ingestion; Fetches satellite imagery and climate sensor data… |
| Workflow Configuration | Set | Define runtime/config fields | Merge Trigger Sources | Fetch EDC Clinical Data | ## How It Works; This workflow automates satellite data processing…; ## Data Ingestion; Fetches satellite imagery and climate sensor data… |
| Fetch EDC Clinical Data | HTTP Request | Fetch external clinical/EDC data | Workflow Configuration | Parse and Normalize Clinical Data | ## How It Works; This workflow automates satellite data processing…; ## Data Ingestion; Fetches satellite imagery and climate sensor data… |
| Parse and Normalize Clinical Data | Code | Normalize/transform incoming data | Fetch EDC Clinical Data | Validate Against Study Protocols | ## How It Works; This workflow automates satellite data processing…; ## Data Ingestion; Fetches satellite imagery and climate sensor data… |
| Validate Against Study Protocols | Code | Validate normalized data vs protocol rules | Parse and Normalize Clinical Data | Adverse Event Detection Agent | ## How It Works; This workflow automates satellite data processing…; ## Data Ingestion; Fetches satellite imagery and climate sensor data… |
| Adverse Event Detection Agent | LangChain Agent | AI detection + structuring of adverse events | Validate Against Study Protocols | Route by Severity | ## How It Works; This workflow automates satellite data processing…; ## AI Analysis; Routes geospatial data through specialized models… |
| OpenAI GPT-4 Model | OpenAI Chat Model (LangChain) | LLM backend for AE agent | — | Adverse Event Detection Agent (ai_languageModel) | ## How It Works; This workflow automates satellite data processing…; ## AI Analysis; Routes geospatial data through specialized models… |
| Structured AE Output Parser | Structured Output Parser (LangChain) | Enforce structured AE JSON/schema | — | Adverse Event Detection Agent (ai_outputParser) | ## How It Works; This workflow automates satellite data processing…; ## AI Analysis; Routes geospatial data through specialized models… |
| Route by Severity | Switch | Branch cases by severity | Adverse Event Detection Agent | Enrich SAE Data | ## Prerequisites; Active satellite data API access (ECC, NASA, ESA)…; ## Use Cases; Automated climate monitoring…; ## Customization; Modify AI analysis prompts…; ## Benefits; Reduces satellite data processing time…; ## AI Analysis; Routes geospatial data through specialized models… |
| Enrich SAE Data | Set | Add fields for SAE persistence/reporting | Route by Severity | Store SAE in Validated DB | ## AI Analysis; Routes geospatial data through specialized models… |
| Store SAE in Validated DB | Postgres | Persist validated SAE cases | Enrich SAE Data | Generate SUSAR Report | ## Validation & Formatting; Validates processed data against regulatory specifications and formats into SDQAR-compliant report structures… |
| Generate SUSAR Report | LangChain Agent | Generate structured SUSAR report | Store SAE in Validated DB | Store Report in Document Repository | ## Validation & Formatting; Validates processed data against regulatory specifications and formats into SDQAR-compliant report structures… |
| OpenAI GPT-4 Model for Reports | OpenAI Chat Model (LangChain) | LLM backend for report agent | — | Generate SUSAR Report (ai_languageModel) | ## Validation & Formatting; Validates processed data against regulatory specifications… |
| Structured Report Output Parser | Structured Output Parser (LangChain) | Enforce structured report schema | — | Generate SUSAR Report (ai_outputParser) | ## Validation & Formatting; Validates processed data against regulatory specifications… |
| Store Report in Document Repository | Postgres | Persist generated report | Generate SUSAR Report | Notify Safety Team via Slack | ## Validation & Formatting; Validates processed data…; ## Authority Submission; Generates submission packages, stores reports… |
| Notify Safety Team via Slack | Slack | Stakeholder notification (Slack) | Store Report in Document Repository | Email QA Team | ## Authority Submission; Generates submission packages, stores reports… |
| Email QA Team | Email Send | Stakeholder notification (email) | Notify Safety Team via Slack | Check if Authority Submission Required | ## Authority Submission; Generates submission packages, stores reports… |
| Check if Authority Submission Required | IF | Decide whether to prepare authority submission | Email QA Team | Prepare Authority Submission Package | ## Authority Submission; Generates submission packages, stores reports… |
| Prepare Authority Submission Package | Code | Build submission package payload | Check if Authority Submission Required | Log Audit Trail | ## Authority Submission; Generates submission packages, stores reports… |
| Log Audit Trail | Postgres | Record audit log | Prepare Authority Submission Package | — | ## Authority Submission; Generates submission packages, stores reports… |
| Sticky Note | Sticky Note | Comment | — | — | ## Prerequisites; Active satellite data API access (ECC, NASA, ESA)… |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Setup Steps; 1. Configure ECC/climate API credentials… 7. Configure Gmail OAuth… |
| Sticky Note2 | Sticky Note | Comment | — | — | ## How It Works; This workflow automates satellite data processing… logs all activities to Google Sheets… |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Validation & Formatting; Validates processed data… SDQAR-compliant… |
| Sticky Note4 | Sticky Note | Comment | — | — | ## AI Analysis; Routes geospatial data through specialized models… |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Data Ingestion; Fetches satellite imagery and climate sensor data from ECC APIs… |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Authority Submission; Generates submission packages… notifies… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (e.g.) **Clinical Trial Data Integration and Regulatory Compliance Automation**.

2. **Add Trigger 1: Schedule**
   - Add node: **Schedule Trigger** → rename to **Schedule Data Ingestion**.
   - Configure a real schedule (e.g., every day/hour). Ensure interval is not empty.
   - This is an entry point.

3. **Add Trigger 2: Webhook**
   - Add node: **Webhook** → rename to **Event Webhook Trigger**.
   - Set a path (any unique string). Decide on method (POST recommended).
   - (Recommended) Add authentication (header token/basic auth) in node options or protect with a reverse proxy.
   - This is a second entry point.

4. **Merge both triggers**
   - Add node: **Merge** → rename to **Merge Trigger Sources**.
   - Connect:
     - **Schedule Data Ingestion → Merge Trigger Sources (Input 1)**
     - **Event Webhook Trigger → Merge Trigger Sources (Input 2)**
   - Set Merge mode suitable for triggers (commonly “Pass-through”/“Append” behavior; avoid “Wait for both” for independent triggers).

5. **Add configuration staging**
   - Add node: **Set** → rename to **Workflow Configuration**.
   - Connect: **Merge Trigger Sources → Workflow Configuration**.
   - Add fields you will use downstream, for example:
     - `edcBaseUrl`, `edcStudyId`, `runSource` (schedule/webhook), `sinceDate`, etc.

6. **Fetch clinical data from EDC**
   - Add node: **HTTP Request** → rename to **Fetch EDC Clinical Data**.
   - Connect: **Workflow Configuration → Fetch EDC Clinical Data**.
   - Configure:
     - Method: GET/POST as required by your EDC
     - URL: e.g. `{{$json.edcBaseUrl}}/studies/{{$json.edcStudyId}}/export`
     - Authentication: API key / OAuth2 / custom headers
     - Response format: JSON
     - Handle pagination if needed.

7. **Normalize incoming data**
   - Add node: **Code** → rename to **Parse and Normalize Clinical Data**.
   - Connect: **Fetch EDC Clinical Data → Parse and Normalize Clinical Data**.
   - Implement mapping to a stable schema (subjects, visits, AEs, labs). Ensure you output consistent JSON keys for later nodes.

8. **Validate against protocol rules**
   - Add node: **Code** → rename to **Validate Against Study Protocols**.
   - Connect: **Parse and Normalize Clinical Data → Validate Against Study Protocols**.
   - Implement checks (required fields, allowed ranges, visit windows). Decide how to handle failures (throw error vs mark record as invalid).

9. **Create the AE detection agent**
   - Add node: **AI Agent (LangChain)** → rename to **Adverse Event Detection Agent**.
   - Connect: **Validate Against Study Protocols → Adverse Event Detection Agent**.
   - In the agent, write instructions to extract AEs and classify severity.
   - Enable/attach an output parser (next steps).

10. **Add the LLM for AE detection**
    - Add node: **OpenAI Chat Model** → rename to **OpenAI GPT-4 Model**.
    - Set **Model** to `gpt-4.1-mini` (or your choice).
    - Configure **OpenAI API credentials** in n8n (Credentials → OpenAI).
    - Connect: **OpenAI GPT-4 Model → Adverse Event Detection Agent** via the **AI Language Model** connection type.

11. **Add structured output parsing for AE**
    - Add node: **Structured Output Parser** → rename to **Structured AE Output Parser**.
    - Define a schema, e.g. fields like `caseId`, `events[]`, `severity`, `isSAE`, `rationale`.
    - Connect: **Structured AE Output Parser → Adverse Event Detection Agent** via the **AI Output Parser** connection type.

12. **Route by severity**
    - Add node: **Switch** → rename to **Route by Severity**.
    - Connect: **Adverse Event Detection Agent → Route by Severity**.
    - Configure rules (example):
      - If `{{$json.isSAE}}` equals `true` → route to SAE branch.
      - Else → optionally route to a non-serious branch (not present in the exported workflow).

13. **Enrich SAE data**
    - Add node: **Set** → rename to **Enrich SAE Data**.
    - Connect: **Route by Severity → Enrich SAE Data** (SAE output).
    - Add derived fields needed for DB/reporting (e.g., `receivedAt`, `reportingRegion`, `product`, `investigatorSite`).

14. **Store SAE in Postgres**
    - Add node: **Postgres** → rename to **Store SAE in Validated DB**.
    - Connect: **Enrich SAE Data → Store SAE in Validated DB**.
    - Configure **Postgres credentials**.
    - Set **Schema** = `public`.
    - Select/create a **table** (required) such as `sae_cases_validated`.
    - Map input fields to columns.

15. **Create the report generation agent**
    - Add node: **AI Agent (LangChain)** → rename to **Generate SUSAR Report**.
    - Connect: **Store SAE in Validated DB → Generate SUSAR Report**.
    - Prompt it to produce the specific regulatory report structure you need.

16. **Add the LLM for report generation**
    - Add node: **OpenAI Chat Model** → rename to **OpenAI GPT-4 Model for Reports**.
    - Model: `gpt-4.1-mini`.
    - Use same or separate OpenAI credential.
    - Connect via **AI Language Model** → **Generate SUSAR Report**.

17. **Add structured output parsing for the report**
    - Add node: **Structured Output Parser** → rename to **Structured Report Output Parser**.
    - Define schema for the report (metadata + sections).
    - Connect via **AI Output Parser** → **Generate SUSAR Report**.

18. **Store report in repository (Postgres in this workflow)**
    - Add node: **Postgres** → rename to **Store Report in Document Repository**.
    - Connect: **Generate SUSAR Report → Store Report in Document Repository**.
    - Choose a table such as `susar_reports`.
    - Ensure column types support large text/JSON.

19. **Notify safety team in Slack**
    - Add node: **Slack** → rename to **Notify Safety Team via Slack**.
    - Connect: **Store Report in Document Repository → Notify Safety Team via Slack**.
    - Configure Slack OAuth2 credentials (Slack app with `chat:write`).
    - Set channel and message body (include report ID/link and severity).

20. **Email QA team**
    - Add node: **Send Email** → rename to **Email QA Team**.
    - Connect: **Notify Safety Team via Slack → Email QA Team**.
    - Configure SMTP or OAuth-based email credential.
    - Set recipients, subject, and body (optionally include report content or a link).

21. **Authority submission decision**
    - Add node: **IF** → rename to **Check if Authority Submission Required**.
    - Connect: **Email QA Team → Check if Authority Submission Required**.
    - Define condition (example): `{{$json.requiresAuthoritySubmission}} is true`.
    - Connect the **true** output to the next step. (Optionally connect false to an end/log step.)

22. **Prepare submission package**
    - Add node: **Code** → rename to **Prepare Authority Submission Package**.
    - Connect: **Check if Authority Submission Required (true) → Prepare Authority Submission Package**.
    - Build the payload/artifacts (often JSON + generated documents). If files are needed, add nodes for PDF generation and binary handling.

23. **Log audit trail**
    - Add node: **Postgres** → rename to **Log Audit Trail**.
    - Connect: **Prepare Authority Submission Package → Log Audit Trail**.
    - Configure table (e.g., `audit_log`) with fields like `timestamp`, `triggerSource`, `caseId`, `reportId`, `submissionRequired`, `status`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** Active satellite data API access (ECC, NASA, ESA) with authentication credentials. **Use Cases:** Automated climate monitoring with monthly regulatory submissions. **Customization:** Modify AI analysis prompts for specific environmental parameters. **Benefits:** Reduces satellite data processing time by 85% through end-to-end automation. | Sticky note content (does not match the clinical/EDC nodes present). |
| **Setup Steps:** 1) Configure ECC/climate API credentials… 2) Set up webhook endpoints… 3) Add OpenAI API key… 4) Configure NVIDIA NIM API… 5) Set Google Sheets credentials for audit logging… 6) Connect Slack workspace… 7) Configure Gmail OAuth… | Sticky note content (workflow uses Postgres for logging, not Google Sheets; no NVIDIA NIM node present). |
| **How It Works:** Describes satellite geospatial ingestion, AI analysis, SDQAR formatting, storage, authority submission, Slack/email notifications, and Google Sheets audit logging. | Sticky note content (high-level narrative; partially inconsistent with actual node implementations). |
| **Validation & Formatting:** “Formats into SDQAR-compliant report structures…” | Sticky note content (actual report agent is named SUSAR; storage is Postgres). |
| **AI Analysis:** “Routes geospatial data through specialized models…” | Sticky note content (actual AI nodes are generic OpenAI chat models + agents). |
| **Data Ingestion:** “Fetches satellite imagery and climate sensor data from ECC APIs…” | Sticky note content (actual ingestion node is generic HTTP request named EDC clinical data). |
| **Authority Submission:** “Generates submission packages… notifies… logs…” | Sticky note content (submission packaging is a Code node; logging is Postgres). |