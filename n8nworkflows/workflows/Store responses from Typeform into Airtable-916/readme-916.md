Store responses from Typeform into Airtable

https://n8nworkflows.xyz/workflows/store-responses-from-typeform-into-airtable-916


# Store responses from Typeform into Airtable

---
### 1. Workflow Overview

This workflow automates the process of capturing new responses from a Typeform survey, storing these responses into an Airtable database, and sending a notification message to a Slack channel with the submitted data. It is designed primarily for organizations or teams that want to streamline form data collection and sharing through automated integration.

The workflow’s logic is composed of the following blocks:

- **1.1 Input Reception:** Captures new Typeform submissions via a webhook trigger.
- **1.2 Data Transformation:** Extracts and formats relevant form fields.
- **1.3 Data Storage:** Appends the formatted data to an Airtable base.
- **1.4 Notification:** Sends a summary message with the form data to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new responses submitted to a specific Typeform form. When a submission occurs, it triggers the workflow execution and provides the raw data for subsequent processing.

- **Nodes Involved:**  
  - Typeform Trigger

- **Node Details:**

  - **Typeform Trigger**  
    - **Type:** Trigger node (Typeform integration)  
    - **Technical Role:** Listens for new form submission events from Typeform via webhook.  
    - **Configuration:**  
      - Connected to a specific Typeform form identified by `formId` ("dpr2kxSL").  
      - Uses provided Typeform API credentials ("Typeform Access Token") to authenticate API requests and webhook setup.  
      - Webhook ID is auto-generated for receiving events.  
    - **Key Expressions/Variables:** None at trigger; outputs raw submission JSON.  
    - **Input/Output Connections:** No input; output connects to Set node.  
    - **Version Requirements:** Requires n8n version supporting Typeform Trigger node v1.  
    - **Potential Failure Modes:**  
      - Authentication errors due to invalid or expired API token.  
      - Webhook registration failures if Typeform API limits are exceeded.  
      - Network timeouts or webhook delivery issues.  
    - **Sub-workflow:** None.

#### 2.2 Data Transformation

- **Overview:**  
  This block extracts specific answers from the raw Typeform submission JSON and maps them into simplified key-value pairs for easier consumption downstream.

- **Nodes Involved:**  
  - Set

- **Node Details:**

  - **Set**  
    - **Type:** Data manipulation node  
    - **Technical Role:** Selects and renames form answers into structured variables.  
    - **Configuration:**  
      - Configured to keep only the set fields (clearing all other data).  
      - Extracts two fields:  
        - `Name` from the question labeled "Let's start with your name."  
        - `Email` from the question labeled "What's your email address?"  
      - Uses expressions to access these fields dynamically from the incoming JSON.  
    - **Key Expressions/Variables:**  
      - `={{$json["Let's start with your name."]}}`  
      - `={{$json["What's your email address?"]}}`  
    - **Input/Output Connections:** Input from Typeform Trigger; output to Airtable node.  
    - **Version Requirements:** Compatible with n8n v1 expression syntax.  
    - **Potential Failure Modes:**  
      - Missing or differently named form fields causing expression failures or empty values.  
      - Data type mismatches if expected string is undefined.  
    - **Sub-workflow:** None.

#### 2.3 Data Storage

- **Overview:**  
  This block appends the extracted form data into an Airtable table as a new record.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**

  - **Airtable**  
    - **Type:** API node (Airtable integration)  
    - **Technical Role:** Inserts new records into a specified Airtable base and table.  
    - **Configuration:**  
      - Operation set to "append" (add new record).  
      - Target table specified as "Table 1".  
      - Uses stored Airtable API credentials for authentication.  
      - No special options enabled (e.g., batch size, typecast).  
    - **Key Expressions/Variables:** Takes input JSON from Set node as record fields.  
    - **Input/Output Connections:** Input from Set; output to Slack node.  
    - **Version Requirements:** Requires n8n version supporting Airtable node v1.  
    - **Potential Failure Modes:**  
      - Authentication failure if Airtable API key is invalid or revoked.  
      - Table name mismatch or missing table causing API errors.  
      - Rate limits or API downtime.  
    - **Sub-workflow:** None.

#### 2.4 Notification

- **Overview:**  
  Sends a formatted message to a Slack channel to notify team members of the new form submission.

- **Nodes Involved:**  
  - Slack

- **Node Details:**

  - **Slack**  
    - **Type:** API node (Slack integration)  
    - **Technical Role:** Posts a message to a Slack channel.  
    - **Configuration:**  
      - Message text includes static and dynamic content:  
        - Static prefix "*New Submission* 🙌"  
        - Dynamic insertion of `Name` and `Email` variables from Set node.  
      - Target Slack channel set to "general".  
      - Uses Slack Bot credentials for authentication.  
      - No attachments or additional options configured.  
    - **Key Expressions/Variables:**  
      - `=*New Submission* 🙌\nName: {{$node["Set"].json["Name"]}}\nEmail: {{$node["Set"].json["Email"]}}`  
    - **Input/Output Connections:** Input from Airtable node; no output (end node).  
    - **Version Requirements:** Requires Slack node v1 and OAuth2 credentials configured with proper scopes (chat:write).  
    - **Potential Failure Modes:**  
      - Authentication errors due to invalid Slack bot token.  
      - Channel not found or bot not invited to the channel.  
      - Message formatting errors if variables are undefined.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role           | Input Node(s)       | Output Node(s) | Sticky Note                                                                                   |
|------------------|---------------------|--------------------------|---------------------|----------------|----------------------------------------------------------------------------------------------|
| Typeform Trigger | Typeform Trigger    | Captures new form responses | -                   | Set            |                                                                                              |
| Set              | Set                 | Extracts and formats form data | Typeform Trigger    | Airtable       | **Configure this Set node if your form uses different fields.**                              |
| Airtable         | Airtable            | Stores form data in Airtable | Set                 | Slack          |                                                                                              |
| Slack            | Slack               | Sends notification to Slack | Airtable            | -              |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger node:**  
   - Node Type: Typeform Trigger  
   - Configure credentials: Add and select “Typeform Access Token” with your API token.  
   - Set `formId` to your Typeform form's ID (e.g., "dpr2kxSL").  
   - This node will auto-generate a webhook URL. Copy this URL and ensure it’s registered in your Typeform form settings (usually automatic with credentials).  

2. **Add Set node:**  
   - Node Type: Set  
   - Enable "Keep Only Set" option to remove all other incoming data.  
   - Add two string fields:  
     - `Name` with value expression: `={{$json["Let's start with your name."]}}`  
     - `Email` with value expression: `={{$json["What's your email address?"]}}`  
   - Connect output of Typeform Trigger to this Set node.  
   - Note: Adjust the field names in expressions if your Typeform question titles differ.

3. **Add Airtable node:**  
   - Node Type: Airtable  
   - Configure credentials: Add and select “Airtable Credentials n8n” with your Airtable API key.  
   - Set operation to "Append".  
   - Select the target table (e.g., "Table 1").  
   - Connect output of Set node to Airtable node.  

4. **Add Slack node:**  
   - Node Type: Slack  
   - Configure credentials: Add and select “Slack Bot Credentials” with OAuth2 token having `chat:write` scope.  
   - Set channel to "general" or your target Slack channel.  
   - Set message text to:  
     ```
     =*New Submission* 🙌
     Name: {{$node["Set"].json["Name"]}}
     Email: {{$node["Set"].json["Email"]}}
     ```  
   - Connect output of Airtable node to Slack node.  

5. **Test the workflow:**  
   - Save and activate the workflow.  
   - Submit a test response in the configured Typeform.  
   - Confirm data is appended in Airtable and notification appears in Slack channel.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Remember to update the Set node expressions if your Typeform question titles change.            | Workflow configuration detail                    |
| Slack Bot token must have `chat:write` permission and be invited to the target channel ("general").| Slack API documentation: https://api.slack.com/ |
| Airtable API key needs appropriate permissions to modify the target base and table.             | Airtable API docs: https://airtable.com/api      |

---

This comprehensive documentation enables users and AI agents to fully understand, reproduce, and maintain the workflow for integrating Typeform responses with Airtable and Slack notifications.