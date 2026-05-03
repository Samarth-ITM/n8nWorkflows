Triage tickets by ranking next actions with VectorPrime via webhook

https://n8nworkflows.xyz/workflows/triage-tickets-by-ranking-next-actions-with-vectorprime-via-webhook-12962


# Triage tickets by ranking next actions with VectorPrime via webhook

# Reference Document: Triage Tickets by Ranking Next Actions with VectorPrime

This document provides a technical breakdown of the n8n workflow designed to triage tickets or incidents by ranking potential "next actions" using the VectorPrime API.

---

### 1. Workflow Overview
The workflow functions as an automated decision-support system. It receives ticket data via a webhook, validates and normalizes the input, consults an external ranking engine (VectorPrime) to determine the best course of action based on a prompt and multiple options, and finally generates an audit-ready response.

**Logical Blocks:**
*   **1.1 Input Reception & Normalization:** Receives the raw JSON and ensures the data structure meets the ranking engine's requirements.
*   **1.2 VectorPrime Integration:** Communicates with the external API to perform the ranking logic.
*   **1.3 Validation & Routing:** Interprets the API response to decide between success or error paths.
*   **1.4 Post-Processing & Output:** Constructs optional payloads (Slack, Email, Backlog) and provides a final HTTP response to the caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block captures the incoming request and ensures that the "options" (the actions to be ranked) are formatted correctly as an array of objects.

*   **Nodes Involved:** `Webhook1`, `Normalize + Build Options`, `Normalize OK?`.
*   **Node Details:**
    *   **Webhook1 (Webhook):** Listens for POST requests. Configured to respond via a specific response node (`Respond Success` or `Respond Error`).
    *   **Normalize + Build Options (Code):** Uses JavaScript to clean the input. It maps incoming fields like `title` or `message` to a standard `prompt` and ensures `options` is an array of `{id, label}`.
        *   *Edge Case:* If fewer than 2 options are provided, it returns an error object to trigger the failure path.
    *   **Normalize OK? (If):** Checks the boolean `ok` property from the normalization step. If false, it routes immediately to the error response.

#### 2.2 VectorPrime Integration
This block handles the technical communication with the VectorPrime ranking kernel.

*   **Nodes Involved:** `VectorPrime Rank (HTTP)`.
*   **Node Details:**
    *   **VectorPrime Rank (HTTP) (HTTP Request):** Sends a POST request to the VectorPrime backend.
    *   **Configuration:** 
        *   URL: `https://vectorprime-kernel-backend.onrender.com/v1/kernel/rank`
        *   Authentication: Uses Header Auth (`Authorization: Bearer vp_...`).
        *   Body: Sends the entire `vp_payload` as a JSON string.
    *   **Constraint:** `Continue on Fail` is enabled to allow the workflow to handle API errors gracefully in the next step.

#### 2.3 Validation & Routing
Processes the raw HTTP response to determine if the ranking was successful.

*   **Nodes Involved:** `Parse VP Result`, `VP OK?`.
*   **Node Details:**
    *   **Parse VP Result (Code):** Normalizes the various ways the HTTP node can return data (body vs. status code). It checks if the response contains a valid `ranking` array.
    *   **VP OK? (If):** Routes the flow based on whether the `ok` variable is true.

#### 2.4 Post-Processing & Output
Prepares the data for storage or immediate action and sends the final response to the webhook initiator.

*   **Nodes Involved:** `Slack Alert Payload`, `Backlog Record`, `Email Draft`, `Create Audit Log`, `Edit Fields`, `Respond Success`, `Respond Error`.
*   **Node Details:**
    *   **Payload Builders (Set):** The nodes `Slack Alert Payload`, `Backlog Record`, and `Email Draft` are placeholders (currently empty `Set` nodes) meant for users to map the ranked results to their specific tool schemas.
    *   **Create Audit Log (Code):** Generates a timestamped object containing the original request, the ranking results, and the top probability for record-keeping.
    *   **Edit Fields (Set):** Aggregates the final JSON structure for the successful response.
    *   **Respond Success/Error (Respond to Webhook):** Returns the final JSON and HTTP status codes (200 for success, 400 for errors) to the client.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook1 | Webhook | Entry Point | (None) | Normalize + Build Options | Receives POST JSON with: decision_id, prompt, options |
| Normalize + Build Options | Code | Data Validation | Webhook1 | Normalize OK? | Validates input and builds `vp_payload`. Returns error JSON if options are missing/invalid. |
| Normalize OK? | If | Logic Gate | Normalize + Build Options | VectorPrime Rank (HTTP), Respond Error | (No specific note) |
| VectorPrime Rank (HTTP) | HTTP Request | API Integration | Normalize OK? | Parse VP Result | Calls the ranking endpoint using your Header Auth credential. Body must be JSON so options stay an array. |
| Parse VP Result | Code | Data Parsing | VectorPrime Rank (HTTP) | VP OK? | Parses response and routes: ok=true → success path, ok=false → error response |
| VP OK? | If | Logic Gate | Parse VP Result | Slack Alert Payload, Respond Error | Parses response and routes: ok=true → success path, ok=false → error response |
| Slack Alert Payload | Set | Transformation | VP OK? | Backlog Record | Builds optional payload drafts, writes audit log, and responds with JSON. |
| Backlog Record | Set | Transformation | Slack Alert Payload | Email Draft | Builds optional payload drafts, writes audit log, and responds with JSON. |
| Email Draft | Set | Transformation | Backlog Record | Create Audit Log | Builds optional payload drafts, writes audit log, and responds with JSON. |
| Create Audit Log | Code | Audit/Logging | Email Draft | Edit Fields | Builds optional payload drafts, writes audit log, and responds with JSON. |
| Edit Fields | Set | Final Formatting | Create Audit Log | Respond Success | Builds optional payload drafts, writes audit log, and responds with JSON. |
| Respond Success | Respond to Webhook | API Response | Edit Fields | (None) | (No specific note) |
| Respond Error | Respond to Webhook | API Response | Normalize OK?, VP OK? | (None) | (No specific note) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:** Create a Webhook node. Set HTTP Method to `POST` and Response Mode to `When Last Node Finishes` (or use the `Respond to Webhook` nodes as configured).
2.  **Normalization Logic:** Add a Code node. Implement logic to extract `decision_id`, `prompt`, and `options`. Ensure `options` is an array of objects. Return `ok: true` or `ok: false`.
3.  **Input Gate:** Add an If node to check `ok === true`.
4.  **Credential Setup:** 
    *   Go to n8n Credentials. Create a **Header Auth** credential.
    *   Name: `Authorization`. Value: `Bearer [YOUR_VECTORPRIME_KEY]`.
5.  **API Call:** Add an HTTP Request node. 
    *   Method: `POST`.
    *   URL: `https://vectorprime-kernel-backend.onrender.com/v1/kernel/rank`.
    *   Authentication: Select the Header Auth credential created above.
    *   Body: Use "JSON" and pass the payload constructed in step 2.
    *   **Crucial:** Do not use "Fields Below" to send options, as this may flatten the array and cause a 422 error.
6.  **Parsing:** Add a Code node to check if the HTTP response contains a `ranking` key.
7.  **Routing:** Add an If node to check if the parsing was successful.
8.  **Payload & Logging:** (Optional) Add Set nodes for Slack, Email, or Database schemas. Add a Code node to generate a ISO timestamp (`new Date().toISOString()`) for auditing.
9.  **Responses:** Add two "Respond to Webhook" nodes. 
    *   One for success (Status 200) linked after the audit log.
    *   One for error (Status 400) linked to the "False" outputs of the If nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create an API key for the ranking kernel. | [VectorPrime Tech](https://vectorprime.tech) |
| Warning: Send the body as JSON so options remain an array to prevent 422 errors. | Node: VectorPrime Rank (HTTP) |
| Workflow is designed to handle both direct VectorPrime payloads or simple generic webhook payloads. | Logic in "Normalize" node |