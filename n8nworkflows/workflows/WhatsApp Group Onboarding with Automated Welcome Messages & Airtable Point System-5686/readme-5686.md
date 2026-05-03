WhatsApp Group Onboarding with Automated Welcome Messages & Airtable Point System

https://n8nworkflows.xyz/workflows/whatsapp-group-onboarding-with-automated-welcome-messages---airtable-point-system-5686


# WhatsApp Group Onboarding with Automated Welcome Messages & Airtable Point System

### 1. Workflow Overview

This workflow automates the onboarding process for new participants joining a specific WhatsApp group. Its primary purpose is to welcome new members with a customized message and to register them in an Airtable-based engagement system with initial points. It is designed for community managers or group administrators who want to streamline member engagement and track participation automatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receiving webhook notifications from Whapi when a user joins the WhatsApp group.
- **1.2 Event Filtering:** Ensuring the event corresponds to the targeted WhatsApp group and that the action is a participant addition.
- **1.3 Welcome Messaging:** Sending a personalized welcome message to the new participant via Whapi’s HTTP API.
- **1.4 Engagement Registration:** Creating a new record in Airtable to track the participant’s points and last interaction date.
- **1.5 Documentation and Annotation:** Sticky notes provide contextual explanations and instructions for maintainers and users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming POST webhook requests from Whapi, indicating that a new participant has joined the WhatsApp group.

- **Nodes Involved:**  
  - New group participant (Webhook)  
  - Sticky Note (New participant)

- **Node Details:**

  - **New group participant**  
    - *Type & Role:* Webhook node; entry point for receiving real-time notifications from Whapi.  
    - *Configuration:* Listens on path `0d328573-da59-489e-8f2f-784aa4c19b82` with HTTP POST method.  
    - *Expressions:* None explicitly used here.  
    - *Input/Output:* No input; outputs webhook payload with participant and group details.  
    - *Edge Cases:* Webhook may fail if Whapi is misconfigured or network issues occur. Payload format changes may break downstream nodes.  
    - *Notes:* Critical starting point; ensure external Whapi service is correctly configured to send events here.

  - **Sticky Note (New participant)**  
    - *Type & Role:* Documentation; explains that this node signifies a new user joining the group.  
    - *No other technical configuration.*

#### 2.2 Event Filtering

- **Overview:**  
  This block filters incoming webhook events to ensure processing only occurs for the intended WhatsApp group and only when the action is a participant addition.

- **Nodes Involved:**  
  - Filter  
  - Sticky Note2 (Conditions)

- **Node Details:**

  - **Filter**  
    - *Type & Role:* Filter node; applies conditional logic to webhook data.  
    - *Configuration:* Checks two conditions:  
      1. `group_id` matches a fixed group ID (`XXXXXXXXXXXXXXXX@g.us`).  
      2. `action` equals `"add"`.  
    - *Expressions Used:*  
      - `{{$json.body.groups_participants[0].group_id}}`  
      - `{{$json.body.groups_participants[0].action}}`  
    - *Input/Output:* Input from webhook node; outputs only if both conditions pass to the next node.  
    - *Edge Cases:*  
      - Group ID or action field missing or malformed may cause false negatives.  
      - Group ID hardcoded; requires manual update if group changes.  
    - *Version-Specific:* Uses Filter v2.2 capabilities.

  - **Sticky Note2 (Conditions)**  
    - *Notes:* Documents the filter logic for maintainers.

#### 2.3 Welcome Messaging

- **Overview:**  
  Sends a welcome message to the newly added participant in the WhatsApp group via the Whapi messaging API.

- **Nodes Involved:**  
  - Welcome WhatsApp message (HTTP Request)  
  - Sticky Note1 (Welcome message)  
  - Sticky Note3 (100 Start Points)

- **Node Details:**

  - **Welcome WhatsApp message**  
    - *Type & Role:* HTTP Request node; sends a POST request to Whapi API to deliver a WhatsApp text message.  
    - *Configuration:*  
      - URL: `https://gate.whapi.cloud/messages/text`  
      - Method: POST  
      - Headers:  
        - Accept: application/json  
        - Authorization: Bearer `{TOKEN}` (placeholder to be replaced with valid token)  
      - JSON Body:  
        ```json
        {
          "to": "{{ $json.body.groups_participants[0].participants[0] }}",
          "body": "MESSAGE"
        }
        ```  
        Here, `"to"` is dynamically set to the new participant’s WhatsApp ID from webhook data. `"MESSAGE"` should be replaced with the actual welcome text.  
    - *Expressions:* Uses JSON path to extract participant ID.  
    - *Input/Output:* Receives filtered event; outputs to Airtable Create node after successful message send.  
    - *Edge Cases:*  
      - HTTP failure (network, 401 Unauthorized if token invalid, rate limiting).  
      - Missing participant ID in payload.  
      - Static `"MESSAGE"` placeholder must be replaced with meaningful text before deployment.  
    - *Version:* HTTP Request v4.2.

  - **Sticky Note1 (Welcome message)**  
    - Explains purpose of sending a welcome message via Whapi.

  - **Sticky Note3 (100 Start Points)**  
    - Notes the initial points assigned to new users (100 points).

#### 2.4 Engagement Registration

- **Overview:**  
  Creates a new record in Airtable to track the new participant’s engagement points and last interaction date.

- **Nodes Involved:**  
  - Airtable Create  
  - Sticky Note3 (also applies here)

- **Node Details:**

  - **Airtable Create**  
    - *Type & Role:* Airtable node; creates a new record in a specified base and table.  
    - *Configuration:*  
      - Base: `appREiqyOxTYwsigc` ("WhatsApp Engagement Database")  
      - Table: `tblIf7YbtyvUvDNm0` ("Table 1")  
      - Fields set:  
        - WhatsApp_ID: dynamically set to new participant’s WhatsApp ID.  
        - Count: 100 (starting points).  
        - Last interaction: current date formatted as `yyyy.MM.dd`.  
      - Mapping mode: manual field definition.  
    - *Expressions:*  
      - `{{$('New group participant').item.json.body.groups_participants[0].participants[0]}}` for WhatsApp_ID  
      - `$now.format('yyyy.MM.dd')` for date  
    - *Credentials:* Uses Airtable Personal Access Token credential.  
    - *Input/Output:* Input from Welcome WhatsApp message node; outputs data for further steps or finalization (none here).  
    - *Edge Cases:*  
      - Airtable API rate limits or authentication errors.  
      - Duplicate participant entries if webhook fires multiple times.  
      - Date formatting failures (unlikely with standard `$now`).  
    - *Version:* Airtable node v2.1.

  - **Sticky Note3**  
    - Emphasizes that new users start with 100 points.

#### 2.5 Documentation and Annotation

- **Overview:**  
  Comprehensive sticky note describing the entire workflow for maintainers and users.

- **Nodes Involved:**  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note4**  
    - *Content:*  
      Describes the workflow’s purpose, steps, and expected outcome in detail.  
      Highlights the webhook trigger, filtering, welcome message sending, Airtable registration, and overall benefit.  
    - *Role:* Documentation for clarity and maintenance.

---

### 3. Summary Table

| Node Name              | Node Type        | Functional Role                         | Input Node(s)          | Output Node(s)          | Sticky Note                                              |
|------------------------|------------------|---------------------------------------|-----------------------|-------------------------|----------------------------------------------------------|
| New group participant  | Webhook          | Receive new participant webhook       | -                     | Filter                  | ## New participant  A new user has joined the group.     |
| Filter                 | Filter           | Filter events for correct group & add | New group participant  | Welcome WhatsApp message | ## Conditions: 1. Right group? 2. Type "add"?            |
| Welcome WhatsApp message| HTTP Request     | Send WhatsApp welcome message          | Filter                | Airtable Create         | ## Welcome message  Send a "welcome message" via Whapi.  |
| Airtable Create        | Airtable         | Create engagement record for user      | Welcome WhatsApp message| -                       | ## 100 Start Points  New users start with 100 points     |
| Sticky Note            | Sticky Note      | Documentation                         | -                     | -                       | ## New participant  A new user has joined the group.     |
| Sticky Note1           | Sticky Note      | Documentation                         | -                     | -                       | ## Welcome message  Send a "welcome message" via Whapi.  |
| Sticky Note2           | Sticky Note      | Documentation                         | -                     | -                       | ## Conditions: 1. Right group? 2. Type "add"?            |
| Sticky Note3           | Sticky Note      | Documentation                         | -                     | -                       | ## 100 Start Points  New users start with 100 points     |
| Sticky Note4           | Sticky Note      | Documentation                         | -                     | -                       | # New WhatsApp Group Participant Workflow ...            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `New group participant`  
   - Type: Webhook (HTTP POST)  
   - Path: `0d328573-da59-489e-8f2f-784aa4c19b82`  
   - Method: POST  
   - Purpose: Receive webhook notifications from Whapi when a participant joins.

2. **Add Filter Node:**  
   - Name: `Filter`  
   - Type: Filter (v2.2)  
   - Connect input from `New group participant`.  
   - Set conditions:  
     - `{{$json.body.groups_participants[0].group_id}}` equals `"XXXXXXXXXXXXXXXX@g.us"` (replace with your WhatsApp group ID).  
     - `{{$json.body.groups_participants[0].action}}` equals `"add"`.  
   - Route only matching events forward.

3. **Add HTTP Request Node:**  
   - Name: `Welcome WhatsApp message`  
   - Type: HTTP Request (v4.2)  
   - Connect input from `Filter`.  
   - Configure:  
     - URL: `https://gate.whapi.cloud/messages/text`  
     - Method: POST  
     - Headers:  
       - `accept: application/json`  
       - `authorization: Bearer {TOKEN}` (replace `{TOKEN}` with your valid Whapi API token).  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "to": "{{ $json.body.groups_participants[0].participants[0] }}",
         "body": "Welcome to the group! Please read the rules and participate to earn points."
       }
       ```  
     - Enable sending body and headers.

4. **Add Airtable Node:**  
   - Name: `Airtable Create`  
   - Type: Airtable (v2.1)  
   - Connect input from `Welcome WhatsApp message`.  
   - Configure credentials with Airtable Personal Access Token.  
   - Select Base: `WhatsApp Engagement Database` (base ID: `appREiqyOxTYwsigc`)  
   - Select Table: `Table 1` (table ID: `tblIf7YbtyvUvDNm0`)  
   - Set operation: Create record  
   - Map fields:  
     - WhatsApp_ID: `{{$('New group participant').item.json.body.groups_participants[0].participants[0]}}`  
     - Count: `100` (starting points)  
     - Last interaction: `{{$now.format('yyyy.MM.dd')}}`  

5. **Optional: Add Sticky Notes** for documentation:  
   - New participant description near webhook node.  
   - Conditions explanation near Filter node.  
   - Welcome message explanation near HTTP Request node.  
   - Points explanation near Airtable node.  
   - Full workflow description as a large note for maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                              |
|------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| This workflow automates onboarding for WhatsApp group participants using Whapi API and Airtable for engagement tracking. | Workflow purpose overview                                                   |
| Replace `"XXXXXXXXXXXXXXXX@g.us"` with your target WhatsApp group ID in the Filter node for correct filtering.           | Critical configuration detail                                                |
| Replace `{TOKEN}` in the HTTP Request node with a valid Bearer token from Whapi for API authentication.                  | API credential setup                                                         |
| Airtable Personal Access Token credential must be configured in n8n with appropriate permissions on the target base.    | Airtable integration credential requirement                                 |
| Welcome message content should be customized to include your group’s rules and participation instructions.               | Message personalization advice                                               |
| Webhook URL path is auto-generated and must be correctly configured in Whapi to POST events to this workflow.           | Webhook configuration detail                                                 |
| Ensure Whapi event payload structure matches expected format to prevent runtime errors.                                  | Payload format dependency                                                    |
| For best reliability, monitor API usage limits for Whapi and Airtable to avoid throttling.                               | Integration best practices                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It fully complies with content policies and contains no illegal, offensive, or protected elements. All data processed are legal and publicly accessible.