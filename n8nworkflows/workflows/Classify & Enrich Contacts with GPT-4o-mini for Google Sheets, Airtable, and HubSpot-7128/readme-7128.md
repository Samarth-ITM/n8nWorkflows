Classify & Enrich Contacts with GPT-4o-mini for Google Sheets, Airtable, and HubSpot

https://n8nworkflows.xyz/workflows/classify---enrich-contacts-with-gpt-4o-mini-for-google-sheets--airtable--and-hubspot-7128


# Classify & Enrich Contacts with GPT-4o-mini for Google Sheets, Airtable, and HubSpot

### 1. Workflow Overview

This workflow automates the classification and enrichment of contact data using GPT-4o-mini and integrates the results into Google Sheets, Airtable, and HubSpot. It is designed to process pending contacts from a Google Sheet, enhance their profiles with AI-driven role and seniority classification, and synchronize enriched data across multiple CRM platforms.

Logical blocks:

- **1.1 Scheduled Trigger**: Periodic initiation of the workflow every hour.
- **1.2 Data Retrieval and Filtering**: Reading pending contact rows from Google Sheets and filtering incomplete entries.
- **1.3 Data Cleaning**: Normalizing and completing contact fields.
- **1.4 AI Classification**: Using GPT-4o-mini to determine function, seniority, and department, with a fallback mechanism.
- **1.5 Data Synchronization**: Updating the enriched contacts back into Google Sheets, Airtable, and HubSpot.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers workflow execution automatically every hour to process new or updated contacts.

- **Nodes Involved:**  
  - ⏰ Run Every Hour

- **Node Details:**  
  - **⏰ Run Every Hour**  
    - Type: Schedule Trigger  
    - Configuration: Trigger interval set to every 1 hour.  
    - Input: None (trigger node).  
    - Output: Triggers downstream nodes to start processing.  
    - Edge Cases: Workflow depends on schedule; if n8n instance is down or paused, triggers may be missed. No authentication issues here.  

#### 1.2 Data Retrieval and Filtering

- **Overview:**  
  Reads contacts from a specified Google Sheet where the "Status" column equals "Pending", then filters out any rows lacking essential data such as Name or Email.

- **Nodes Involved:**  
  - 📄 Read Pending Contacts  
  - 🧪 Filter Name & Email

- **Node Details:**  
  - **📄 Read Pending Contacts**  
    - Type: Google Sheets node  
    - Configuration: Reads rows from Sheet1 of a Google Sheet identified by `YOUR_GOOGLE_SHEET_ID`. Filters rows where "Status" column equals "Pending". Returns only the first match per execution (can be changed to all if needed).  
    - Input: Trigger from schedule node.  
    - Output: Rows with pending contacts.  
    - Edge Cases: Authentication with Google Sheets via OAuth2 or API key must be valid. Possible errors include sheet access denied, invalid document ID, or network errors.  
  - **🧪 Filter Name & Email**  
    - Type: Filter node  
    - Configuration: Checks that both "Name" and "Email" fields are non-empty strings. Case-sensitive and strict validation.  
    - Input: Output from Google Sheets node.  
    - Output: Only contacts with valid Name and Email proceed.  
    - Edge Cases: Contacts missing either field get filtered out and do not continue, which might hide some partial data issues if critical fields are missing.

#### 1.3 Data Cleaning

- **Overview:**  
  Normalizes field names by trimming spaces and ensures all key contact attributes exist, filling missing optional fields with default placeholders.

- **Nodes Involved:**  
  - 🧹 Clean Contact Data

- **Node Details:**  
  - **🧹 Clean Contact Data**  
    - Type: Code node (JavaScript)  
    - Configuration: Iterates over all incoming items; trims whitespace from keys; assigns default values if fields like Title, Company, Phone, LinkedIn, or Notes are missing.  
    - Input: Filtered contacts with valid Name and Email.  
    - Output: Cleaned contact objects with consistent fields.  
    - Edge Cases: Assumes input JSON keys might have extra spaces; code is synchronous and handles all items. Errors could arise if input is unexpectedly formatted or empty.

#### 1.4 AI Classification

- **Overview:**  
  Uses GPT-4o-mini via LangChain agent node to classify contacts by function, seniority, and department. Parses the AI JSON response; if parsing fails, applies a keyword-based fallback. Adds an analysis timestamp.

- **Nodes Involved:**  
  - 🧠 LLM - GPT-4o-mini  
  - 🧩 Parse or Fallback Role Tags

- **Node Details:**  
  - **🧠 LLM - GPT-4o-mini**  
    - Type: LangChain Agent node (GPT-4o-mini model)  
    - Configuration: Sends a prompt instructing the AI to categorize the contact by function, seniority, department, confidence score (0-100), and reasoning. Uses contact fields as prompt variables.  
    - Input: Cleaned contacts.  
    - Output: AI-generated JSON classification string per contact.  
    - Version-specific: Requires n8n LangChain integration and valid OpenAI API credentials supporting GPT-4o-mini.  
    - Edge Cases: API rate limits, authentication failures, timeouts, or malformed AI responses can occur.  
  - **🧩 Parse or Fallback Role Tags**  
    - Type: Code node (JavaScript)  
    - Configuration: Parses the AI JSON response using regex and JSON.parse; on failure, performs fallback classification by keyword matching title text. Adds enriched fields and timestamp.  
    - Input: AI response and original contact data (from cleaned data node).  
    - Output: Enriched contact JSON with classification fields.  
    - Edge Cases: Parsing may fail if AI response is malformed; fallback ensures processing continues but with low confidence. Timing and indexing rely on node input order consistency.

#### 1.5 Data Synchronization

- **Overview:**  
  Updates the enriched contact data back into Google Sheets (marking status as Reviewed), upserts the record into Airtable, and pushes or updates the contact in HubSpot.

- **Nodes Involved:**  
  - 📊 Update Contact in Sheet  
  - 🔁 Sync to Airtable  
  - 📬 Push to HubSpot

- **Node Details:**  
  - **📊 Update Contact in Sheet**  
    - Type: Google Sheets node  
    - Configuration: Append or update operation keyed on Email, updates columns including Function, Seniority, Confidence Score, and Status set to "Reviewed". Uses the same Google Sheet and Sheet1.  
    - Input: Enriched contact output from parse/fallback node.  
    - Output: Updated rows in Google Sheets.  
    - Edge Cases: Google Sheets API quota limits, access errors, or mismatched email keys may cause update failures.  
  - **🔁 Sync to Airtable**  
    - Type: Airtable node  
    - Configuration: Upserts records by Email in specified Airtable Base and Table. Maps enriched fields (Name, Email, Phone, Title, Company, Function, LinkedIn, Seniority, Department, Confidence Score). Uses App Token authentication.  
    - Input: Enriched contact data.  
    - Output: Synchronized Airtable records.  
    - Edge Cases: Airtable API rate limits, authentication expiry, missing base/table IDs, or schema mismatches.  
  - **📬 Push to HubSpot**  
    - Type: HubSpot node  
    - Configuration: Creates or updates a contact based on email using App Token authentication. No additional fields mapped beyond email in this config, but can be extended.  
    - Input: Enriched contact data.  
    - Output: HubSpot contact updated or created.  
    - Edge Cases: HubSpot API limits, authentication failures, or email format issues.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                             | Input Node(s)          | Output Node(s)                          | Sticky Note                                                                                                  |
|-------------------------|-----------------------------|---------------------------------------------|-----------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------|
| ⏰ Run Every Hour         | Schedule Trigger            | Initiates workflow every hour                | None                  | 📄 Read Pending Contacts               | **Step-1**: Automated schedule to trigger workflow hourly.                                                   |
| 📄 Read Pending Contacts | Google Sheets               | Reads "Pending" contacts from Google Sheet  | ⏰ Run Every Hour       | 🧪 Filter Name & Email                 | **Step-2**: Reads rows with Status "Pending" and filters out missing Name or Email.                           |
| 🧪 Filter Name & Email    | Filter                     | Filters contacts missing Name or Email      | 📄 Read Pending Contacts| 🧹 Clean Contact Data                  |                                                                                                              |
| 🧹 Clean Contact Data     | Code (JavaScript)           | Cleans and normalizes contact data           | 🧪 Filter Name & Email  | 🧠 LLM - GPT-4o-mini                   | **Step-3**: Cleans field names and fills defaults for missing optional fields.                               |
| 🧠 LLM - GPT-4o-mini     | LangChain Agent (GPT-4o-mini)| Classifies contact by function, seniority, department | 🧹 Clean Contact Data  | 🧩 Parse or Fallback Role Tags        | **Step-4**: Classifies contact roles using AI with fallback on parse failure; timestamps analysis.           |
| 🧩 Parse or Fallback Role Tags | Code (JavaScript)       | Parses AI output or falls back to keyword matching | 🧠 LLM - GPT-4o-mini   | 📊 Update Contact in Sheet, 🔁 Sync to Airtable, 📬 Push to HubSpot |                                                                                                              |
| 📊 Update Contact in Sheet| Google Sheets               | Updates enriched contacts in Google Sheet   | 🧩 Parse or Fallback Role Tags | 🔁 Sync to Airtable, 📬 Push to HubSpot | **Step-5**: Updates Sheet with function, seniority, confidence; marks status "Reviewed".                      |
| 🔁 Sync to Airtable       | Airtable                    | Upserts enriched contact into Airtable base | 📊 Update Contact in Sheet | 📬 Push to HubSpot                     | **Step-5**: Syncs enriched contact data to Airtable CRM.                                                     |
| 📬 Push to HubSpot        | HubSpot                     | Creates or updates contact in HubSpot CRM   | 🔁 Sync to Airtable    | None                                  | **Step-5**: Pushes enriched contact to HubSpot CRM.                                                          |
| Sticky Note              | Sticky Note                 | Notes for step 1                             | None                  | None                                  | **Step-1**: Automated schedule to trigger the workflow every hour to fetch and process new contacts.         |
| Sticky Note1             | Sticky Note                 | Notes for step 2                             | None                  | None                                  | **Step-2**: Reads rows from Google Sheets where Status = "Pending" and filters missing essential fields.     |
| Sticky Note2             | Sticky Note                 | Notes for step 3                             | None                  | None                                  | **Step-3**: Cleans up field names and fills in defaults (Title, Company, LinkedIn, etc.).                    |
| Sticky Note3             | Sticky Note                 | Notes for step 4                             | None                  | None                                  | **Step-4**: Classifies contact by Function, Seniority, Department using GPT-4o-mini; fallback applied.         |
| Sticky Note4             | Sticky Note                 | Notes for step 5                             | None                  | None                                  | **Step-5**: Distributes enriched contact to Google Sheets, Airtable, and HubSpot.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval: Every 1 hour  
   - This node initiates the workflow on a recurring hourly basis.

2. **Create Google Sheets Node to Read Pending Contacts**  
   - Type: Google Sheets  
   - Operation: Read rows from sheet named "Sheet1" (gid=0)  
   - Document ID: Your Google Sheet ID  
   - Add filter: "Status" column equals "Pending"  
   - Option: Return only the first matching row (adjust as needed)  
   - Connect Schedule Trigger output to this node input.  
   - Configure Google Sheets credentials (OAuth2 or API key with read access).

3. **Create Filter Node to Validate Name and Email**  
   - Type: Filter  
   - Conditions:  
     - Name is not empty  
     - Email is not empty  
   - Connect Google Sheets node output to Filter node input.

4. **Create Code Node to Clean Contact Data**  
   - Type: Code (JavaScript)  
   - Paste the JavaScript code that:  
     - Iterates all input items  
     - Trims whitespace from all keys  
     - Maps fields: name, email, title, company, phone, linkedin, notes  
     - Assigns default values ("Not provided") if optional fields missing  
   - Connect Filter node output to Code node input.

5. **Create LangChain Agent Node with GPT-4o-mini Model**  
   - Type: LangChain Agent  
   - Model: GPT-4o-mini  
   - Prompt: Use the provided prompt instructing the AI to classify contact by function, seniority, department, confidence score, and reasoning in strict JSON format. Include contact fields as variables.  
   - Connect Code node output to LangChain Agent node input.  
   - Configure OpenAI API credentials with access to GPT-4o-mini.

6. **Create Code Node to Parse AI Response or Fallback**  
   - Type: Code (JavaScript)  
   - Paste JavaScript code that:  
     - Parses AI JSON response using regex and JSON.parse  
     - On failure, uses fallback keyword matching on title for function and seniority  
     - Adds fields: function, seniority, department, confidence_score, reasoning, analyzed_at timestamp  
     - Merges enriched data with original cleaned contact data  
   - Connect LangChain Agent node output to this node input.  
   - Also ensure access to original cleaned contact data (via referencing or parallel input).

7. **Create Google Sheets Node to Update Contact**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Sheet: Same Google Sheet and sheet name as read node  
   - Matching column: Email  
   - Columns to update: Email, Status (set "Reviewed"), Function, Seniority, Confidence Score  
   - Connect parse/fallback code node output to this node input.  
   - Use valid Google Sheets credentials.

8. **Create Airtable Node to Sync Enriched Contact**  
   - Type: Airtable  
   - Operation: Upsert  
   - Base ID: Your Airtable Base ID  
   - Table ID: Your Airtable Table ID  
   - Matching column: Email  
   - Map fields: Name, Email, Phone, Title, Company, Function, LinkedIn, Seniority, Department, Confidence Score  
   - Connect Google Sheets update node output to Airtable node input.  
   - Configure Airtable App Token credentials.

9. **Create HubSpot Node to Push Contact**  
   - Type: HubSpot  
   - Operation: Create or Update contact by Email  
   - Email field: Use enriched contact email  
   - Connect Airtable node output to HubSpot node input.  
   - Configure HubSpot App Token credentials.

10. **Connect Workflow**  
    - Connect all nodes as per dependencies:  
      Schedule Trigger → Read Pending Contacts → Filter Name & Email → Clean Contact Data → LLM GPT-4o-mini → Parse or Fallback Role Tags → Update Contact in Sheet → Sync to Airtable → Push to HubSpot.

11. **Add Sticky Notes** (Optional)  
    - Add sticky notes at appropriate workflow points describing steps 1 to 5 as per original for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The AI prompt expects strict JSON format output to reliably parse the classification.                             | Embedded in the LangChain Agent node configuration.                                                   |
| Fallback mechanism ensures contact classification proceeds even if AI response parsing fails, safeguarding flow.| Important for robustness under API errors or unexpected outputs.                                      |
| Google Sheets, Airtable, and HubSpot integrations require valid credentials with appropriate scopes and tokens.  | Credentials must be configured in n8n prior to execution.                                            |
| Workflow designed for periodic batch processing but can be adapted for real-time triggers if needed.              | Schedule trigger can be replaced or supplemented with webhook triggers.                               |
| Airtable and HubSpot nodes use App Token authentication methods; ensure tokens have sufficient permissions.       | See Airtable and HubSpot API documentation for token generation and scope.                            |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.