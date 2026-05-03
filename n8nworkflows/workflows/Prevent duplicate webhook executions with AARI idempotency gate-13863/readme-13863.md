Prevent duplicate webhook executions with AARI idempotency gate

https://n8nworkflows.xyz/workflows/prevent-duplicate-webhook-executions-with-aari-idempotency-gate-13863


# Prevent duplicate webhook executions with AARI idempotency gate

# Workflow Reference: Prevent Duplicate Webhook Executions with AARI Idempotency Gate

This document provides a technical breakdown of an n8n workflow designed to ensure idempotency in webhook-driven processes. By using AARI as an external state manager, the workflow prevents duplicate "side effects" (like payments, emails, or database writes) even if the source provider sends the same webhook multiple times.

---

### 1. Workflow Overview

The primary goal of this workflow is to protect downstream systems from redundant executions caused by "at-least-once" delivery common in platforms like Stripe, GitHub, or Shopify. It functions as a gateway that checks an incoming event's unique fingerprint against a 24-hour history.

**Logical Blocks:**
- **1.1 Input Reception:** Captures the incoming webhook from an external service.
- **1.2 Idempotency Check:** Communicates with the AARI API to determine if this specific event has been processed recently.
- **1.3 Flow Control:** Routes the execution based on whether the event is "ALLOWED" (new) or "BLOCK" (duplicate).
- **1.4 Side Effect Execution:** Placeholder for the actual business logic (e.g., sending an email).
- **1.5 Status Reporting:** Informs AARI that the action was successfully completed to close the idempotency loop.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception
- **Overview:** Acts as the entry point for external data.
- **Nodes Involved:** `Incoming webhook event`
- **Node Details:**
    - **Node Name:** Incoming webhook event
    - **Type:** Webhook Node (v2)
    - **Configuration:** Set to `POST` method. Response mode is `onReceived` (immediately acknowledges the sender).
    - **Edge Cases:** If the sender requires a specific response body (other than n8n's default), the response mode might need adjustment.

#### 1.2 Idempotency Check
- **Overview:** Generates a fingerprint and asks AARI for a decision.
- **Nodes Involved:** `AARI idempotency gate`
- **Node Details:**
    - **Node Name:** AARI idempotency gate
    - **Type:** HTTP Request (v4.1)
    - **Key Expressions:** 
        - `idempotency_key`: A complex fallback logic searching for IDs in `body.id`, `body.event_id`, `headers.x-event-id`, or a concatenated string of event metadata.
        - `agent_id`: Uses `{{ $workflow.name }}`.
    - **Connection:** Outgoing to "Duplicate detected?".
    - **Potential Failures:** Invalid API Key (`401`), Timeout (set to 10s), or Schema mismatch in the incoming webhook preventing key generation.

#### 1.3 Flow Control
- **Overview:** Evaluates the JSON response from AARI to decide whether to continue or stop.
- **Nodes Involved:** `Duplicate detected?`, `⛔ Stop duplicate execution`
- **Node Details:**
    - **Node Name:** Duplicate detected?
    - **Type:** If Node (v2)
    - **Logic:** Checks if `{{ $json.decision }}` strictly equals `BLOCK`.
    - **Node Name:** ⛔ Stop duplicate execution
    - **Type:** Set Node (v3.3)
    - **Role:** Terminal node for duplicates; assigns `status: duplicate_blocked`.

#### 1.4 Side Effect Execution
- **Overview:** The "safe zone" where the actual work happens.
- **Nodes Involved:** `✅ Run your action here`
- **Node Details:**
    - **Node Name:** ✅ Run your action here
    - **Type:** Set Node (v3.3)
    - **Technical Role:** Currently a placeholder for mapping status. In a production environment, this should be followed or replaced by nodes like "Send Email" or "Stripe Charge".

#### 1.5 Status Reporting
- **Overview:** Finalizes the lifecycle of the action in AARI's records.
- **Nodes Involved:** `📋 Report SUCCESS`
- **Node Details:**
    - **Node Name:** 📋 Report SUCCESS
    - **Type:** HTTP Request (v4.1)
    - **Configuration:** `POST` to `https://api.getaari.com/actions/{{ $json.action_id }}/complete`.
    - **Payload:** `{"outcome": "SUCCESS"}`.
    - **Role:** Confirms that the side effect was performed successfully so that subsequent retries are accurately blocked.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Incoming webhook event | Webhook | Entry Point | None | AARI idempotency gate | |
| AARI idempotency gate | HTTP Request | Fraud/Duplicate Check | Incoming webhook event | Duplicate detected? | Setup (2 minutes) |
| Duplicate detected? | If | Logic Routing | AARI idempotency gate | ⛔ Stop duplicate execution, ✅ Run your action here | Test it: Send the same webhook payload twice. |
| ⛔ Stop duplicate execution | Set | Error Handling | Duplicate detected? | None | Test it: Second call -> decision: BLOCK |
| ✅ Run your action here | Set | Action Execution | Duplicate detected? | 📋 Report SUCCESS | Setup (2 minutes): Replace with real node |
| 📋 Report SUCCESS | HTTP Request | Completion Signal | ✅ Run your action here | None | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Obtain an AARI API Key from [api.getaari.com/n8n](https://api.getaari.com/n8n).
    *   In n8n, go to **Credentials** > **New**, select **Header Auth**. 
    *   Name: `AARI API Key`. Header Name: `X-API-Key`. Value: [Your Key].

2.  **Create Webhook:**
    *   Add a **Webhook** node. Set path to `webhook-dedup` and HTTP Method to `POST`.

3.  **Setup the Gate:**
    *   Add an **HTTP Request** node named "AARI idempotency gate".
    *   Method: `POST`. URL: `https://api.getaari.com/gate`.
    *   Authentication: Select the "AARI API Key" credential.
    *   Body Parameters (Name/Value):
        *   `agent_id`: `{{ $workflow.name }}`
        *   `run_id`: `{{ $execution.id }}`
        *   `action_type`: `workflow.side_effect`
        *   `idempotency_key`: (Use the logic to extract IDs from `$json.body` or `$json.headers`).
        *   `mode`: `advisory`

4.  **Add Logic Filter:**
    *   Add an **If** node. Condition: String `{{ $json.decision }}` equals `BLOCK`.
    *   Connect the "True" branch to a **Set** node indicating a blocked execution.
    *   Connect the "False" branch to your desired action node.

5.  **Finalize Completion:**
    *   After your action node, add an **HTTP Request** node named "Report SUCCESS".
    *   Method: `POST`. URL: `https://api.getaari.com/actions/{{ $json.action_id }}/complete`.
    *   Body: JSON `{"outcome": "SUCCESS"}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Fingerprint Fallback Order** | 1. body.id, 2. headers, 3. concatenated object properties, 4. execution.id |
| **AARI Registration** | [https://api.getaari.com/n8n](https://api.getaari.com/n8n) |
| **Usage Limits** | Free tier includes 2,500 gate calls/month without a credit card. |
| **Retention** | Duplicate checks are performed against a 24-hour window. |