Route MCP tool calls through an intelligent gateway with Claude AI

https://n8nworkflows.xyz/workflows/route-mcp-tool-calls-through-an-intelligent-gateway-with-claude-ai-13590


# Route MCP tool calls through an intelligent gateway with Claude AI

This document provides a technical breakdown of the **MCP Intelligent Tool Access Gateway with Claude AI**. This workflow acts as a secure intermediary between AI agents (using the Model Context Protocol) and internal business APIs (CRM, ERP, Databases), providing safety guardrails, auditing, and rate limiting.

---

### 1. Workflow Overview
The workflow transforms standard REST APIs into AI-ready MCP tools. It ensures that every request from an AI agent is validated for schema compliance, checked against a permissions registry, verified for safety/intent by Claude AI, and logged for compliance before and after execution.

**Functional Blocks:**
*   **1.1 Ingestion & Auth:** Receives the MCP JSON payload and validates the API key and structure.
*   **1.2 Tool Registry & AI Verification:** Checks if the tool is registered and uses Claude AI to perform a safety/intent audit on the parameters.
*   **1.3 Governance & Execution:** Enforces rate limits and routes the request to either a "Dry Run" simulator or the live backend API.
*   **1.4 Response Normalization & Auditing:** Standardizes the API response, sends alerts for high-risk calls, and logs the transaction to an immutable audit trail.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Request Ingestion & Auth
This block serves as the entry point, ensuring only authenticated and well-formed requests proceed.
*   **Nodes Involved:** `Receive MCP Tool Request`, `Validate Auth and MCP Schema`.
*   **Logic:**
    *   **Webhook:** Listens for POST requests on the `/mcp-gateway` path.
    *   **Validation (Code):** Hard-validates required fields (`mcpVersion`, `clientId`, `apiKey`). It enforces a naming convention (`category.action`) and checks the `apiKey` against an internal allowlist. It generates a unique `gatewayRequestId` for tracing.
*   **Edge Cases:** Missing fields or unsupported MCP versions trigger an immediate 400-style error.

#### 2.2 Tool Registry + AI Intent Verification
Determines what the tool does and if the AI's current request is safe.
*   **Nodes Involved:** `Lookup Tool in Registry`, `Resolve Tool Config and Permissions`, `Claude AI Intent and Safety Verification`, `Claude AI Model`, `Parse Intent Verification Result`.
*   **Logic:**
    *   **Registry (Google Sheets):** Fetches backend URL, required scopes, and parameter mappings for the requested tool.
    *   **Permissions (Code):** Compares the requested scope (e.g., `read`) against the tool's requirement (e.g., `write`).
    *   **AI Guardrail (Claude):** An Anthropic agent analyzes the `callerContext` and `parameters`. It looks for SQL injection, PII (Personally Identifiable Information), and intent misalignment.
    *   **Verification (Code):** Parses Claude's JSON response. If Claude returns a `CRITICAL` risk or `BLOCK` recommendation, the workflow terminates with a security exception.

#### 2.3 Rate Limit + Backend API Execution
Enforces operational constraints and communicates with the target system.
*   **Nodes Involved:** `Check Rate Limit and Quota`, `Wait For Result`, `Enforce Rate Limit and Build Execution Context`, `Dry Run or Live Execution?`, `Execute Backend API Call`, `Build Dry Run Response`.
*   **Logic:**
    *   **Rate Limiting:** Queries Google Sheets for calls made in the last 60 seconds. If the count exceeds the registry's `rateLimitPerMinute`, the call is throttled.
    *   **Context Building:** Maps AI-provided parameters to the specific keys required by the backend REST API (e.g., renaming `customerId` to `u_id`).
    *   **Conditional Routing:** If the request is marked as `dryRun: true`, it bypasses the live API and returns a mock "would execute" response.
    *   **HTTP Request:** Performs the actual REST call using Bearer Token authentication stored in environment variables.

#### 2.4 Response Normalization, Audit & MCP Return
The final stage standardizes the output and satisfies compliance requirements.
*   **Nodes Involved:** `Normalize Backend Response to MCP Schema`, `Send Security Alert...`, `Write Immutable Audit Log`, `Update Rate Limit Usage Log`, `Build Final MCP Tool Result`, `Return MCP Tool Result to Caller`.
*   **Logic:**
    *   **Normalization:** Converts raw API responses into a consistent MCP Result schema, including success/failure flags and status codes.
    *   **Alerting (SMTP):** If the AI identified a `HIGH` risk but allowed execution (or if the call failed), an email is sent to administrators.
    *   **Logging:** Appends detailed execution data to two Google Sheets: one for an immutable audit trail and one for tracking rate limit quotas.
    *   **Final Output:** Strips internal metadata to return only the strictly defined MCP JSON structure to the caller.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive MCP Tool Request | Webhook | Gateway Entry | (None) | Validate Auth... | ## 1. MCP Request Ingestion & Auth |
| Validate Auth and MCP Schema | Code | Structural Validation | Receive MCP Tool... | Lookup Tool... | ## 1. MCP Request Ingestion & Auth |
| Lookup Tool in Registry | Google Sheets | Config Retrieval | Validate Auth... | Resolve Tool... | ## 2. Tool Registry + AI Intent Verification |
| Resolve Tool Config and Permissions | Code | Access Control | Lookup Tool... | Claude AI Intent... | ## 2. Tool Registry + AI Intent Verification |
| Claude AI Intent...Verification | AI Agent | Safety Guardrail | Resolve Tool... | Parse Intent... | ## 2. Tool Registry + AI Intent Verification |
| Claude AI Model | Anthropic Chat | AI Provider | (Connected to Agent) | (Connected to Agent) | ## 2. Tool Registry + AI Intent Verification |
| Parse Intent Verification Result | Code | Decision Logic | Claude AI Intent... | Check Rate Limit... | ## 2. Tool Registry + AI Intent Verification |
| Check Rate Limit and Quota | Google Sheets | Usage Monitoring | Parse Intent... | Wait For Result | ## 3. Rate Limit + Backend API Execution |
| Wait For Result | Wait | Flow Control | Check Rate Limit... | Enforce Rate... | ## 3. Rate Limit + Backend API Execution |
| Enforce Rate Limit...Context | Code | Prep Execution | Wait For Result | Dry Run or Live...? | ## 3. Rate Limit + Backend API Execution |
| Dry Run or Live Execution? | If | Mode Switch | Enforce Rate... | Execute Backend API; Build Dry Run | ## 3. Rate Limit + Backend API Execution |
| Execute Backend API Call | HTTP Request | System Integration | Dry Run... (False) | Normalize Backend... | ## 3. Rate Limit + Backend API Execution |
| Build Dry Run Response | Code | Mock Provider | Dry Run... (True) | Normalize Backend... | ## 3. Rate Limit + Backend API Execution |
| Normalize Backend Response... | Code | Data Formatting | Execute Backend; Build Dry Run | Alert; Audit Log; Usage Log | ## 4. Response Normalization, Audit & MCP Return |
| Send Security Alert... | Email | Notification | Normalize Backend... | Build Final MCP... | ## 4. Response Normalization, Audit & MCP Return |
| Write Immutable Audit Log | Google Sheets | Compliance | Normalize Backend... | Build Final MCP... | ## 4. Response Normalization, Audit & MCP Return |
| Update Rate Limit Usage Log | Google Sheets | Usage Tracking | Normalize Backend... | Build Final MCP... | ## 4. Response Normalization, Audit & MCP Return |
| Build Final MCP Tool Result | Code | Final Cleanup | Alert; Audit; Usage | Return MCP... | ## 4. Response Normalization, Audit & MCP Return |
| Return MCP Tool Result... | Respond to Webhook | API Response | Build Final MCP... | (None) | ## 4. Response Normalization, Audit & MCP Return |

---

### 4. Reproducing the Workflow from Scratch

1.  **Registry Setup:** Create a Google Sheet with columns: `toolName`, `toolCategory`, `description`, `backendUrl`, `httpMethod`, `requiredScope`, `parameterMapping` (JSON string), `rateLimitPerMinute`.
2.  **Webhook Node:** Create a Webhook node set to `POST` and path `mcp-gateway`. Set Response Mode to `When Last Node Finishes`.
3.  **Authentication Code:** Add a Code node that checks the incoming `headers` or `body` for an `apiKey`. Define a list of accepted keys within the code.
4.  **Google Sheets (Registry):** Add a Google Sheets node to "Get Many Rows". Filter by the `toolName` provided in the Webhook body.
5.  **AI Integration:**
    *   Add an **AI Agent** node. Use the "Define below" prompt type.
    *   Provide the system message instructing it to act as a safety specialist.
    *   Pass the tool definition and parameters into the prompt.
    *   Connect an **Anthropic Chat Model** node (Claude 3.5 Sonnet recommended).
6.  **Safety Logic:** Add a Code node after the agent to `JSON.parse` the agent's response and throw an error if the risk level is too high.
7.  **Rate Limiting:** Use a Google Sheets node to count entries in a "Usage Log" sheet where `clientId` matches and `timestamp` is within the last minute. Add a Code node to compare this count against the registry's limit.
8.  **Execution Switch:** Add an **If Node** checking the boolean `dryRun`.
    *   **True branch:** A Code node returning a JSON object describing what *would* have happened.
    *   **False branch:** An **HTTP Request** node. Use expressions to set the URL, Method, and Body based on the Registry config and the mapped parameters.
9.  **Post-Processing:**
    *   Add a Code node to unify the response from either branch into the MCP standard (success, result, error).
    *   Add **Google Sheets** (Append) nodes to log the execution to the Audit and Usage sheets.
    *   Add an **Email** node with a conditional expression to send alerts only if `riskLevel` is HIGH.
10. **Final Response:** Use the **Respond to Webhook** node. Return the standardized result and set custom headers for `X-MCP-Gateway-Request-ID`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Model Context Protocol (MCP)** | Structured standard for AI-to-API communication. |
| **Security Best Practice** | The `apiKey` validation in this workflow is illustrative; for production, use a Database or Vault. |
| **Immutability** | The Audit Log sheet should have restricted "Append Only" permissions for service accounts. |
| **Parameter Mapping** | The workflow supports JSON-based mapping to bridge AI naming (camelCase) to legacy API naming (snake_case). |