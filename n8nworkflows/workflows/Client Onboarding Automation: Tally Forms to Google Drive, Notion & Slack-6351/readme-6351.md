Client Onboarding Automation: Tally Forms to Google Drive, Notion & Slack

https://n8nworkflows.xyz/workflows/client-onboarding-automation--tally-forms-to-google-drive--notion---slack-6351


# Client Onboarding Automation: Tally Forms to Google Drive, Notion & Slack

### 1. Workflow Overview

This workflow automates client onboarding by integrating responses from a Tally form into Google Drive, Notion, and Slack. It is designed for teams that want to streamline the creation of client-specific folders, maintain organized client data in Notion, and notify team members via Slack upon new client onboarding.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts and receives client onboarding submissions from a Tally form via an HTTP webhook.
- **1.2 Data Extraction:** Parses and extracts relevant client information (Name, Email, Project Type, Budget) from the raw Tally form submission.
- **1.3 Google Drive Folder Creation:** Creates a client-specific folder inside a predefined parent Google Drive folder.
- **1.4 Data Augmentation:** Adds folder ID and unique execution ID to the extracted client data.
- **1.5 Notion Database Page Creation:** Creates a new page in a Notion database representing the client with extracted and augmented data.
- **1.6 Slack Notification:** Sends a formatted notification message to a Slack channel to inform the team about the new onboarding.
- **1.7 Data Merging:** Combines data streams from folder creation and data extraction to unify inputs for downstream nodes.
- **1.8 Documentation & Setup Notes:** A sticky note provides setup instructions and reminders.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives HTTP POST requests from Tally form submissions at a dedicated webhook endpoint.
- **Nodes Involved:** `Webhook`
- **Node Details:**
  - **Type:** Webhook
  - **Role:** Entry point listening for POST requests at path `/tally-submission`.
  - **Configuration:** 
    - HTTP Method: POST
    - Path: `/tally-submission`
  - **Input/Output:** No inputs; outputs raw submission JSON.
  - **Edge Cases:** 
    - Missing or malformed POST data.
    - Unauthorized or unexpected requests.
    - Webhook URL must be publicly accessible.
  - **Notes:** Expected Tally fields are Name, Email, Project Type, Budget.

#### 1.2 Data Extraction

- **Overview:** Extracts and normalizes client information from the Tally form submission JSON.
- **Nodes Involved:** `Extract Client Fields`
- **Node Details:**
  - **Type:** Code (JavaScript)
  - **Role:** Parses JSON array of fields, identifies and extracts values for name, email, project type, and budget.
  - **Key Expressions:** Uses label matching (case-insensitive) and option ID lookups.
  - **Input:** Raw Tally payload from `Webhook`.
  - **Output:** JSON object with keys: `name`, `email`, `projectType`, `budget`.
  - **Edge Cases:**
    - Missing fields or empty values.
    - Unexpected field labels or data structures.
    - Option IDs not found in options array.
  - **Version:** Uses n8n Code node v2.

#### 1.3 Google Drive Folder Creation

- **Overview:** Creates a new folder in Google Drive named after the client plus "- Onboarding" suffix.
- **Nodes Involved:** `Create folder`
- **Node Details:**
  - **Type:** Google Drive
  - **Role:** Creates a folder resource inside a fixed parent folder.
  - **Configuration:**
    - Folder name: `${name} - Onboarding`
    - Parent folder ID: Hardcoded `"1rCt7cyX7b3FQSJRDB8bRT4ho9QEUh0mA"`
    - Drive: "My Drive"
  - **Input:** Extracted client data with `name`.
  - **Output:** Folder metadata including folder ID.
  - **Credentials:** Google Drive OAuth2
  - **Edge Cases:**
    - OAuth token expiration or invalid credentials.
    - Folder name conflicts or quota limits.
    - Parent folder ID incorrect or inaccessible.
  - **Notes:** Parent folder ID is hardcoded, must be updated for other environments.

#### 1.4 Data Augmentation

- **Overview:** Adds execution-level metadata (workflow execution ID) and the Google Drive folder ID to the client data.
- **Nodes Involved:** `Edit Fields1`
- **Node Details:**
  - **Type:** Set
  - **Role:** Adds two string fields:
    - `uuid`: The current workflow execution ID (`$execution.id`)
    - `id`: The Google Drive folder ID (`$json.id`)
  - **Input:** Folder creation output.
  - **Output:** Augmented JSON with added fields.
  - **Edge Cases:** Missing input data or execution context.

#### 1.5 Notion Database Page Creation

- **Overview:** Inserts a new page into a Notion database with client details and a link to the Google Drive folder.
- **Nodes Involved:** `Create a database page`
- **Node Details:**
  - **Type:** Notion
  - **Role:** Creates a database page with properties:
    - Name (title)
    - Email (email)
    - Project Type (select)
    - Budget (select)
    - Onboarding Link (URL linking to Google Drive folder)
  - **Configuration:**
    - Database ID: Hardcoded `"230bb34e-b122-800a-87e4-c5f2e30500b9"`
    - Property mapping uses expressions to fill from JSON data.
  - **Input:** Merged data including augmented client info.
  - **Output:** Notion page metadata including URL.
  - **Credentials:** Notion OAuth2
  - **Edge Cases:**
    - Invalid or missing database ID.
    - Missing required fields in Notion database schema.
    - OAuth token invalid/expired.
    - API rate limits.
  - **Version:** n8n Notion node v2.2

#### 1.6 Slack Notification

- **Overview:** Sends a formatted notification message to a Slack channel about the new client onboarding.
- **Nodes Involved:** `Send a message`
- **Node Details:**
  - **Type:** Slack
  - **Role:** Posts message with client details and links to Google Drive folder and Notion page.
  - **Configuration:**
    - Channel: `#client-notifications` (hardcoded)
    - Message text uses expressions referencing properties from Notion page creation output.
    - Authentication: OAuth2
  - **Input:** Output from Notion node.
  - **Output:** Slack API response.
  - **Credentials:** Slack OAuth2
  - **Edge Cases:**
    - Invalid channel name or missing permissions.
    - OAuth token expired.
    - Slack API rate limits or network issues.

#### 1.7 Data Merging

- **Overview:** Combines the data streams from extracted client info and folder creation to unify input for Notion page creation.
- **Nodes Involved:** `Merge`
- **Node Details:**
  - **Type:** Merge
  - **Role:** Combines two inputs by position (index).
  - **Inputs:** 
    - Index 0: Augmented client data including folder ID.
    - Index 1: Extracted client data.
  - **Output:** Merged JSON with full client info and folder metadata.
  - **Edge Cases:** Mismatched input array lengths or missing inputs.

#### 1.8 Documentation & Setup Notes

- **Overview:** Provides setup instructions and important notes for users.
- **Nodes Involved:** `Sticky Note`
- **Node Details:**
  - **Type:** Sticky Note
  - **Role:** Contains manual setup instructions, OAuth2 credential reminders, and notes about hardcoded IDs.
  - **Content Highlights:**
    - Webhook URL and expected Tally fields.
    - Parent Google Drive folder ID must be updated.
    - Notion database ID requirements.
    - Slack channel and OAuth2 credentials needed.
    - Warning about hardcoded values.
  - **Position:** Placed visually off to the side for user reference.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                    | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                                                         |
|---------------------|---------------------|----------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook             | Receives Tally form submissions  | —                     | Extract Client Fields     | See setup instructions in Sticky Note                                                                                              |
| Extract Client Fields| Code (JavaScript)   | Parses and extracts client data  | Webhook               | Create folder, Merge      |                                                                                                                                    |
| Create folder       | Google Drive        | Creates client Google Drive folder| Extract Client Fields  | Edit Fields1             | Parent folder ID hardcoded; update as needed                                                                                       |
| Edit Fields1        | Set                 | Adds execution and folder IDs    | Create folder          | Merge                    |                                                                                                                                    |
| Merge               | Merge               | Combines extracted and augmented data | Extract Client Fields, Edit Fields1 | Create a database page |                                                                                                                                    |
| Create a database page | Notion             | Creates Notion page with client data | Merge                  | Send a message           | Database ID hardcoded; update as needed                                                                                            |
| Send a message      | Slack               | Sends Slack notification         | Create a database page | —                        | Slack channel hardcoded; update as needed                                                                                          |
| Sticky Note         | Sticky Note         | Setup instructions and notes     | —                     | —                        | Contains comprehensive setup guide and warnings about hardcoded values and OAuth2 credentials                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `/tally-submission`
   - This node will receive Tally form submissions.
   - No credentials needed.
   
2. **Add a Code node named "Extract Client Fields":**
   - Type: Code (JavaScript)
   - Input: Output of the Webhook node.
   - Paste the following code to extract fields:
     ```javascript
     const fields = $json.body.data.fields;
     let result = {};
     fields.forEach(field => {
       const label = field.label.toLowerCase();
       if (label.includes('name')) {
         result.name = field.value;
       }
       if (label.includes('email')) {
         result.email = field.value;
       }
       if (label.includes('project')) {
         const selectedId = field.value[0];
         const option = field.options.find(opt => opt.id === selectedId);
         result.projectType = option?.text || '';
       }
       if (label.includes('budget')) {
         const selectedId = field.value[0];
         const option = field.options.find(opt => opt.id === selectedId);
         result.budget = option?.text || '';
       }
     });
     return [{ json: result }];
     ```
   
3. **Add a Google Drive node named "Create folder":**
   - Type: Google Drive
   - Resource: Folder
   - Operation: Create folder
   - Name: Expression — `{{$json.name + " - Onboarding"}}`
   - Parent Folder ID: Hardcoded to `"1rCt7cyX7b3FQSJRDB8bRT4ho9QEUh0mA"` (replace with your own)
   - Drive ID: "My Drive"
   - Connect input from "Extract Client Fields".
   - Configure Google Drive OAuth2 credentials.
   
4. **Add a Set node named "Edit Fields1":**
   - Type: Set
   - Add fields:
     - `uuid` (string): `{{$execution.id}}`
     - `id` (string): `{{$json.id}}` (folder ID from previous node)
   - Connect input from "Create folder".
   
5. **Add a Merge node named "Merge":**
   - Type: Merge
   - Mode: Combine
   - Combine By: Position
   - Connect two inputs:
     - Input 1: Output of "Edit Fields1"
     - Input 2: Output of "Extract Client Fields"
   
6. **Add a Notion node named "Create a database page":**
   - Type: Notion
   - Resource: Database Page
   - Database ID: Hardcoded to `"230bb34e-b122-800a-87e4-c5f2e30500b9"` (replace with your own)
   - Properties mapping:
     - Name (title): `{{$json.name}}`
     - Email (email): `{{$json.email}}`
     - Project Type (select): `{{$json.projectType}}`
     - Budget (select): `{{$json.budget}}`
     - Onboarding Link (url): `https://drive.google.com/drive/folders/{{$json["id"]}}`
   - Connect input from "Merge".
   - Configure Notion OAuth2 credentials.
   
7. **Add a Slack node named "Send a message":**
   - Type: Slack
   - Operation: Post message
   - Channel: `#client-notifications` (replace if needed)
   - Message Text (use multiline expression):
     ```
     🚀 New Client Onboarded!

     👤 Name: {{$json["property_name"]}}
     📧 Email: {{$json["property_email"]}}
     💼 Project Type: {{$json["property_project_type"]}}
     💰 Budget: {{$json["property_budget"]}}
     📂 Folder: {{$json["property_onboarding_link"]}}
     🔗 Notion Page: {{$json["url"]}}
     ```
   - Connect input from "Create a database page".
   - Configure Slack OAuth2 credentials.
   
8. **Add a Sticky Note node for documentation (optional):**
   - Paste setup guide content about webhook URL, credentials, and hardcoded IDs.
   
9. **Connect the nodes in this order:**
   - Webhook → Extract Client Fields
   - Extract Client Fields → Create folder
   - Create folder → Edit Fields1
   - Edit Fields1 → Merge (input 1)
   - Extract Client Fields → Merge (input 2)
   - Merge → Create a database page
   - Create a database page → Send a message

10. **Credential Setup:**
    - Google Drive OAuth2: Add credentials with access to the parent folder.
    - Notion OAuth2: Add credentials with access to the target database.
    - Slack OAuth2: Add credentials with message posting rights to the target channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow uses hardcoded folder IDs, database IDs, and Slack channel names. Replace all hardcoded values with your own before deployment.                                                                                                                                             | Setup instructions in Sticky Note node                          |
| All external integrations use OAuth2 authentication—no API keys are stored in plaintext.                                                                                                                                                                                                    | Security best practices                                         |
| The webhook URL endpoint (`/tally-submission`) must be publicly accessible for Tally to send data. Consider using n8n webhook URL with HTTPS and proper domain setup.                                                                                                                        | Tally form integration                                          |
| For Notion, ensure the database schema supports properties: Name (title), Email (email), Project Type (select), Budget (select), and Onboarding Link (URL).                                                                                                                                  | Notion database setup                                          |
| Slack messages are sent to `#client-notifications`. Adjust the channel as needed and verify that the Slack OAuth2 app has permission to post in it.                                                                                                                                          | Slack app permissions                                          |
| Refer to n8n documentation for OAuth2 credential creation and node-specific configuration details for Google Drive, Notion, and Slack.                                                                                                                                                      | https://docs.n8n.io                                              |

---

**Disclaimer:** This text is derived solely from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.