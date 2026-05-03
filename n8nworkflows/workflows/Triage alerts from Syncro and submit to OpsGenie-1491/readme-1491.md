Triage alerts from Syncro and submit to OpsGenie

https://n8nworkflows.xyz/workflows/triage-alerts-from-syncro-and-submit-to-opsgenie-1491


# Triage alerts from Syncro and submit to OpsGenie

### 1. Workflow Overview

This workflow processes alerts received from Syncro via webhook and submits them to OpsGenie for alert management. Its primary use case is to handle "agent_offline_trigger" alerts, determining whether an alert is new or resolved, and then creating or closing the corresponding alert in OpsGenie accordingly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receive and parse Syncro webhook alerts.
- **1.2 Alert Filtering:** Identify if the alert is of type "agent_offline_trigger".
- **1.3 Data Preparation:** Extract and format necessary data for OpsGenie.
- **1.4 Alert State Decision:** Determine if the alert is new or resolved.
- **1.5 OpsGenie Integration:** Create new alerts or close existing ones in OpsGenie.
- **1.6 No Operation Handling:** Bypass non-relevant alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives incoming alerts from Syncro via a webhook POST request and passes the raw JSON payload downstream.

- **Nodes Involved:**  
Webhook

- **Node Details:**

  - **Webhook**  
    - **Type & Role:** n8n webhook node; entry point for Syncro alert data.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `/fromsyncro`  
      - Responds with all entries, using the last node's output.  
    - **Key Expressions:** None (raw data intake)  
    - **Input Connections:** None (trigger node)  
    - **Output Connections:** Connected to Switch node  
    - **Version Requirements:** Standard webhook, no special version needs  
    - **Potential Failures:** Network issues, malformed requests, webhook misconfiguration in Syncro  
    - **Notes:** This node must be publicly accessible and configured in Syncro as the webhook URL.

---

#### 2.2 Alert Filtering

- **Overview:**  
Filters incoming alerts to continue only if the alert's trigger type is "agent_offline_trigger". Other alerts are bypassed.

- **Nodes Involved:**  
Switch, NoOp

- **Node Details:**

  - **Switch**  
    - **Type & Role:** Decision node; filters alert by trigger type.  
    - **Configuration:**  
      - Evaluates `{{$node["Webhook"].json["body"]["attributes"]["properties"]["trigger"]}}`  
      - Continues if equals `"agent_offline_trigger"`  
    - **Input Connections:** From Webhook  
    - **Output Connections:**  
      - True branch → Set node  
      - False branch → NoOp node  
    - **Potential Failures:** Expression errors if JSON structure changes or missing properties  
    - **Edge Cases:** Alerts without `properties` or `trigger` field might cause errors or be incorrectly bypassed.

  - **NoOp**  
    - **Type & Role:** Placeholder node; used to end execution gracefully for non-relevant alerts.  
    - **Configuration:** None  
    - **Input Connections:** From Switch (false branch)  
    - **Output Connections:** None  
    - **Potential Failures:** None  

---

#### 2.3 Data Preparation

- **Overview:**  
Extracts specific alert data from the webhook payload and formats it into simplified variables for easier use downstream.

- **Nodes Involved:**  
Set

- **Node Details:**

  - **Set**  
    - **Type & Role:** Data transformation node; creates simplified fields for alert ID and description.  
    - **Configuration:**  
      - Sets two string fields:  
        - `AlertID` = `{{$node["Webhook"].json["body"]["attributes"]["id"]}}`  
        - `Description` = `{{$node["Webhook"].json["body"]["attributes"]["computer_name"]}} ({{$node["Webhook"].json["body"]["attributes"]["customer"]["business_then_name"]}}): {{$node["Webhook"].json["body"]["attributes"]["formatted_output"]}}`  
      - Keeps only these set fields (clears other data)  
    - **Input Connections:** From Switch (true branch)  
    - **Output Connections:** To IF node  
    - **Potential Failures:** Expression failures if expected properties missing or renamed.  
    - **Edge Cases:** Customer object or business name missing could lead to empty or malformed description.

---

#### 2.4 Alert State Decision

- **Overview:**  
Determines whether the received alert is marked as resolved or new by checking the `resolved` boolean flag.

- **Nodes Involved:**  
IF

- **Node Details:**

  - **IF**  
    - **Type & Role:** Conditional branching based on alert resolution state.  
    - **Configuration:**  
      - Checks if `{{$node["Webhook"].json["body"]["attributes"]["resolved"]}}` is true (boolean condition)  
    - **Input Connections:** From Set  
    - **Output Connections:**  
      - True (resolved) → Close Alert node  
      - False (new alert) → Create Alert node  
    - **Potential Failures:** Expression failure if `resolved` property is absent or not boolean.  
    - **Edge Cases:** Alerts without `resolved` field may default to false branch; careful validation recommended.

---

#### 2.5 OpsGenie Integration

- **Overview:**  
Based on the alert state, either creates a new OpsGenie alert or closes an existing alert using OpsGenie's REST API.

- **Nodes Involved:**  
Create Alert, Close Alert

- **Node Details:**

  - **Create Alert**  
    - **Type & Role:** HTTP Request node; creates a new alert in OpsGenie.  
    - **Configuration:**  
      - URL: `https://api.opsgenie.com/v2/alerts`  
      - Method: POST  
      - Authentication: HTTP Header Auth (OpsGenie API Key)  
      - Body Parameters:  
        - `message`: formatted as `<computer_name> (<customer_business_name>): <formatted_output>`  
        - `alias`: unique alert ID from Syncro (`id` attribute)  
        - `description`: includes alert text and link from webhook payload  
    - **Input Connections:** From IF (false branch - new alert)  
    - **Output Connections:** None  
    - **Potential Failures:**  
      - Authentication errors (invalid/missing API key)  
      - API rate limits or downtime  
      - Invalid or missing parameters causing rejection  
    - **Edge Cases:** Duplicate alias submission may affect alert deduplication in OpsGenie.

  - **Close Alert**  
    - **Type & Role:** HTTP Request node; closes an alert in OpsGenie by alias.  
    - **Configuration:**  
      - URL: `https://api.opsgenie.com/v2/alerts/<alert_id>/close?identifierType=alias`  
        - `<alert_id>` replaced by Syncro alert ID  
      - Method: POST  
      - Authentication: HTTP Header Auth (OpsGenie API Key)  
      - Body Parameters:  
        - `note`: "Issue resolved automatically according to Syncro."  
    - **Input Connections:** From IF (true branch - resolved alert)  
    - **Output Connections:** None  
    - **Potential Failures:**  
      - Auth errors  
      - Alert ID not found or already closed  
      - API downtime or rate limits  
    - **Edge Cases:** Closing an alert that never existed in OpsGenie or race conditions on alert state.

---

#### 2.6 No Operation Handling

- **Overview:**  
Handles all alerts that do not match the "agent_offline_trigger" trigger type, effectively ending workflow execution without action.

- **Nodes Involved:**  
NoOp (already detailed in 2.2)

---

### 3. Summary Table

| Node Name   | Node Type           | Functional Role                             | Input Node(s) | Output Node(s)    | Sticky Note                                                                                                         |
|-------------|---------------------|--------------------------------------------|---------------|-------------------|---------------------------------------------------------------------------------------------------------------------|
| Webhook     | Webhook             | Entry point; receives Syncro alerts via POST | None          | Switch            | Must be publicly accessible URL; configure Syncro webhook to this path.                                            |
| Switch      | Switch              | Filters alerts by trigger type              | Webhook       | Set, NoOp         | Filters only "agent_offline_trigger" alerts; others bypassed gracefully.                                            |
| NoOp        | No Operation        | Ends workflow for non-relevant alerts       | Switch        | None              | No action taken for alerts not matching the trigger type.                                                          |
| Set         | Set                 | Extracts and formats alert data for OpsGenie | Switch        | IF                | Prepares AlertID and Description fields from webhook JSON.                                                         |
| IF          | If                  | Routes based on alert resolution state      | Set           | Create Alert, Close Alert | Checks if alert is resolved to decide create or close action.                                                        |
| Create Alert| HTTP Request        | Creates a new alert in OpsGenie              | IF            | None              | API integration required in OpsGenie; uses alias for alert deduplication.                                          |
| Close Alert | HTTP Request        | Closes an existing alert in OpsGenie         | IF            | None              | Closes alert by alias; note added to indicate automatic resolution from Syncro.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**
   - Type: Webhook  
   - Name: "Webhook"  
   - HTTP Method: POST  
   - Path: `fromsyncro`  
   - Response Data: All Entries  
   - Response Mode: Last Node  
   - Save and expose URL; configure Syncro notifications to send alerts here.

2. **Create Switch Node:**
   - Type: Switch  
   - Name: "Switch"  
   - Value to test: Expression `{{$node["Webhook"].json["body"]["attributes"]["properties"]["trigger"]}}`  
   - Data Type: String  
   - Add Rule: Equals `agent_offline_trigger`  
   - Connect Webhook output to Switch input.

3. **Create NoOp Node:**
   - Type: No Operation  
   - Name: "NoOp"  
   - Connect Switch's false output to NoOp input.

4. **Create Set Node:**
   - Type: Set  
   - Name: "Set"  
   - Add two string parameters:  
     - `AlertID` = `{{$node["Webhook"].json["body"]["attributes"]["id"]}}`  
     - `Description` = `{{$node["Webhook"].json["body"]["attributes"]["computer_name"]}} ({{$node["Webhook"].json["body"]["attributes"]["customer"]["business_then_name"]}}): {{$node["Webhook"].json["body"]["attributes"]["formatted_output"]}}`  
   - Enable "Keep Only Set" values  
   - Connect Switch's true output to Set's input.

5. **Create IF Node:**
   - Type: If  
   - Name: "IF"  
   - Condition: Boolean check  
     - Value 1: `{{$node["Webhook"].json["body"]["attributes"]["resolved"]}}`  
     - Condition: Equals `true`  
   - Connect Set output to IF input.

6. **Create Create Alert Node:**
   - Type: HTTP Request  
   - Name: "Create Alert"  
   - HTTP Method: POST  
   - URL: `https://api.opsgenie.com/v2/alerts`  
   - Authentication: HTTP Header Auth (create and select OpsGenie API key credentials)  
   - Body Parameters (JSON):  
     - `message`: `{{$node["Webhook"].json["body"]["attributes"]["computer_name"]}} ({{$node["Webhook"].json["body"]["attributes"]["customer"]["business_then_name"]}}): {{$node["Webhook"].json["body"]["attributes"]["formatted_output"]}}`  
     - `alias`: `{{$node["Webhook"].json["body"]["attributes"]["id"]}}`  
     - `description`: `{{$node["Webhook"].json["body"]["text"]}}\n{{$node["Webhook"].json["body"]["link"]}}`  
   - Connect IF node's false (not resolved) output to Create Alert's input.

7. **Create Close Alert Node:**
   - Type: HTTP Request  
   - Name: "Close Alert"  
   - HTTP Method: POST  
   - URL: `https://api.opsgenie.com/v2/alerts/{{$node["Webhook"].json["body"]["attributes"]["id"]}}/close?identifierType=alias`  
   - Authentication: HTTP Header Auth (use same OpsGenie API credentials)  
   - Body Parameters (JSON):  
     - `note`: "Issue resolved automatically according to Syncro."  
   - Connect IF node's true (resolved) output to Close Alert's input.

8. **Final Checks:**
   - Confirm credential setup for OpsGenie API key in n8n credentials manager.  
   - Verify webhook URL is reachable and configured in Syncro.  
   - Test with sample Syncro alerts of type "agent_offline_trigger" for both new and resolved states.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow is part of an MSP collection, originally published by bionemesis.                                      | https://github.com/bionemesis/n8nsyncro                          |
| OpsGenie allows alert deduplication via `alias`, removing the need for external state storage (e.g., Google Sheets).| OpsGenie API documentation                                       |
| In Syncro, automated remediation scripts are required to close alerts by triggering when the asset comes back online.| Workflow description instructions                                 |

---

This structured documentation fully describes the "Syncro Alert to OpsGenie" workflow, enabling reproduction, modification, and error anticipation.