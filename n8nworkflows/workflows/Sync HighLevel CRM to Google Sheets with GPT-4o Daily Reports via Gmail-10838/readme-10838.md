Sync HighLevel CRM to Google Sheets with GPT-4o Daily Reports via Gmail

https://n8nworkflows.xyz/workflows/sync-highlevel-crm-to-google-sheets-with-gpt-4o-daily-reports-via-gmail-10838


# Sync HighLevel CRM to Google Sheets with GPT-4o Daily Reports via Gmail

### 1. Workflow Overview

This workflow automates the synchronization of opportunity data from the HighLevel CRM system into a Google Sheets spreadsheet and generates a daily HTML summary report sent via Gmail. It leverages AI (GPT-4o via Azure OpenAI) to create a clean, well-formatted sales report based solely on the CRM data, enhancing reporting accuracy and presentation.

The workflow comprises the following logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetch opportunity records from HighLevel CRM.
- **1.3 Data Validation:** Check for valid opportunity records.
- **1.4 Data Logging:** Log invalid records for auditing.
- **1.5 Data Extraction:** Extract key fields from raw CRM data.
- **1.6 Data Normalization:** Standardize and normalize opportunity field names and values.
- **1.7 Data Upsert:** Update or append normalized data into Google Sheets.
- **1.8 Data Aggregation:** Merge all normalized opportunities into a single JSON array.
- **1.9 AI Report Generation:** Use GPT-4o to generate a daily HTML summary report.
- **1.10 Email Dispatch:** Send the generated HTML report by Gmail to the sales team.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** The workflow starts on manual execution, allowing user control over when the sync runs.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’
- **Node Details:**  
  - Type: Manual Trigger  
  - Configuration: Default manual trigger without parameters  
  - Input: None (manual start)  
  - Output: Initiates data fetching  
  - Edge Cases: None, but user must manually trigger to start the process  

---

#### 1.2 Data Retrieval

- **Overview:** Fetches the latest opportunity records from HighLevel CRM (limited to 5 by default).
- **Nodes Involved:**  
  - Fetch Opportunities from HighLevel CRM
- **Node Details:**  
  - Type: HighLevel CRM node  
  - Configuration: Resource = `opportunity`, Operation = `getAll`, Limit = 5  
  - Inputs: Trigger from manual node  
  - Outputs: Raw list of opportunities with contact, company, source, and stage info  
  - Credentials: OAuth2 with HighLevel account  
  - Edge Cases: API failure, authentication errors, empty data sets  
- **Sticky Note:** Explains purpose and data scope for this node.

---

#### 1.3 Data Validation

- **Overview:** Validates that each opportunity record has a non-empty `id` field to ensure data integrity.
- **Nodes Involved:**  
  - Validate Opportunity Data Payload
- **Node Details:**  
  - Type: If node (conditional check)  
  - Configuration: Checks if `id` field is not empty (`{{$json.id}} !== ""`)  
  - Inputs: Raw opportunities from HighLevel node  
  - Outputs:  
    - True branch: Valid records forwarded for extraction  
    - False branch: Invalid records sent for logging  
  - Edge Cases: Unexpected data structure, missing `id` field, empty records  
- **Sticky Note:** Details validation logic and branching.

---

#### 1.4 Data Logging

- **Overview:** Appends invalid opportunity records into a dedicated Google Sheets tab for review and data cleaning.
- **Nodes Involved:**  
  - Log Invalid Opportunities to Google Sheets
- **Node Details:**  
  - Type: Google Sheets (append operation)  
  - Configuration: Appends to a specified sheet and document (sheet name and document ID expected to be set)  
  - Inputs: Invalid opportunities from validation node  
  - Credentials: Google Sheets OAuth2 configured for “automations@techdome.ai”  
  - Edge Cases: Authentication failure, quota limits, malformed data  
- **Sticky Note:** Emphasizes use for tracking incomplete or invalid CRM data.

---

#### 1.5 Data Extraction

- **Overview:** Extracts and simplifies essential fields from raw CRM opportunity data to a consistent JSON structure.
- **Nodes Involved:**  
  - Extract Key Fields from HighLevel Data
- **Node Details:**  
  - Type: Code node (JavaScript)  
  - Configuration: Maps each item extracting fields like `id`, `name`, `company`, `email`, `phone`, `source`, `assignedTo`, pipeline and stage IDs, `tags`, monetary value, and timestamps  
  - Inputs: Validated raw opportunity data  
  - Outputs: Cleaned opportunity objects with simplified fields  
  - Edge Cases: Missing nested fields, null or undefined values  
- **Sticky Note:** Lists all extracted fields and purpose.

---

#### 1.6 Data Normalization

- **Overview:** Normalizes field names and fills missing contact details from nested objects to ensure consistent data structure for Google Sheets update.
- **Nodes Involved:**  
  - Normalize Opportunity Structure
- **Node Details:**  
  - Type: Code node (JavaScript, runs once per item)  
  - Configuration: Ensures fields like `company`, `email`, and `phone` are present, falling back on nested contact data if needed.  
  - Inputs: Extracted key fields  
  - Outputs: Fully normalized opportunity data objects  
  - Edge Cases: Partial data, inconsistent field availability  
- **Sticky Note:** Explains normalization logic and consistency goals.

---

#### 1.7 Data Upsert

- **Overview:** Appends or updates the normalized opportunity records into a Google Sheets sheet named “ghl database” in the `sample_leads_50` spreadsheet, matching on the `id` field.
- **Nodes Involved:**  
  - Update Opportunity Records in Google Sheets
- **Node Details:**  
  - Type: Google Sheets node (appendOrUpdate operation)  
  - Configuration:  
    - Document ID: Google Sheets spreadsheet ID (preset)  
    - Sheet Name: “ghl database” tab (preset)  
    - Matching column: `id` used to update existing rows or append new ones  
    - Mapped fields: id, name, company, email, phone, source, assignedTo, pipelineId, stageId, monetaryValue, createdAt, updatedAt, lastStageChangeAt  
  - Inputs: Normalized opportunity data  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Authentication, permission errors, concurrency issues, schema mismatch  
- **Sticky Note:** Describes upsert mechanism maintaining CRM and sheet sync.

---

#### 1.8 Data Aggregation

- **Overview:** Merges all normalized opportunity records into a single JSON array under the key `opportunities` to prepare for AI processing.
- **Nodes Involved:**  
  - Merge All Opportunities into Single JSON Array
- **Node Details:**  
  - Type: Code node (JavaScript)  
  - Configuration: Collects all incoming items’ JSON data into one array and returns a single item with a field `opportunities` containing that array  
  - Inputs: Normalized opportunities  
  - Outputs: One JSON object wrapping all opportunities  
  - Edge Cases: No input items, large datasets causing memory issues  
- **Sticky Note:** Emphasizes preparation for AI report generation.

---

#### 1.9 AI Report Generation

- **Overview:** Uses GPT-4o model (Azure OpenAI via Langchain) to generate a clean, Gmail-friendly HTML report summarizing the daily opportunities.
- **Nodes Involved:**  
  - Configure GPT-4o Model  
  - Generate Daily Opportunity Summary Report
- **Node Details:**  
  - **Configure GPT-4o Model:**  
    - Type: Langchain Azure OpenAI chat model  
    - Configuration: Model set to GPT-4o, credentials to Azure OpenAI account  
    - Outputs: Provides AI model to the agent node  
  - **Generate Daily Opportunity Summary Report:**  
    - Type: Langchain Agent node  
    - Configuration:  
      - Receives merged JSON array as input  
      - System message enforces strict rules: use only provided data, no invention, output valid HTML with specific structure and styling  
      - Prompt includes detailed instructions for table columns, styling, and null value handling  
    - Inputs: Merged JSON opportunities and GPT-4o model reference  
    - Outputs: HTML text report in the `output` field  
  - Edge Cases: API rate limits, data format issues, incomplete or null data, unexpected AI output  
- **Sticky Note:** Describes strict production of clean HTML report by GPT-4o.

---

#### 1.10 Email Dispatch

- **Overview:** Sends the AI-generated daily opportunity HTML report via Gmail to a predefined email address.
- **Nodes Involved:**  
  - Send Daily Opportunity Summary via Gmail
- **Node Details:**  
  - Type: Gmail node  
  - Configuration:  
    - Recipient: newscctv22@gmail.com (hardcoded)  
    - Subject: “Daily Opportunity Report – Summary of New Leads”  
    - Message body: AI-generated HTML report content  
  - Credentials: Gmail OAuth2 account  
  - Inputs: HTML report from AI node  
  - Outputs: Email sent confirmation  
  - Edge Cases: Authentication failure, Gmail quota/rate limits, malformed HTML causing rendering issues  
- **Sticky Note:** Notes Gmail-friendly formatting and automated delivery.

---

### 3. Summary Table

| Node Name                               | Node Type                              | Functional Role                             | Input Node(s)                      | Output Node(s)                             | Sticky Note                                                                                                      |
|----------------------------------------|--------------------------------------|--------------------------------------------|----------------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’       | Manual Trigger                       | Workflow start trigger                      | None                             | Fetch Opportunities from HighLevel CRM    |                                                                                                                 |
| Fetch Opportunities from HighLevel CRM | HighLevel CRM node                   | Fetch raw opportunities from CRM            | When clicking ‘Execute workflow’ | Validate Opportunity Data Payload          | ## 🔗 Fetch Opportunities from HighLevel CRM  \nPulls recent opportunity data (limit: 5 by default) from your HighLevel CRM account.  \nIncludes contact details, company, source, and stage info.  \nThis is the main data entry point. |
| Validate Opportunity Data Payload      | If node                             | Validate presence of opportunity `id`       | Fetch Opportunities from HighLevel CRM | Extract Key Fields from HighLevel Data, Log Invalid Opportunities to Google Sheets | ## 🧩 Validate Opportunity Data Payload  \nChecks whether each opportunity record has a valid “id” field.  \n✅ Valid → moves to extraction  \n❌ Invalid → logged into Google Sheets for review.  |
| Log Invalid Opportunities to Google Sheets | Google Sheets (append)               | Log invalid records for auditing             | Validate Opportunity Data Payload (False branch) | None                                      | ## ⚠️ Log Invalid Opportunities  \nStores invalid or incomplete opportunities in a dedicated Google Sheet.  \nUseful for cleaning CRM data and tracking integration issues.                                |
| Extract Key Fields from HighLevel Data | Code                                | Extract key fields from raw data             | Validate Opportunity Data Payload (True branch) | Normalize Opportunity Structure            | ## 🧾 Extract Key Fields from HighLevel Data  \nExtracts only the essential details from CRM opportunities, including:  \n- id, name, company, email, phone  \n- source, assignedTo, pipelineId, stageId  \n- tags, monetary value, timestamps  \nSimplifies data for consistency before sheet update.  |
| Normalize Opportunity Structure        | Code                                | Normalize and standardize field names       | Extract Key Fields from HighLevel Data | Update Opportunity Records in Google Sheets, Merge All Opportunities into Single JSON Array | ## ⚙️ Normalize Opportunity Structure  \nEnsures every opportunity has standardized field names.  \nFills missing contact info, adjusts IDs, and prepares for sheet update and reporting.                |
| Update Opportunity Records in Google Sheets | Google Sheets (appendOrUpdate)       | Upsert normalized opportunities into sheet  | Normalize Opportunity Structure  | None                                      | ## 📋 Update Opportunity Records in Google Sheets  \nUpserts (append or update) opportunity data into the “ghl database” tab inside your master sheet (`sample_leads_50`).  \nMatching key: `id`.  \nKeeps CRM and sheet data fully synced. |
| Merge All Opportunities into Single JSON Array | Code                                | Aggregate all opportunities into one JSON   | Normalize Opportunity Structure  | Generate Daily Opportunity Summary Report  | ## 🧠 Merge All Opportunities  \nCombines all normalized records into a single JSON array under the field `opportunities`.  \nPrepares data for AI processing and HTML report generation.            |
| Configure GPT-4o Model                 | Azure OpenAI (Langchain)             | Configure GPT-4o AI model                    | None                             | Generate Daily Opportunity Summary Report  | ## ⚙️ Configure GPT-4o Model  \nConnects to the Azure OpenAI GPT-4o model.  \nUsed as the AI engine for generating HTML reports and summaries.                                                |
| Generate Daily Opportunity Summary Report | Langchain Agent                      | Generate HTML summary report using GPT-4o   | Merge All Opportunities into Single JSON Array, Configure GPT-4o Model | Send Daily Opportunity Summary via Gmail | ## 🧠 Generate Daily Opportunity Summary Report  \nGPT-4o analyzes the merged opportunities JSON and generates a clean HTML report.  \nStructure:\n- <h2> title: “Daily Opportunity Summary”  \n- Description paragraph  \n- Styled table with:  \n  Name, Company, Email, Phone, Source, Pipeline ID, Stage ID, Value, Created At  \n- Nulls replaced with “-”  \n- Gmail-friendly design (bordered table, padded cells) |
| Send Daily Opportunity Summary via Gmail | Gmail                              | Send HTML report email to sales team         | Generate Daily Opportunity Summary Report | None                                      | ## 📧 Send Daily Opportunity Summary via Gmail  \nSends the AI-generated HTML report to the sales inbox.  \nSubject: “Daily Opportunity Report – Summary of New Leads.”  \nAutomatically formatted for Gmail rendering. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Name: `When clicking ‘Execute workflow’`
   - Type: Manual Trigger
   - No special parameters.
   - This node initiates the workflow.

2. **Add HighLevel CRM Node:**
   - Name: `Fetch Opportunities from HighLevel CRM`
   - Type: HighLevel node
   - Set Resource: `opportunity`
   - Operation: `getAll`
   - Limit: 5 (default)
   - Connect input from Manual Trigger node.
   - Configure OAuth2 credentials for HighLevel account.

3. **Add If Node for Validation:**
   - Name: `Validate Opportunity Data Payload`
   - Type: If node
   - Condition: Check if `id` field in each item is not empty (`{{$json.id}} !== ""`)
   - Connect input from HighLevel node.
   - Configure two outputs:
     - True branch: valid records
     - False branch: invalid records

4. **Add Google Sheets Node to Log Invalid Data:**
   - Name: `Log Invalid Opportunities to Google Sheets`
   - Type: Google Sheets node
   - Operation: `append`
   - Configure Spreadsheet ID and Sheet Name for invalid records logging.
   - Input from the False branch of Validation node.
   - Use Google Sheets OAuth2 credentials.

5. **Add Code Node to Extract Key Fields:**
   - Name: `Extract Key Fields from HighLevel Data`
   - Type: Code node (JavaScript)
   - Input from True branch of Validation node.
   - Code sample:
     ```js
     return items.map(item => {
       const o = item.json;
       return {
         json: {
           id: o.id,
           name: o.name,
           company: o.contact?.companyName || null,
           email: o.contact?.email || null,
           phone: o.contact?.phone || null,
           source: o.source || null,
           assignedTo: o.assignedTo || null,
           pipelineId: o.pipelineId,
           stageId: o.pipelineStageId,
           tags: o.contact?.tags?.join(", ") || null,
           monetaryValue: o.monetaryValue || 0,
           createdAt: o.createdAt,
           updatedAt: o.updatedAt,
           lastStageChangeAt: o.lastStageChangeAt
         }
       };
     });
     ```

6. **Add Code Node to Normalize Data:**
   - Name: `Normalize Opportunity Structure`
   - Type: Code node (JavaScript)
   - Mode: Run once per item
   - Input: Extracted key fields
   - Code sample:
     ```js
     const op = item.json;
     return {
       json: {
         id: op.id,
         name: op.name,
         company: op.company || op.contact?.companyName || null,
         email: op.email || op.contact?.email || null,
         phone: op.phone || op.contact?.phone || null,
         source: op.source,
         assignedTo: op.assignedTo,
         pipelineId: op.pipelineId,
         stageId: op.stageId || op.pipelineStageId,
         tags: op.tags,
         monetaryValue: op.monetaryValue,
         createdAt: op.createdAt,
         updatedAt: op.updatedAt,
         lastStageChangeAt: op.lastStageChangeAt
       }
     };
     ```

7. **Add Google Sheets Node to Upsert Data:**
   - Name: `Update Opportunity Records in Google Sheets`
   - Type: Google Sheets node
   - Operation: `appendOrUpdate`
   - Configure Spreadsheet ID and Sheet Name: use the “sample_leads_50” spreadsheet and “ghl database” tab.
   - Define matching column: `id`
   - Map fields as per normalized data keys (`id`, `name`, `company`, `email`, `phone`, `source`, `assignedTo`, `pipelineId`, `stageId`, `monetaryValue`, `createdAt`, `updatedAt`, `lastStageChangeAt`)
   - Input from Normalize Opportunity Structure node.
   - Use Google Sheets OAuth2 credentials.

8. **Add Code Node to Merge All Opportunities:**
   - Name: `Merge All Opportunities into Single JSON Array`
   - Type: Code node (JavaScript)
   - Input from Normalize Opportunity Structure node.
   - Code sample:
     ```js
     const merged = items.map(i => i.json);
     return [{
       json: {
         opportunities: merged
       }
     }];
     ```

9. **Add Azure OpenAI Node for GPT-4o Model:**
   - Name: `Configure GPT-4o Model`
   - Type: Langchain Azure OpenAI Chat model
   - Set Model: `gpt-4o`
   - Configure Azure OpenAI credentials.

10. **Add Langchain Agent Node for Report Generation:**
    - Name: `Generate Daily Opportunity Summary Report`
    - Type: Langchain Agent
    - Input from Merged Opportunities node and GPT-4o model node
    - Configure prompt:
      - Pass the `opportunities` JSON stringified with indentation.
      - System message: instruct strict use of input data only, no invention.
      - Task: generate a clean HTML report with `<h2>Daily Opportunity Summary</h2>`, description paragraph, and styled table including columns: Name, Company, Email, Phone, Source, Pipeline ID, Stage ID, Value, Created At.
      - Replace null values with “-”.
      - Ensure Gmail-friendly HTML (borders, padding).
    - Output: HTML content as string.

11. **Add Gmail Node to Send Report:**
    - Name: `Send Daily Opportunity Summary via Gmail`
    - Type: Gmail node
    - Recipient: `newscctv22@gmail.com`
    - Subject: “Daily Opportunity Report – Summary of New Leads”
    - Message body: from AI-generated HTML report output
    - Use Gmail OAuth2 credentials.
    - Input from AI Report Generation node.

12. **Connect Nodes According to Data Flow:**
    - Manual Trigger → Fetch Opportunities from HighLevel
    - Fetch Opportunities → Validate Opportunity Data Payload
    - Validate True → Extract Key Fields → Normalize → Update Google Sheets
    - Normalize → Merge All Opportunities
    - Merge → Generate Daily Report (with GPT model)
    - Generate Report → Send Gmail
    - Validate False → Log Invalid Opportunities to Google Sheets

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed to synchronize CRM data with Google Sheets and produce AI-powered daily HTML email summaries, improving sales data visibility and reporting automation.                                                                                                                                      | Workflow purpose and overview.                                                                  |
| GPT-4o is used strictly with user-provided JSON data to avoid hallucinations, ensuring accurate report generation based on real CRM data only.                                                                                                                                                               | AI model usage and prompt design.                                                              |
| Google Sheets upsert uses the `id` field as a key to maintain synchronization between CRM and sheet without duplicates.                                                                                                                                                                                    | Data integrity and synchronization.                                                            |
| Gmail node sends the report with HTML formatting optimized for Gmail clients, ensuring readability and usability by sales teams.                                                                                                                                                                            | Email formatting considerations.                                                               |
| HighLevel CRM node requires OAuth2 authentication with appropriate API access to fetch opportunity data.                                                                                                                                                                                                   | HighLevel API credential setup.                                                                 |
| For Google Sheets and Gmail nodes, OAuth2 credentials must be set up for the respective accounts with adequate permission scopes.                                                                                                                                                                          | OAuth2 credential requirements.                                                                |
| The workflow runs on manual trigger but can be scheduled or triggered by other means as needed for automation.                                                                                                                                                                                             | Trigger flexibility.                                                                            |
| See official n8n docs for additional node configuration options and limits: https://docs.n8n.io/nodes/                                                                                                                                                                                                     | Official n8n documentation.                                                                    |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow built with n8n, an integration and automation tool. The content complies strictly with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.