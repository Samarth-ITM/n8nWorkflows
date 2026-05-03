Orchestrate enterprise MCP AI tool access with Claude and Google Sheets

https://n8nworkflows.xyz/workflows/orchestrate-enterprise-mcp-ai-tool-access-with-claude-and-google-sheets-13592


# Orchestrate enterprise MCP AI tool access with Claude and Google Sheets

## 1. Workflow Overview

**Title:** Orchestrate enterprise MCP AI tool access with Claude and Google Sheets  
**Workflow name (n8n):** MCP Enterprise AI Control Layer with Claude AI  
**Purpose:** Provide a secure enterprise “control plane” for AI agents using the **Model Context Protocol (MCP)**. The workflow receives MCP requests via webhook, authenticates and validates them (JWT + tenant isolation), loads RBAC policies and session context from Google Sheets, asks **Claude** to plan compliant tool execution, dispatches tool calls to multiple enterprise systems, applies DLP/PII redaction, writes audit/session logs to Google Sheets, optionally sends alerts, and returns a MCP-compliant JSON response.

### Logical blocks
1.1 **Input Reception (MCP Webhook)**  
1.2 **Enterprise Auth, JWT Validation, RBAC & Context Assembly (Google Sheets)**  
1.3 **AI Orchestration & Policy Enforcement (Claude + parsing)**  
1.4 **Multi-System Tool Dispatch (HTTP to CRM/ERP/Analytics) + Merge**  
1.5 **Aggregation, DLP Redaction, Audit Logging, Session Update, Alerts, MCP Response**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (MCP Webhook)
**Overview:** Accepts a standardized MCP request payload from any AI agent platform and starts the workflow in “respond via Respond to Webhook node” mode.

**Nodes involved**
- Receive Enterprise AI Agent Request (Webhook)

**Node details**

**Receive Enterprise AI Agent Request**
- **Type / role:** `Webhook` trigger; entry point for MCP requests.
- **Config (interpreted):**
  - **Path:** `POST /mcp-enterprise`
  - **Response mode:** `responseNode` (workflow must end in a Respond to Webhook node)
- **Inputs/outputs:**
  - **Output:** to **Enterprise Auth, JWT and RBAC Validation**
- **Edge cases / failures:**
  - Invalid or missing JSON body.
  - Timeouts if downstream processing is slow (agent planning + tool calls).
  - If Respond to Webhook isn’t reached (errors), requester may get a generic error unless additional error handling is added.
- **Version notes:** Webhook node `typeVersion: 2`.

---

### 2.2 Enterprise Auth, JWT Validation, RBAC & Context Assembly
**Overview:** Validates MCP schema, performs simplified JWT checks, enforces tenant isolation, normalizes roles, and builds a normalized `enterpriseRequest`. Then it loads RBAC policies and session context from Google Sheets and produces a consolidated `contextPackage` plus an authorized tool list.

**Nodes involved**
- Enterprise Auth, JWT and RBAC Validation (Code)
- Fetch RBAC Policies for Role and Tools (Google Sheets)
- Fetch Active Session and Prior Context (Google Sheets)
- Build Enterprise Context Package (Code)

**Node details**

**Enterprise Auth, JWT and RBAC Validation**
- **Type / role:** `Code` node; gateway validation and normalization.
- **Key configuration choices:**
  - Validates required MCP fields: `mcpVersion, agentId, jwtToken, tenantId, userId, toolRequests, agentGoal`
  - Restricts MCP versions to: `1.0, 1.1, 1.2`
  - Validates JWT format (3 parts) and decodes payload (base64) **without signature verification** (explicitly simplified).
  - Checks `exp` and `iss === "enterprise-idp"`.
  - Tenant isolation: compares request `tenantId` with JWT claim `tenantId` or `org`.
  - Tool request limits: non-empty array, max 10 tools, `toolName` pattern `category.action`.
  - Role normalization + role level mapping.
  - Builds `enterpriseRequest` with governance and tracing fields (e.g., `gatewayRequestId`, `traceId`, `maxExecutionTimeMs` capped at 60s).
- **Variables produced:** `enterpriseRequest` object under `json.enterpriseRequest`
- **Connections:**
  - **Input:** Webhook
  - **Outputs:** parallel to both Google Sheets fetch nodes
- **Edge cases / failures:**
  - JWT decode errors; missing/invalid claims.
  - `iss` mismatch can block legitimate tokens if issuer differs.
  - Tenant mismatch blocks request.
  - Role not in allowlist blocks request.
  - Tool name format mismatch blocks request.
  - **Security gap:** no JWT signature validation; a forged token could pass these checks.
- **Version notes:** Code node `typeVersion: 2`.

**Fetch RBAC Policies for Role and Tools**
- **Type / role:** `Google Sheets` read; policy store lookup.
- **Config (interpreted):**
  - Document and sheet are placeholders (`=`) and must be configured.
  - `continueOnFail: true` means it will not stop the workflow if it fails.
- **Connections:**
  - **Input:** from Auth node
  - **Output:** to Build Enterprise Context Package
- **Edge cases / failures:**
  - Missing/incorrect Google credentials or permissions.
  - Sheet not found / wrong documentId.
  - Because `continueOnFail` is enabled, downstream code may receive empty policies and deny access (or behave unexpectedly depending on data shape).
- **Version notes:** `typeVersion: 4.5` (Google Sheets node).

**Fetch Active Session and Prior Context**
- **Type / role:** `Google Sheets` read; session registry / prior tool call history lookup.
- **Config (interpreted):**
  - Document and sheet placeholders (`=`).
  - `continueOnFail: true`
- **Connections:**
  - **Input:** from Auth node
  - **Output:** to Build Enterprise Context Package
- **Edge cases / failures:**
  - Empty result set leads to “new session” behavior.
  - Invalid JSON in stored `toolCallHistory` will crash parsing later unless handled.
- **Version notes:** `typeVersion: 4.5`.

**Build Enterprise Context Package**
- **Type / role:** `Code` node; merges request + RBAC + session context; computes authorization.
- **Key behaviors:**
  - Reads:
    - `enterpriseRequest` from the Auth node
    - `rbacPolicies` from Sheets (all rows)
    - `sessionData` from Sheets (all rows)
  - Builds `allowedTools` map from policy rows where `permission === "ALLOW"`.
  - Authorizes each requested tool if:
    - Exact tool policy exists, OR
    - Wildcard category policy like `crm.*` exists, OR
    - `roleLevel >= 5` (it_admin and above “bypass most restrictions”)
  - Denies request if **no tools** are authorized.
  - Session restoration:
    - Uses first row only as session record
    - Parses `toolCallHistory` JSON (defaults to `[]`)
  - Geo/time checks:
    - Business hours: 06:00–22:00 UTC (non-blocking informational)
    - Blocks restricted regions in a hardcoded list.
  - Outputs:
    - `enterpriseRequest`
    - `contextPackage` containing RBAC summary, authorizedRequests, sessionContext, environmentContext
- **Connections:**
  - **Inputs:** from both Sheets nodes
  - **Output:** to Claude agent node
- **Edge cases / failures:**
  - Policy field parsing assumptions (`allowedDataClasses`, `maxRecords`, booleans as strings).
  - Potential bug: `allowDelete: policy.allowDelete === 'false'` (this sets `allowDelete` true only when the string is `'false'`).
  - Session record selection: only first row is used even if multiple matches exist.
  - `JSON.parse(sessionRecord.toolCallHistory)` will throw if stored value is malformed.
- **Version notes:** Code node `typeVersion: 2`.

---

### 2.3 AI Orchestration & Policy Enforcement (Claude)
**Overview:** Uses Claude (Anthropic) to generate a structured JSON “execution plan” for the authorized tool requests with risk scoring, redaction guidance, parallelization/sequencing, and audit notes. Then enforces a hard policy block on CRITICAL risk and prepares data structures for dispatch.

**Nodes involved**
- Claude AI Contextual Orchestration and Tool Planning (LangChain Agent)
- Claude AI Model (Anthropic Chat Model)
- Parse Orchestration Plan and Enforce Policies (Code)

**Node details**

**Claude AI Contextual Orchestration and Tool Planning**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (LangChain agent); prompts orchestration logic.
- **Configuration choices:**
  - Uses a “define” prompt with a long instruction set embedding:
    - `enterpriseRequest` identifiers and governance constraints
    - `authorizedRequests`, denied tools, recent session tool calls
    - Enterprise policy rules (risk, mutating tools sequencing, tool chain limits, PII redaction)
  - Requires **JSON-only** output, with a specified schema including `executionPlan`, `parallelGroups`, risk levels, redaction flags, etc.
  - System message reinforces: JSON only, cautious governance.
- **Connections:**
  - **Input:** Build Enterprise Context Package
  - **AI language model input:** from Claude AI Model node
  - **Output:** to Parse Orchestration Plan and Enforce Policies
- **Edge cases / failures:**
  - Model may output invalid JSON or include extra text; downstream parser attempts cleanup but can still fail.
  - If authorized tool set is empty or insufficient, model may respond with plan that doesn’t match expectations.
- **Version notes:** `typeVersion: 1.6`.

**Claude AI Model**
- **Type / role:** `lmChatAnthropic` model provider for the agent.
- **Configuration:**
  - Model: `claude-sonnet-4-20250514`
  - Temperature: `0.15` (low variance, more deterministic)
  - Requires Anthropic credentials.
- **Connections:**
  - Connected to agent node as `ai_languageModel`.
- **Edge cases / failures:**
  - Anthropic auth errors, quota/rate limiting, or model name availability changes.
- **Version notes:** `typeVersion: 1`.

**Parse Orchestration Plan and Enforce Policies**
- **Type / role:** `Code` node; parses Claude JSON output and applies hard gates.
- **Key behaviors:**
  - Extracts model response from likely fields: `response/output/text` or `content[0].text`.
  - Strips ```json fences if present.
  - `JSON.parse` into `orchestrationPlan`; throws with truncated raw output on failure.
  - Hard blocks if `overallRiskLevel === "CRITICAL"`.
  - Computes:
    - `activeTools`: execution steps excluding skipped ones
    - `toolDispatchMap`: maps toolName → parameters from authorizedRequests
  - Outputs a dispatch-ready package including redaction flags and PII tool list.
- **Connections:**
  - **Input:** Claude agent output
  - **Output:** fan-out to all three dispatch HTTP nodes
- **Edge cases / failures:**
  - Any non-JSON response breaks flow.
  - No enforcement for “max 5 tool calls without elevated role” beyond the model’s compliance; there is no deterministic validation here.
  - If Claude outputs tools not in `authorizedRequests`, they may appear in `activeTools` but lack parameters; dispatch nodes send whatever is in `activeTools`.
- **Version notes:** Code node `typeVersion: 2`.

---

### 2.4 Multi-System Tool Dispatch + Merge
**Overview:** Sends tool execution batches to external system-specific “MCP dispatch” endpoints (CRM, ERP, Analytics/Warehouse). Results are merged for aggregation.

**Nodes involved**
- Dispatch — CRM System (HTTP Request)
- Dispatch — ERP System (HTTP Request)
- Dispatch — Analytics and Data Warehouse (HTTP Request)
- Merge All System Dispatch Results (Merge)

**Node details**

**Dispatch — CRM System**
- **Type / role:** `HTTP Request` to CRM dispatcher.
- **Configuration:**
  - POST `https://YOUR_CRM_API/api/v2/mcp-dispatch`
  - Timeout: 12s
  - JSON body includes:
    - `tools`: active tools whose name starts with `crm.`
    - `parameters`: toolDispatchMap filtered to `crm.`
    - `userId`, `tenantId`, `requestId`
  - Headers include Authorization placeholder and tenant/request metadata.
  - `continueOnFail: true`
- **Connections:** Input from Parse node; output to Merge node.
- **Edge cases:**
  - Placeholder URL/token must be replaced.
  - If there are zero `crm.` tools, it still calls endpoint with empty tools list (depends on your dispatcher’s behavior).
  - Auth/tenant header mismatch can cause 401/403.
- **Version notes:** HTTP node `typeVersion: 4.2`.

**Dispatch — ERP System**
- Same pattern as CRM:
  - POST `https://YOUR_ERP_API/api/v2/mcp-dispatch`, timeout 12s
  - Filters `erp.` tools/params
  - `continueOnFail: true`
- **Edge cases:** same as above.

**Dispatch — Analytics and Data Warehouse**
- Similar pattern, but:
  - POST `https://YOUR_ANALYTICS_API/api/v2/mcp-dispatch`, timeout 20s
  - Filters `analytics.` or `data.` tools/params
  - Includes `maxClassification`
  - Adds `X-Data-Classification` header
  - `continueOnFail: true`
- **Edge cases:** classification handling must be supported server-side.

**Merge All System Dispatch Results**
- **Type / role:** `Merge` node to combine results.
- **Configuration:** `mergeByPosition`
- **Connections:**
  - Inputs: CRM (index 0), ERP (index 1), Analytics (index 1) — note two are connected to index 1 in the JSON connections.
  - Output: Aggregate Results and Apply DLP Redaction
- **Edge cases / failures:**
  - `mergeByPosition` assumes consistent item positions; with `continueOnFail` and variable outputs, pairing can be unreliable.
  - Two inputs wired to the same index suggests the merge may not behave as intended; typically you’d use **Merge (Combine)** patterns or multiple merges chained.

---

### 2.5 Aggregation, DLP Redaction, Audit Logging, Session Update, Alerts, MCP Response
**Overview:** Aggregates dispatch results, applies DLP/PII redaction conditionally, generates execution summaries, sends alerts, writes audit and session logs to Google Sheets, formats the final MCP response, and returns it to the caller.

**Nodes involved**
- Aggregate Results and Apply DLP Redaction (Code)
- Send Policy and Security Alert (Email)
- Write SOC2 Compliance Audit Log (Google Sheets)
- Update Session Registry (Google Sheets)
- Build Final MCP Enterprise Response (Code)
- Return MCP Enterprise Result to Agent (Respond to Webhook)

**Node details**

**Aggregate Results and Apply DLP Redaction**
- **Type / role:** `Code` node; aggregation + redaction engine + summary computation.
- **Key behaviors:**
  - Pulls dispatch outputs directly from the three HTTP nodes.
  - Builds `rawResults` with statuses: SUCCESS/FAILED/SKIPPED.
  - DLP patterns redact:
    - emails, phone numbers, SSNs, credit cards, API keys/tokens/secrets-like strings
  - Deep redaction recursively up to depth 8; also redacts based on sensitive key names (`password`, `token`, `ssn`, `dob`, etc.)
  - Redaction is applied only if:
    - `orchestrationData.redactionRequired` is true **and**
    - user role is not in `[hr_admin, it_admin, super_admin]`
  - Produces `executionSummary` with per-system status counts.
- **Connections:**
  - Input: Merge node
  - Output: fan-out to Email, Audit Log sheet, Session Registry sheet
- **Edge cases / failures:**
  - If dispatch nodes return non-JSON or unexpected shapes, status logic may be misleading.
  - Redaction is pattern-based and may miss or over-redact.
  - If Claude recommends redaction but role has clearance, redaction is skipped entirely.
- **Version notes:** Code node `typeVersion: 2`.

**Send Policy and Security Alert**
- **Type / role:** `Email Send` (SMTP); not conditionally gated in the workflow.
- **Configuration:**
  - Subject uses risk level, tenant, agentId.
  - `toEmail` and `fromEmail` are placeholders (`=`) and must be configured.
  - `continueOnFail: true`
- **Connections:** Input from Aggregate; output to Build Final MCP Enterprise Response.
- **Edge cases / failures:**
  - As built, it attempts to send an email **for every execution**, not only high-risk ones.
  - SMTP auth issues, invalid to/from addresses.
- **Version notes:** `typeVersion: 2.1`.

**Write SOC2 Compliance Audit Log**
- **Type / role:** `Google Sheets` append; audit trail.
- **Configuration:**
  - Operation: `append`
  - DocumentId and sheetName placeholders (`=`).
  - Column mapping is empty (must be defined).
  - `continueOnFail: true`
- **Connections:** Input from Aggregate; output to Build Final MCP Enterprise Response.
- **Edge cases:**
  - With no column mapping, append may fail or append empty rows depending on node behavior.
  - Sheet permissions, quotas, schema mismatch.
- **Version notes:** `typeVersion: 4.5`.

**Update Session Registry**
- **Type / role:** `Google Sheets` append; session tracking.
- **Configuration:** similar to audit log (append, placeholders, empty mapping), `continueOnFail: true`
- **Connections:** Input from Aggregate; output to Build Final MCP Enterprise Response.
- **Edge cases:** same as above.

**Build Final MCP Enterprise Response**
- **Type / role:** `Code` node; formats the final response envelope.
- **Output structure:**
  - MCP envelope fields: `mcpVersion`, `gatewayRequestId`, `requestId` (uses `sessionId`), `traceId`
  - `success`, `isDryRun`
  - `results` (redacted or raw)
  - `executionSummary`
  - `orchestrationInsights` (goal alignment, gaps, warnings, skipped tools)
  - `governance` (risk, redaction, approvals, classification, denied tools)
  - `respondedAt`
- **Connections:** Input from the three branches (email/log/session); output to Respond to Webhook.
- **Edge cases / failures:**
  - Because it receives **three inputs**, you may emit **multiple responses** attempts; only the first successful Respond to Webhook will matter, others may error or be ignored. Usually you would merge these branches before responding.
- **Version notes:** Code node `typeVersion: 2`.

**Return MCP Enterprise Result to Agent**
- **Type / role:** `Respond to Webhook`; returns final JSON to original caller.
- **Configuration:**
  - Respond with JSON stringified body.
  - Response headers set:
    - `X-MCP-Gateway-Request-ID` = gatewayRequestId
    - `X-MCP-Risk-Level` = governance.overallRiskLevel
    - `X-MCP-Tenant-ID` = `gatewayRequestId.split('-')[0]` (**likely incorrect**, will return `ECTL` not tenant)
    - `X-MCP-Trace-ID` = traceId
- **Edge cases / failures:**
  - Header `X-MCP-Tenant-ID` should likely be `tenantId`, not derived from `gatewayRequestId`.
  - If multiple upstream branches call this, n8n may warn about responding multiple times.
- **Version notes:** `typeVersion: 1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation banner / overall description |  |  | ## MCP Enterprise AI Control Layer with Claude AI … (full enterprise feature description and setup steps + sample MCP request) |
| Sticky Note1 | Sticky Note | Section label |  |  | ## 1. Enterprise Auth, RBAC & Context Assembly |
| Sticky Note2 | Sticky Note | Section label |  |  | ## 2. AI Orchestration & Policy Enforcement |
| Sticky Note3 | Sticky Note | Section label |  |  | ## 3. Multi-System Tool Dispatch |
| Sticky Note4 | Sticky Note | Section label |  |  | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |
| Receive Enterprise AI Agent Request | Webhook | MCP request intake | — | Enterprise Auth, JWT and RBAC Validation | ## 1. Enterprise Auth, RBAC & Context Assembly |
| Enterprise Auth, JWT and RBAC Validation | Code | MCP schema validation + JWT decode checks + tenant isolation + role normalization | Receive Enterprise AI Agent Request | Fetch RBAC Policies for Role and Tools; Fetch Active Session and Prior Context | ## 1. Enterprise Auth, RBAC & Context Assembly |
| Fetch RBAC Policies for Role and Tools | Google Sheets | Load RBAC policies | Enterprise Auth, JWT and RBAC Validation | Build Enterprise Context Package | ## 1. Enterprise Auth, RBAC & Context Assembly |
| Fetch Active Session and Prior Context | Google Sheets | Load session registry / tool call history | Enterprise Auth, JWT and RBAC Validation | Build Enterprise Context Package | ## 1. Enterprise Auth, RBAC & Context Assembly |
| Build Enterprise Context Package | Code | Compute authorized tools + session/environment context | Fetch RBAC Policies for Role and Tools; Fetch Active Session and Prior Context | Claude AI Contextual Orchestration and Tool Planning | ## 1. Enterprise Auth, RBAC & Context Assembly |
| Claude AI Contextual Orchestration and Tool Planning | LangChain Agent | Claude-driven planning, risk scoring, governance guidance | Build Enterprise Context Package | Parse Orchestration Plan and Enforce Policies | ## 2. AI Orchestration & Policy Enforcement |
| Claude AI Model | Anthropic Chat Model | LLM backend for the agent | — | Claude AI Contextual Orchestration and Tool Planning (ai_languageModel) | ## 2. AI Orchestration & Policy Enforcement |
| Parse Orchestration Plan and Enforce Policies | Code | Parse JSON plan + hard policy block + dispatch prep | Claude AI Contextual Orchestration and Tool Planning | Dispatch — CRM System; Dispatch — ERP System; Dispatch — Analytics and Data Warehouse | ## 2. AI Orchestration & Policy Enforcement |
| Dispatch — CRM System | HTTP Request | Execute CRM tools via system dispatcher | Parse Orchestration Plan and Enforce Policies | Merge All System Dispatch Results | ## 3. Multi-System Tool Dispatch |
| Dispatch — ERP System | HTTP Request | Execute ERP tools via system dispatcher | Parse Orchestration Plan and Enforce Policies | Merge All System Dispatch Results | ## 3. Multi-System Tool Dispatch |
| Dispatch — Analytics and Data Warehouse | HTTP Request | Execute analytics/data tools via system dispatcher | Parse Orchestration Plan and Enforce Policies | Merge All System Dispatch Results | ## 3. Multi-System Tool Dispatch |
| Merge All System Dispatch Results | Merge | Combine dispatch outputs | Dispatch — CRM System; Dispatch — ERP System; Dispatch — Analytics and Data Warehouse | Aggregate Results and Apply DLP Redaction | ## 3. Multi-System Tool Dispatch |
| Aggregate Results and Apply DLP Redaction | Code | Aggregate + conditional PII/DLP redaction + execution summary | Merge All System Dispatch Results | Send Policy and Security Alert; Write SOC2 Compliance Audit Log; Update Session Registry | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |
| Send Policy and Security Alert | Email Send (SMTP) | Send risk/security notification | Aggregate Results and Apply DLP Redaction | Build Final MCP Enterprise Response | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |
| Write SOC2 Compliance Audit Log | Google Sheets | Append audit log row(s) | Aggregate Results and Apply DLP Redaction | Build Final MCP Enterprise Response | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |
| Update Session Registry | Google Sheets | Append session record row(s) | Aggregate Results and Apply DLP Redaction | Build Final MCP Enterprise Response | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |
| Build Final MCP Enterprise Response | Code | Construct MCP-compliant response JSON | Send Policy and Security Alert; Write SOC2 Compliance Audit Log; Update Session Registry | Return MCP Enterprise Result to Agent | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |
| Return MCP Enterprise Result to Agent | Respond to Webhook | Return JSON response to agent platform | Build Final MCP Enterprise Response | — | ## 4. Aggregation, DLP Redaction, Audit & MCP Response |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **MCP Enterprise AI Control Layer with Claude AI** (or your preferred name).

2) **Add the Webhook trigger**
- Add node: **Webhook**
- Name: **Receive Enterprise AI Agent Request**
- Method: **POST**
- Path: **mcp-enterprise**
- Response mode: **Respond to Webhook node**
- Connect it to the auth node (next step).

3) **Add MCP + JWT validation**
- Add node: **Code**
- Name: **Enterprise Auth, JWT and RBAC Validation**
- Mode: **Run once for each item**
- Paste logic implementing:
  - required field checks
  - supported MCP version list
  - JWT format check and payload decode
  - `exp` check, `iss` check, tenant match check
  - toolRequests constraints (1–10) and toolName regex
  - role normalization + role level mapping
  - build `enterpriseRequest` object
- Output: `{ enterpriseRequest }`

4) **Add Google Sheets RBAC policy fetch**
- Add node: **Google Sheets**
- Name: **Fetch RBAC Policies for Role and Tools**
- Operation: choose a read operation appropriate to your sheet (e.g., “Read” / “Get many rows” depending on your n8n version)
- Set **Document** (Spreadsheet) and **Sheet**
- Configure Google Sheets OAuth2 credentials
- Enable **Continue On Fail** (optional; matches the provided workflow)
- Connect from **Enterprise Auth, JWT and RBAC Validation** to this node.

5) **Add Google Sheets session fetch**
- Add node: **Google Sheets**
- Name: **Fetch Active Session and Prior Context**
- Configure Document/Sheet for session registry
- Credentials: same Google Sheets OAuth2
- Enable **Continue On Fail** (optional)
- Connect from **Enterprise Auth, JWT and RBAC Validation** to this node.

6) **Build the enterprise context package**
- Add node: **Code**
- Name: **Build Enterprise Context Package**
- Implement logic to:
  - read `enterpriseRequest` from the auth node
  - read all RBAC rows and build an allowlist map
  - determine `authorizedRequests` vs `unauthorizedTools`
  - optionally bypass restrictions for high roles (roleLevel >= 5)
  - restore session history from sheet field like `toolCallHistory`
  - enforce geo restrictions
  - output `{ enterpriseRequest, contextPackage }`
- Connect both Google Sheets nodes into this Code node.

7) **Add Claude orchestration agent**
- Add node: **AI Agent** (LangChain Agent node in n8n)
- Name: **Claude AI Contextual Orchestration and Tool Planning**
- Prompt type: **Define**
- Provide:
  - A system message that enforces “JSON only”
  - A user prompt that embeds `enterpriseRequest` and `contextPackage` and asks for the fixed JSON schema (execution plan, risk, redaction, etc.)
- Connect **Build Enterprise Context Package** → this agent node.

8) **Add Anthropic model node**
- Add node: **Anthropic Chat Model**
- Name: **Claude AI Model**
- Model: `claude-sonnet-4-20250514` (or your available equivalent)
- Temperature: `0.15`
- Configure **Anthropic API credentials**
- Connect this model node to the agent node via the **AI language model** connector.

9) **Parse the orchestration plan + enforce hard policies**
- Add node: **Code**
- Name: **Parse Orchestration Plan and Enforce Policies**
- Parse agent output as JSON (strip code fences if needed)
- Hard-block CRITICAL risk
- Compute:
  - `activeTools`
  - `toolDispatchMap` (tool → parameters) from authorizedRequests
  - governance flags like `redactionRequired`, `requiresHumanApproval`
- Connect agent node → this code node.

10) **Add HTTP dispatch nodes (CRM/ERP/Analytics)**
- Add node: **HTTP Request** (3 times)

  **A. Dispatch — CRM System**
  - POST `https://YOUR_CRM_API/api/v2/mcp-dispatch`
  - Timeout ~12000ms
  - JSON body: tools filtered by prefix `crm.`, plus parameters map filtered by prefix `crm.`
  - Headers: Authorization bearer token, tenant ID, gateway request ID, role, content-type

  **B. Dispatch — ERP System**
  - POST `https://YOUR_ERP_API/api/v2/mcp-dispatch`
  - Similar filtering for `erp.`

  **C. Dispatch — Analytics and Data Warehouse**
  - POST `https://YOUR_ANALYTICS_API/api/v2/mcp-dispatch`
  - Filter for `analytics.` and `data.`
  - Include classification fields/headers

- Enable **Continue On Fail** on each if you want partial results.
- Connect **Parse Orchestration Plan and Enforce Policies** to all three dispatch nodes.

11) **Merge dispatch results**
- Add node: **Merge**
- Name: **Merge All System Dispatch Results**
- Mode: pick a merge strategy that matches your runtime behavior. (The provided workflow uses **merge by position**, but for robustness you may prefer a deterministic merge approach.)
- Connect the three dispatch nodes into the merge node.

12) **Aggregate + DLP redaction**
- Add node: **Code**
- Name: **Aggregate Results and Apply DLP Redaction**
- Implement:
  - Collect results from each dispatch node
  - Build `rawResults` and a per-system status summary
  - Apply deep redaction using regex patterns if `redactionRecommended` and role lacks clearance
- Connect Merge → this node.

13) **Send alert (SMTP)**
- Add node: **Email Send**
- Name: **Send Policy and Security Alert**
- Configure SMTP credentials
- Set To/From addresses (and ideally add IF logic so it only triggers on MEDIUM/HIGH/CRITICAL)
- Connect Aggregate → Email node.

14) **Write audit log to Google Sheets**
- Add node: **Google Sheets**
- Name: **Write SOC2 Compliance Audit Log**
- Operation: **Append**
- Configure Document/Sheet and define column mapping (e.g., requestId, tenantId, userId, tools, risk score, redaction applied, timestamps, etc.)
- Connect Aggregate → Audit node.

15) **Update session registry in Google Sheets**
- Add node: **Google Sheets**
- Name: **Update Session Registry**
- Operation: **Append** (or “Upsert” if you maintain one row per session)
- Configure mapping (sessionId, createdAt/updatedAt, toolCallHistory, etc.)
- Connect Aggregate → Session node.

16) **Build the final MCP response**
- Add node: **Code**
- Name: **Build Final MCP Enterprise Response**
- Combine:
  - MCP envelope fields
  - aggregated results and execution summary
  - orchestration insights and governance metadata
- Connect Email + Audit + Session nodes into this node (or better: merge them first, then connect once).

17) **Respond to the webhook**
- Add node: **Respond to Webhook**
- Name: **Return MCP Enterprise Result to Agent**
- Respond with: **JSON**
- Body: the output of the previous node
- Add response headers as desired (gateway request id, risk level, trace id).
- Connect Build Final MCP Enterprise Response → Respond to Webhook.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “MCP Enterprise AI Control Layer with Claude AI… secure, scalable enterprise AI orchestration layer built on MCP… includes RBAC/ABAC, DLP redaction, SOC2/ISO27001 audit trail, geo/time policy, multi-tenant isolation.” | Included in the large sticky note description at the top of the canvas |
| Setup requires credentials for Anthropic, Google Sheets, SMTP/Slack, and a JWT secret/validation approach. | Sticky note “Setup Steps” |
| Sample MCP request JSON is provided and defines expected request fields. | Sticky note “Sample Enterprise MCP Request” |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.