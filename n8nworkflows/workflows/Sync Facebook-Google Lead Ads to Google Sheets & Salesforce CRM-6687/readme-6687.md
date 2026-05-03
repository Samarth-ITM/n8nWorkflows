Sync Facebook/Google Lead Ads to Google Sheets & Salesforce CRM

https://n8nworkflows.xyz/workflows/sync-facebook-google-lead-ads-to-google-sheets---salesforce-crm-6687


# Sync Facebook/Google Lead Ads to Google Sheets & Salesforce CRM

---

### 1. Workflow Overview

This workflow automates the synchronization of new leads captured via Facebook Lead Ads into both Google Sheets and Salesforce CRM, enabling seamless tracking and follow-up. It is designed for marketing and sales teams who want to centralize lead data and maintain updated CRM records without manual data entry.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a new Facebook Lead Ad submission.
- **1.2 Data Preparation:** Extracts and formats lead data, particularly splitting the full name into first and last names for CRM compatibility.
- **1.3 Lead Logging:** Inserts the lead data into a Google Sheets document for record-keeping and status tracking.
- **1.4 CRM Integration:** Creates a new Lead record in Salesforce CRM using the prepared data.
- **1.5 Status Update:** Updates the lead’s status in Google Sheets after successful CRM synchronization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for new lead submissions from Facebook Lead Ads to initiate the workflow.

- **Nodes Involved:**  
  - Start  
  - Facebook Lead Ad (Trigger)

- **Node Details:**  

  - **Start**  
    - Type: Start Node  
    - Role: Entry point of the workflow, triggered automatically when a lead is received  
    - Configuration: Default, no parameters  
    - Connections: Output to Facebook Lead Ad node  
    - Edge cases: None; standard start node  

  - **Facebook Lead Ad**  
    - Type: Trigger Node specialized for Facebook Lead Ads  
    - Role: Initiates workflow on new Facebook lead submission  
    - Configuration: Uses credentials for Facebook Lead Ads API with dynamic parameters for `formId` and `pageId` from credentials  
    - Expressions: `formId` and `pageId` are injected via credential references (`{{$credential.formId}}`, `{{$credential.pageId}}`)  
    - Input: Start node output  
    - Output: Fires on new lead, outputs lead data JSON to "Prepare CRM Data" node  
    - Edge cases:  
      - Authorization failures if credentials are invalid or expired  
      - Network or API timeout errors  
      - Changes in Facebook API schema or permission scopes  
    - Credential Requirements: Valid Facebook Lead Ads API credentials  

#### 1.2 Data Preparation

- **Overview:**  
  Processes raw lead data, particularly splitting the full name into first and last names to conform with Salesforce lead creation requirements.

- **Nodes Involved:**  
  - Prepare CRM Data (Set Node)

- **Node Details:**  

  - **Prepare CRM Data**  
    - Type: Set Node  
    - Role: Parses and structures lead data for downstream CRM use  
    - Configuration:  
      - Extracts `firstName` as all parts of `full_name` except the last word  
      - Extracts `lastName` as the last word from `full_name`  
      - Uses JavaScript expressions to split the name string reliably, with fallback to full name if splitting fails  
    - Expressions:  
      - `firstName`: `={{ $json.full_name.split(' ').slice(0, -1).join(' ') || $json.full_name }}`  
      - `lastName`: `={{ $json.full_name.split(' ').pop() }}`  
    - Input: Facebook Lead Ad output  
    - Output: Prepared data forwarded to Google Sheets logging  
    - Edge cases:  
      - Single-word names where the last name extraction might be identical to first name  
      - Null or empty `full_name` fields causing expression errors  
      - Names with multiple spaces or unusual characters  
    - Version-specific: Requires n8n versions supporting `.split()` and `.pop()` in expressions (generally recent versions)  

#### 1.3 Lead Logging

- **Overview:**  
  Logs the new lead data into a Google Sheets document with timestamp and initial CRM status.

- **Nodes Involved:**  
  - Log Lead to Google Sheets

- **Node Details:**  

  - **Log Lead to Google Sheets**  
    - Type: Google Sheets Node (Append Row)  
    - Role: Records the lead information in a Google Sheet for audit and tracking  
    - Configuration:  
      - Appends a new row with columns: Timestamp, FullName, Email, PhoneNumber, LeadID, CRM_Status  
      - Timestamp is generated at runtime using current datetime formatted as `yyyy-MM-dd HH:mm:ss`  
      - CRM_Status initially set to `"Processing"` to indicate pending CRM sync  
      - `sheetName` and `documentId` are placeholders, to be replaced with actual Google Sheet identifiers  
    - Expressions:  
      - Timestamp: `={{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`  
      - Other columns mapped directly from incoming JSON  
    - Input: Prepared CRM data  
    - Output: Leads to Salesforce Lead creation node  
    - Credential Requirements: Google Sheets API credentials with edit access to target document  
    - Edge cases:  
      - Google API rate limits or permission issues  
      - Incorrect document or sheet names causing write failures  
      - Network interruptions  
    - Version: Uses version 3 of Google Sheets node with column mapping support  

#### 1.4 CRM Integration

- **Overview:**  
  Creates a new Lead record in Salesforce CRM using the data prepared and logged.

- **Nodes Involved:**  
  - Create Lead in Salesforce

- **Node Details:**  

  - **Create Lead in Salesforce**  
    - Type: Salesforce Node  
    - Role: Inserts a new Lead object into Salesforce CRM  
    - Configuration:  
      - Uses fields from `Prepare CRM Data` node output (email, phone_number, firstName, lastName)  
      - Sets static `Company` field as `"Lead from Facebook"`  
      - Sets `LeadSource` to `"Facebook Lead Ad"` to track origin  
      - Operation: Create Lead resource  
    - Expressions: Uses dot notation to access JSON data from `Prepare CRM Data` node  
    - Input: Output of Google Sheets logging node  
    - Output: Leads to status update in Google Sheets  
    - Credential Requirements: Salesforce API credentials with permissions to create Lead records  
    - Edge cases:  
      - Authentication failures or expired tokens  
      - Salesforce API limits or validation errors (e.g., missing required fields)  
      - Network or timeout issues  
    - Version: Salesforce node version 2  

#### 1.5 Status Update

- **Overview:**  
  Updates the lead’s status in the Google Sheet to confirm successful CRM synchronization.

- **Nodes Involved:**  
  - Update Status in Sheet

- **Node Details:**  

  - **Update Status in Sheet**  
    - Type: Google Sheets Node (Update Row)  
    - Role: Updates the previously logged lead row’s `CRM_Status` column to `"Synced to Salesforce"`  
    - Configuration:  
      - Uses `LeadID` as key to identify the correct row to update  
      - Updates only the `CRM_Status` column  
      - Sheet and document IDs dynamically reused from the earlier Google Sheets node parameters to ensure consistency  
    - Expressions:  
      - Key and value expressions refer back to `Log Lead to Google Sheets` node outputs and parameters  
    - Input: Output of Salesforce Lead creation node  
    - Credential Requirements: Same Google Sheets API credentials as logging node  
    - Edge cases:  
      - If the row is not found by `LeadID` key, update will fail silently or error based on Google Sheets node behavior  
      - API permission issues or network errors  
    - Version: Google Sheets node version 3  

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role               | Input Node(s)            | Output Node(s)                | Sticky Note                              |
|-------------------------|----------------------------|------------------------------|--------------------------|------------------------------|-----------------------------------------|
| Start                   | Start                      | Workflow entry point          | —                        | Facebook Lead Ad              |                                         |
| Facebook Lead Ad        | Facebook Lead Ads Trigger  | Receives new Facebook leads  | Start                    | Prepare CRM Data             |                                         |
| Prepare CRM Data        | Set                        | Splits full name into parts  | Facebook Lead Ad          | Log Lead to Google Sheets    |                                         |
| Log Lead to Google Sheets| Google Sheets (Append)     | Logs lead to Google Sheets   | Prepare CRM Data          | Create Lead in Salesforce    |                                         |
| Create Lead in Salesforce| Salesforce                 | Creates lead in Salesforce   | Log Lead to Google Sheets | Update Status in Sheet       |                                         |
| Update Status in Sheet  | Google Sheets (Update)     | Updates lead sync status     | Create Lead in Salesforce | —                            |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Start Node**  
   - Add a Start node with default settings.

2. **Add Facebook Lead Ad Trigger Node**  
   - Node Type: Facebook Lead Ads Trigger  
   - Configure credentials: Add Facebook Lead Ads API credentials with access to your Facebook page and form.  
   - Set `pageId` and `formId` parameters dynamically from credentials or enter manually.  
   - Connect Start node output to this node.

3. **Add Set Node for Data Preparation**  
   - Node Type: Set  
   - Add two string fields:  
     - `firstName` with expression: `{{$json.full_name.split(' ').slice(0, -1).join(' ') || $json.full_name}}`  
     - `lastName` with expression: `{{$json.full_name.split(' ').pop()}}`  
   - Connect Facebook Lead Ad output to this node.

4. **Add Google Sheets Node for Logging Lead**  
   - Node Type: Google Sheets (Append Row)  
   - Configure credentials: Add Google Sheets API credentials with write access.  
   - Set Spreadsheet ID and Sheet Name to your target Google Sheet.  
   - Map columns:  
     - Timestamp: `{{$now.toFormat('yyyy-MM-dd HH:mm:ss')}}`  
     - FullName: `{{$json.full_name}}`  
     - Email: `{{$json.email}}`  
     - PhoneNumber: `{{$json.phone_number}}`  
     - LeadID: `{{$json.leadgen_id}}`  
     - CRM_Status: `"Processing"`  
   - Connect Prepare CRM Data node output to this node.

5. **Add Salesforce Node to Create Lead**  
   - Node Type: Salesforce  
   - Configure credentials: Provide Salesforce API credentials with permissions to create leads.  
   - Set resource to Lead and operation to Create.  
   - Map fields:  
     - Email: `{{$nodes["Prepare CRM Data"].json.email}}`  
     - Phone: `{{$nodes["Prepare CRM Data"].json.phone_number}}`  
     - FirstName: `{{$nodes["Prepare CRM Data"].json.firstName}}`  
     - LastName: `{{$nodes["Prepare CRM Data"].json.lastName}}`  
     - Company: `"Lead from Facebook"` (static)  
     - LeadSource: `"Facebook Lead Ad"` (static)  
   - Connect Google Sheets logging node output to this node.

6. **Add Google Sheets Node to Update Lead Status**  
   - Node Type: Google Sheets (Update Row)  
   - Use the same Google Sheets credentials as before.  
   - Configure to find the row by key: Column `LeadID` equals `{{$nodes["Log Lead to Google Sheets"].json.LeadID}}`.  
   - Update only the column `CRM_Status` to `"Synced to Salesforce"`.  
   - Use the same spreadsheet ID and sheet name as the logging node (reference to parameters).  
   - Connect Salesforce node output to this node.

7. **Verify Connections**  
   - Start → Facebook Lead Ad → Prepare CRM Data → Log Lead to Google Sheets → Create Lead in Salesforce → Update Status in Sheet

8. **Activate Workflow**  
   - Save and activate the workflow to start listening for new Facebook Lead Ads submissions.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                    |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------|
| The workflow assumes all credentials (Facebook, Google Sheets, Salesforce) are preconfigured. | Credential setup is mandatory before activation.  |
| To handle name splitting more robustly, consider additional logic for edge cases (e.g., single names). | Potential enhancement for "Prepare CRM Data" node.|
| Salesforce API limits and permissions may affect lead creation; monitor API usage accordingly. | Salesforce API documentation: https://developer.salesforce.com/docs |
| Google Sheets columns must exist exactly as named; mismatches will cause errors.               | Ensure sheet schema matches node configuration.   |
| Facebook Lead Ads API permissions require page admin rights and form access.                   | Facebook developer portal: https://developers.facebook.com/docs/lead-ads |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---