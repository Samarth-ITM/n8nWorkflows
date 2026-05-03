Export new deals from HubSpot to Slack and Airtable

https://n8nworkflows.xyz/workflows/export-new-deals-from-hubspot-to-slack-and-airtable-1225


# Export new deals from HubSpot to Slack and Airtable

### 1. Workflow Overview

This workflow automates the processing of newly created deals in HubSpot by branching logic based on deal stage, type, and value to notify teams, create presentations, log lost deals, or create support tickets. It integrates HubSpot, Slack, Airtable, and Google Slides to streamline deal management and follow-up actions.

Logical blocks:

- **1.1 Input Reception and Deal Data Retrieval**  
  Trigger on new HubSpot deal creation, then fetch detailed deal information.

- **1.2 Deal Data Preparation**  
  Extract and prepare key deal properties as usable variables.

- **1.3 Deal Stage Branching**  
  Branch into three paths based on deal stage: closed-won, presentation scheduled, or closed-lost.

- **1.4 Deal Stage Actions**  
  - Closed-won: Notify team via Slack.  
  - Presentation scheduled: Create Google Slides presentation.  
  - Closed-lost: Append deal info to Airtable for analysis.

- **1.5 Deal Type and Value Branching**  
  Further branch based on deal type (new or existing business) and deal value to create HubSpot tickets with appropriate priority.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Deal Data Retrieval

- **Overview:**  
  This block listens for new deal creation events in HubSpot and retrieves detailed deal data for processing.

- **Nodes Involved:**  
  - Hubspot Trigger  
  - Hubspot

- **Node Details:**

  - **Hubspot Trigger**  
    - Type: Trigger node, listens to HubSpot events.  
    - Config: Triggered on `deal.creation` event, webhook enabled with ID `12345`.  
    - Inputs: External event trigger.  
    - Outputs: Emits dealId and minimal deal data upon new deal creation.  
    - Edge Cases: Webhook failures, event delays, or missing permissions can prevent triggering.

  - **Hubspot**  
    - Type: HubSpot API node, fetch operation.  
    - Config: Gets full deal details based on `dealId` from trigger.  
    - Inputs: From Hubspot Trigger.  
    - Outputs: Full deal properties JSON.  
    - Edge Cases: API rate limits, invalid dealId, or connectivity issues.

#### 2.2 Deal Data Preparation

- **Overview:**  
  Extracts specific deal attributes such as value, name, description, stage, and type into individual variables for easier access downstream.

- **Nodes Involved:**  
  - Set

- **Node Details:**

  - **Set**  
    - Type: Data manipulation node.  
    - Config: Maps nested JSON properties from HubSpot deal data to flat variables: `deal_value`, `deal_id`, `deal_name`, `deal_date`, `deal_description`, `deal_type`, `deal_stage`.  
    - Inputs: From Hubspot node.  
    - Outputs: Single JSON item with prepared variables.  
    - Edge Cases: Missing or null properties can lead to undefined variables; expressions assume consistent data structure.

#### 2.3 Deal Stage Branching

- **Overview:**  
  Routes workflow execution based on deal stage into three cases: closed won, presentation scheduled, and closed lost.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - Type: Conditional branching node.  
    - Config: Evaluates `dealstage` value from HubSpot deal data to route to one of three outputs:  
      - Output 1: `closedwon`  
      - Output 2: `presentationscheduled`  
      - Output 3: `closedlost`  
    - Inputs: From Set node (variables prepared).  
    - Outputs: One of three paths.  
    - Edge Cases: Unexpected dealstage values cause no branch to match; consider default fallback.

#### 2.4 Deal Stage Actions

- **Overview:**  
  Executes specific actions depending on deal stage:

  - Closed won: Send Slack notification.  
  - Presentation scheduled: Create Google Slides presentation.  
  - Closed lost: Append deal info to Airtable.

- **Nodes Involved:**  
  - #closedwon (Slack)  
  - Google Slides  
  - Airtable

- **Node Details:**

  - **#closedwon (Slack)**  
    - Type: Slack node, sends message.  
    - Config: Posts a celebratory message containing the deal name to Slack channel `deals`.  
    - Inputs: From Switch node output 1.  
    - Outputs: None (terminal node).  
    - Credentials: Slack OAuth token configured.  
    - Edge Cases: Slack API errors, invalid channel, or auth failure.

  - **Google Slides**  
    - Type: Google Slides node, creates presentation.  
    - Config: Creates a new presentation titled "Presentation for deal [deal_name]".  
    - Inputs: From Switch node output 2.  
    - Credentials: OAuth2 authentication with Google Slides API.  
    - Edge Cases: OAuth token expiration, API rate limits, or permission issues.

  - **Airtable**  
    - Type: Airtable node, appends record.  
    - Config: Appends a new record to `lost_deals` table with fields `deal_name`, `deal_id`, and `deal_type`.  
    - Inputs: From Switch node output 3.  
    - Credentials: Airtable API key configured.  
    - Edge Cases: API limits, missing fields, or table misconfiguration.

#### 2.5 Deal Type and Value Branching

- **Overview:**  
  Further branches based on deal type (`newbusiness` or existing) and deal value relative to 500 to create prioritized tickets in HubSpot.

- **Nodes Involved:**  
  - IF  
  - high-priority  
  - low-priority

- **Node Details:**

  - **IF**  
    - Type: Conditional node.  
    - Config: Checks two conditions concurrently:  
      - Deal value greater than 500  
      - Deal type equals `newbusiness`  
      - Deal stage not equal to `closedlost` or `closedwon` (negated condition)  
    - Outputs:  
      - True: High priority ticket creation  
      - False: Low priority ticket creation  
    - Inputs: From Set node (variables).  
    - Outputs: Two paths based on conditions.  
    - Edge Cases: Logical expression errors or missing variables.

  - **high-priority (HubSpot ticket)**  
    - Type: HubSpot API node, creates ticket.  
    - Config: Creates a ticket in pipeline 0, stage 1, with priority HIGH, assigned to user ID 12345. Ticket name includes deal name, description included.  
    - Inputs: From IF node (true).  
    - Credentials: HubSpot API credentials.  
    - Edge Cases: Invalid user ID, API errors, or permissions issues.

  - **low-priority (HubSpot ticket)**  
    - Type: HubSpot API node, creates ticket.  
    - Config: Creates a ticket in pipeline 0, stage 1, with priority MEDIUM, ticket name includes deal name and description. No owner assigned.  
    - Inputs: From IF node (false).  
    - Credentials: HubSpot API credentials.  
    - Edge Cases: API errors, missing data.

---

### 3. Summary Table

| Node Name      | Node Type               | Functional Role                                  | Input Node(s)       | Output Node(s)                    | Sticky Note                                                                                 |
|----------------|-------------------------|-------------------------------------------------|---------------------|---------------------------------|---------------------------------------------------------------------------------------------|
| Hubspot Trigger| HubSpot Trigger         | Initiates workflow on new deal creation         | External event      | Hubspot                         |                                                                                             |
| Hubspot        | HubSpot API             | Retrieves full deal data from HubSpot            | Hubspot Trigger     | Set                             |                                                                                             |
| Set            | Set node                | Prepares deal variables from raw JSON data       | Hubspot             | Switch, IF                      |                                                                                             |
| Switch         | Switch node             | Branches by deal stage                            | Set                 | #closedwon, Google Slides, Airtable |                                                                                             |
| #closedwon     | Slack                   | Sends Slack notification for closed won deals   | Switch              | None                           |                                                                                             |
| Google Slides  | Google Slides API       | Creates presentation for scheduled presentations | Switch              | None                           |                                                                                             |
| Airtable       | Airtable                | Appends lost deal data for analysis               | Switch              | None                           |                                                                                             |
| IF             | If node                 | Branches by deal value and type for ticket priority | Set                 | high-priority, low-priority     |                                                                                             |
| high-priority  | HubSpot API             | Creates high priority ticket for new business deals over 500 | IF                  | None                           |                                                                                             |
| low-priority   | HubSpot API             | Creates low priority ticket for other deals      | IF                  | None                           |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a HubSpot Trigger node**  
   - Type: HubSpot Trigger  
   - Event: `deal.creation`  
   - Configure webhook with unique ID (e.g., `12345`)  
   - No credentials needed beyond HubSpot OAuth setup.

2. **Add a HubSpot node**  
   - Type: HubSpot API  
   - Operation: `get`  
   - Resource: `deal`  
   - Parameter: `dealId` set to `{{$json["dealId"]}}` from trigger  
   - Connect output of HubSpot Trigger to input of this node  
   - Use HubSpot API credentials.

3. **Add a Set node**  
   - Extract and map deal details into variables:  
     - `deal_value`: `{{$json["properties"]["amount"]["value"]}}` (number)  
     - `deal_id`: `{{$json["dealId"]}}` (number)  
     - `deal_name`: `{{$json["properties"]["dealname"]["value"]}}` (string)  
     - `deal_date`: `{{$json["properties"]["closedate"]["timestamp"]}}` (string)  
     - `deal_description`: `{{$json["properties"]["description"]["value"]}}` (string)  
     - `deal_type`: `{{$json["properties"]["dealtype"]["value"]}}` (string)  
     - `deal_stage`: `{{$json["properties"]["dealstage"]["value"]}}` (string)  
   - Connect HubSpot node output to Set node input.

4. **Add a Switch node**  
   - Set "Value to Check": `{{$node["Hubspot"].json["properties"]["dealstage"]["value"]}}`  
   - Add rules for exact matches:  
     - `closedwon` → output 1  
     - `presentationscheduled` → output 2  
     - `closedlost` → output 3  
   - Connect Set node output to Switch node input.

5. **Add Slack node (#closedwon)**  
   - Type: Slack  
   - Channel: `deals`  
   - Text: `We successfully closed the deal {{$node["Set"].json["deal_name"]}}!`  
   - Connect Switch output 1 to Slack node input.  
   - Configure Slack API credentials.

6. **Add Google Slides node**  
   - Type: Google Slides  
   - Title: `Presentation for deal {{$node["Set"].json["deal_name"]}}`  
   - Authentication: OAuth2 for Google Slides API  
   - Connect Switch output 2 to this node input.

7. **Add Airtable node**  
   - Type: Airtable  
   - Operation: Append  
   - Table: `lost_deals`  
   - Fields: `deal_name`, `deal_id`, `deal_type`  
   - Map fields to corresponding variables from Set node  
   - Connect Switch output 3 to Airtable node input  
   - Configure Airtable API credentials.

8. **Add IF node**  
   - Conditions (AND):  
     - Number: `{{$json["deal_value"]}} > 500`  
     - String: `{{$json["deal_type"]}} == "newbusiness"`  
     - String: `{{$json["deal_stage"]}} != "closedlost"` and `{{$json["deal_stage"]}} != "closedwon"` (use not equal with regex or logic)  
   - Connect Set node output to IF node input.

9. **Add HubSpot node for high-priority tickets**  
   - Operation: Create ticket  
   - Pipeline ID: `0`  
   - Stage ID: `1`  
   - Ticket Name: `Deal: {{$json["deal_name"]}}`  
   - Priority: HIGH  
   - Description: `{{$json["deal_description"]}}`  
   - Ticket Owner ID: `12345` (experienced team member)  
   - Connect IF true output to this node.  
   - Use HubSpot credentials.

10. **Add HubSpot node for low-priority tickets**  
    - Operation: Create ticket  
    - Pipeline ID: `0`  
    - Stage ID: `1`  
    - Ticket Name: `Deal: {{$json["deal_name"]}}`  
    - Priority: MEDIUM  
    - Description: `{{$json["deal_description"]}}`  
    - Connect IF false output to this node.  
    - Use HubSpot credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                      |
|----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Slack channel used: `deals` for team notifications on closed won deals                                    | Slack node configuration                                           |
| Airtable table named `lost_deals` used to store lost deal data for analysis                               | Airtable node configuration                                        |
| HubSpot ticket owner ID `12345` should be replaced with a real user ID in your HubSpot account           | HubSpot ticket nodes                                               |
| Google Slides OAuth2 credential must have permissions to create presentations in your Google Workspace    | Google Slides node configuration                                   |
| HubSpot webhook ID `12345` is an example; ensure your webhook is properly registered in HubSpot           | HubSpot Trigger node                                               |