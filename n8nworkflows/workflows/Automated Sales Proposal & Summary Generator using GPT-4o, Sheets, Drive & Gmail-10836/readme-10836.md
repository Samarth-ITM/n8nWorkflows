Automated Sales Proposal & Summary Generator using GPT-4o, Sheets, Drive & Gmail

https://n8nworkflows.xyz/workflows/automated-sales-proposal---summary-generator-using-gpt-4o--sheets--drive---gmail-10836


# Automated Sales Proposal & Summary Generator using GPT-4o, Sheets, Drive & Gmail

### 1. Workflow Overview

This workflow automates the generation and distribution of personalized sales collateral for booked leads using GPT-4o, Google Sheets, Google Drive, and Gmail. It targets sales and marketing teams who manage lead data in Google Sheets and want to streamline the creation of tailored sales materials and reporting.

The workflow consists of the following logical blocks:

- **1.1 Input Reception & Data Retrieval:** Manual trigger initiates the workflow, fetching lead data from Google Sheets.
- **1.2 Data Validation & Filtering:** Validates lead email addresses and filters for leads with “BOOKED” status; logs invalid leads separately.
- **1.3 AI-Driven Sales Collateral Generation:** Uses GPT-4o (via Azure OpenAI) to generate structured sales collateral per booked lead (summary, one-pager, proposal).
- **1.4 Collateral Parsing and File Conversion:** Parses AI JSON output and converts it into formatted text report files.
- **1.5 File Upload & Lead Record Update:** Uploads generated collateral files to Google Drive, matches uploaded files with lead data, and updates Google Sheets with proposal links.
- **1.6 Sales Summary Email Generation and Sending:** Aggregates uploaded file metadata, generates an HTML sales summary email via GPT-4o, and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Retrieval

**Overview:**  
Starts the workflow manually and retrieves lead records from a specific Google Sheets document and tab.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Retrieve Lead Records from Google Sheets

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: Default (no parameters).  
  - Connections: Outputs to "Retrieve Lead Records from Google Sheets".  
  - Potential failures: None; manual trigger.

- **Retrieve Lead Records from Google Sheets**  
  - Type: Google Sheets (Read operation)  
  - Role: Fetches all lead data from a defined sheet/tab.  
  - Configuration: Reads from spreadsheet ID `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`, sheet named "outreach automation" (gid: 46113423).  
  - Credentials: OAuth2 credentials for Google Sheets.  
  - Inputs: Trigger node.  
  - Outputs: Lead data (JSON array).  
  - Edge cases: API quota limits, credential expiry, empty or malformed sheet data.

---

#### 1.2 Data Validation & Filtering

**Overview:**  
Validates each lead’s email format; routes valid leads forward and logs invalid leads in another sheet. Then filters leads to only those with Booking Status = "BOOKED".

**Nodes Involved:**  
- Validate Lead Data Payload  
- Log Invalid Leads to Google Sheets  
- Filter for Booked Leads

**Node Details:**

- **Validate Lead Data Payload**  
  - Type: If Node (conditional branching)  
  - Role: Checks if the Email field matches a regex for valid email format.  
  - Configuration: Condition uses JavaScript regex test on `{{ $json.Email }}`.  
  - Inputs: Lead data from Sheets node.  
  - Outputs: True branch (valid emails) to "Filter for Booked Leads", False branch to "Log Invalid Leads to Google Sheets".  
  - Edge cases: Emails missing or empty, regex failures.

- **Log Invalid Leads to Google Sheets**  
  - Type: Google Sheets (Append operation)  
  - Role: Appends invalid lead records to a separate sheet for cleanup.  
  - Configuration: Intended to append invalid leads but the documentId and sheetName appear empty (likely needs setup).  
  - Credentials: OAuth2.  
  - Inputs: From false branch of validation node.  
  - Outputs: None further.  
  - Edge cases: Empty configuration causing failures; Google API errors.

- **Filter for Booked Leads**  
  - Type: If Node  
  - Role: Filters leads where `Booking Status` equals `"BOOKED"`.  
  - Inputs: Valid leads from validation node.  
  - Outputs: True branch to "Generate Sales Collateral (AI)".  
  - Edge cases: Missing or misspelled Booking Status fields.

---

#### 1.3 AI-Driven Sales Collateral Generation

**Overview:**  
For each booked lead, generates three types of sales collateral using GPT-4o: a short sales summary, a one-pager, and a detailed proposal draft.

**Nodes Involved:**  
- Configure GPT-4o Model (for collateral generation)  
- Generate Sales Collateral (AI)  
- Parse AI JSON Output

**Node Details:**

- **Configure GPT-4o Model**  
  - Type: Langchain Azure OpenAI node (model configuration)  
  - Role: Sets GPT-4o as the AI model for generation.  
  - Credentials: Azure OpenAI API.  
  - Outputs: Connects to the AI generation node.

- **Generate Sales Collateral (AI)**  
  - Type: Langchain Agent Node  
  - Role: Sends a prompt to GPT-4o with lead details to generate sales materials.  
  - Configuration:  
    - Prompt includes lead fields like Company Name, Contact Person, Industry, Booking Status, Client Message.  
    - Instructions specify JSON output with keys: summary, one_pager, proposal.  
    - System message defines tone and task constraints (short, friendly, professional B2B sales materials).  
  - Inputs: Booked leads from filter node.  
  - Outputs: Raw AI JSON text.

- **Parse AI JSON Output**  
  - Type: Code node (JavaScript)  
  - Role: Cleans and parses the AI text output into structured JSON fields (summary, one_pager, proposal).  
  - Key expressions: Uses regex to remove markdown code fences and tries JSON.parse with error fallback.  
  - Inputs: Raw AI output.  
  - Outputs: Clean JSON objects with sales collateral.  
  - Edge cases: Invalid JSON responses from AI, missing fields.

---

#### 1.4 Collateral Parsing and File Conversion

**Overview:**  
Converts the parsed sales collateral JSON into formatted plain text reports, one file per lead.

**Nodes Involved:**  
- Convert Collateral into Text Reports

**Node Details:**

- **Convert Collateral into Text Reports**  
  - Type: Code node (JavaScript)  
  - Role: Constructs a human-readable text report combining summary, one-pager, and proposal sections.  
  - Output: Generates a UTF-8 encoded Buffer containing the report text, with a filename like `Lead_Report_X.txt`.  
  - Inputs: Parsed collateral JSON from previous node.  
  - Outputs: Binary file data usable for upload.  
  - Edge cases: Missing or incomplete collateral fields, encoding issues.

---

#### 1.5 File Upload & Lead Record Update

**Overview:**  
Uploads the generated text reports to a specified Google Drive folder, matches uploaded files back to leads, and updates the lead records in Google Sheets with the proposal file link.

**Nodes Involved:**  
- Upload Sales Collateral to Google Drive  
- Aggregate Uploaded File Metadata  
- Map Uploaded Files with Lead Data  
- Update Lead Record with Proposal Link

**Node Details:**

- **Upload Sales Collateral to Google Drive**  
  - Type: Google Drive Upload node  
  - Role: Uploads each `.txt` report file to the “collatral data” folder (folder ID provided).  
  - Credentials: Google Drive OAuth2.  
  - Outputs: File metadata including webViewLink and webContentLink.  
  - Edge cases: API upload limits, permission errors.

- **Aggregate Uploaded File Metadata**  
  - Type: Code node (JavaScript)  
  - Role: Builds an HTML unordered list of uploaded file names with links and counts total leads processed.  
  - Outputs: JSON with totalLeads, fileList (array), fileListText (HTML snippet).  
  - Inputs: Uploaded file metadata.  
  - Edge cases: Missing links or file names.

- **Map Uploaded Files with Lead Data**  
  - Type: Code node (JavaScript)  
  - Role: Matches each uploaded file with its corresponding lead by index, preparing data to update the sheet.  
  - Outputs: JSON with email, proposal link, file name, company, contact person, job title, timestamp, and skip flag.  
  - Inputs: Uploaded file data and lead data from "Filter for Booked Leads".  
  - Edge cases: Mismatched array lengths, missing emails or links.

- **Update Lead Record with Proposal Link**  
  - Type: Google Sheets (appendOrUpdate)  
  - Role: Updates lead rows in Google Sheets matching on Email, inserting the Proposal Link.  
  - Configuration: Maps Email and Proposal Link columns; uses "outreach automation" sheet.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Matching failures, API errors.

---

#### 1.6 Sales Summary Email Generation and Sending

**Overview:**  
Generates a professional HTML-formatted sales summary email with total booked leads and uploaded proposal file links, then sends it via Gmail to the sales inbox.

**Nodes Involved:**  
- Generate Sales Summary Email  
- Send Sales Summary Email via Gmail

**Node Details:**

- **Generate Sales Summary Email**  
  - Type: Langchain Agent Node  
  - Role: Uses GPT-4o to create an HTML snippet summarizing sales results and file uploads.  
  - Inputs: Aggregated uploaded file metadata (total leads, file list in HTML).  
  - Configuration: System message enforces clean HTML output (no full HTML tags), professional formatting with `<h2>`, `<ul>`, `<p>`.  
  - Outputs: HTML string for email body.  
  - Edge cases: AI output format inconsistencies.

- **Send Sales Summary Email via Gmail**  
  - Type: Gmail node (send email)  
  - Role: Sends the generated HTML email to a specified recipient (`newscctv22@gmail.com`).  
  - Configuration: Subject line "Sales Collateral Summary", message body from AI output.  
  - Credentials: Gmail OAuth2.  
  - Edge cases: Gmail quota, authentication failures.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                                  | Input Node(s)                  | Output Node(s)                                 | Sticky Note                                                                                          |
|----------------------------------|-------------------------------------|-------------------------------------------------|-------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                      | Manual start of workflow                         | None                          | Retrieve Lead Records from Google Sheets       |                                                                                                    |
| Retrieve Lead Records from Google Sheets | Google Sheets (Read)               | Fetch lead data from sheet                        | When clicking ‘Execute workflow’ | Validate Lead Data Payload                      | ## 🧾 Retrieve Lead Records from Google Sheets: Fetches lead data (Company Name, Email, Booking Status, etc.) from the sheet. Acts as the workflow’s entry point. |
| Validate Lead Data Payload        | If Node                            | Validate email format                             | Retrieve Lead Records from Google Sheets | Filter for Booked Leads, Log Invalid Leads to Google Sheets | ## 🧩 Validate Lead Data Payload: Checks if lead records have a valid email address. Valid leads proceed, invalid are logged. |
| Log Invalid Leads to Google Sheets | Google Sheets (Append)              | Log invalid or malformed leads                    | Validate Lead Data Payload (False) | None                                         | ## ⚠️ Log Invalid Leads to Google Sheets: Stores malformed or missing lead entries; prevents workflow interruption. |
| Filter for Booked Leads           | If Node                            | Filter leads with Booking Status = "BOOKED"      | Validate Lead Data Payload (True) | Generate Sales Collateral (AI)                  | ## 🎯 Filter for Booked Leads: Filters only leads marked as “BOOKED”; ensures collateral generation only for confirmed clients. |
| Configure GPT-4o Model            | Langchain Azure OpenAI Model Config | Set GPT-4o model for sales collateral generation | Filter for Booked Leads          | Generate Sales Collateral (AI)                  | ## ⚙️ Configure GPT-4o Model: Connects Azure OpenAI GPT-4o for downstream nodes; used for sales collateral and summary generation. |
| Generate Sales Collateral (AI)   | Langchain Agent                    | Generate sales collateral (summary, one-pager, proposal) | Configure GPT-4o Model         | Parse AI JSON Output                            | ## 🧠 Generate Sales Collateral (AI): GPT-4o generates 3 structured outputs per booked lead as JSON. |
| Parse AI JSON Output              | Code                              | Parse and clean AI JSON output                    | Generate Sales Collateral (AI)  | Convert Collateral into Text Reports            | ## 🧹 Parse AI JSON Output: Cleans and parses GPT-4o output into usable JSON keys; handles invalid responses. |
| Convert Collateral into Text Reports | Code                              | Convert parsed collateral into text report files | Parse AI JSON Output            | Upload Sales Collateral to Google Drive         | ## 📄 Convert Collateral into Text Reports: Converts parsed AI output into `.txt` files formatted for readability. |
| Upload Sales Collateral to Google Drive | Google Drive Upload               | Upload sales collateral files to Drive            | Convert Collateral into Text Reports | Aggregate Uploaded File Metadata, Map Uploaded Files with Lead Data | ## ☁️ Upload Sales Collateral to Google Drive: Uploads `.txt` files to Drive folder; returns file links. |
| Aggregate Uploaded File Metadata | Code                              | Aggregate file metadata into HTML list and count | Upload Sales Collateral to Google Drive | Generate Sales Summary Email                     | ## 🗂️ Aggregate Uploaded File Metadata: Builds HTML-formatted list of uploaded collateral files for email generation. |
| Generate Sales Summary Email     | Langchain Agent                    | Generate HTML sales summary email snippet         | Aggregate Uploaded File Metadata | Send Sales Summary Email via Gmail               | ## ✉️ Generate Sales Summary Email (AI): Uses GPT-4o to create HTML-formatted sales summary email with lead count and file links. |
| Send Sales Summary Email via Gmail | Gmail Send Email                   | Send the sales summary email                       | Generate Sales Summary Email    | None                                            | ## 📧 Send Sales Summary Email via Gmail: Delivers AI-generated HTML report to marketing inbox. |
| Map Uploaded Files with Lead Data | Code                              | Match uploaded files with leads; prepare update data | Upload Sales Collateral to Google Drive | Update Lead Record with Proposal Link           | ## 🔗 Map Uploaded Files with Lead Data: Matches files with leads by index; prepares data for Sheets update. |
| Update Lead Record with Proposal Link | Google Sheets (AppendOrUpdate)    | Update lead rows with proposal file links          | Map Uploaded Files with Lead Data | None                                            | ## ✅ Update Lead Record with Proposal Link: Updates lead sheet with proposal links for traceability. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Manually start the workflow.

2. **Add Google Sheets Node to Retrieve Leads**  
   - Type: Google Sheets (Read operation)  
   - Parameters:  
     - Document ID: `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`  
     - Sheet Name: `outreach automation` (gid: 46113423)  
   - Credentials: Google Sheets OAuth2 for your account  
   - Connect Manual Trigger output to this node.

3. **Add If Node to Validate Email Format**  
   - Condition: Check if `{{ $json.Email }}` matches regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`  
   - True branch: proceed; False branch: log invalid leads.

4. **Add Google Sheets Node to Log Invalid Leads**  
   - Operation: Append  
   - Configure target spreadsheet and sheet for invalid leads logging (set Document ID and Sheet Name)  
   - Credentials: Google Sheets OAuth2  
   - Connect False branch from validation node here.

5. **Add If Node to Filter Booked Leads**  
   - Condition: `{{ $json["Booking Status"] }}` equals `"BOOKED"`  
   - Connect True branch to next AI generation node.

6. **Add Langchain Azure OpenAI Node to Configure GPT-4o Model**  
   - Model: `gpt-4o`  
   - Credentials: Azure OpenAI API  
   - Connect output from booked leads filter node.

7. **Add Langchain Agent Node to Generate Sales Collateral**  
   - Input Text: Template including lead fields (Company Name, Contact Person, Job Title, Email, Phone, Industry, Location, Lead Source, Booking Status, ClientMessage)  
   - Instructions: Generate JSON with keys `summary`, `one_pager`, `proposal` (see prompt in workflow)  
   - System Message: Friendly, professional B2B sales assistant tone  
   - Connect to GPT-4o model node.

8. **Add Code Node to Parse AI JSON Output**  
   - JavaScript: Clean markdown code fences and parse JSON; fallback on error  
   - Connect from AI generation node.

9. **Add Code Node to Convert Collateral into Text Reports**  
   - JavaScript: Format the parsed JSON into plain text reports including Summary, One-Pager, and Proposal sections.  
   - Output binary data with UTF-8 encoding and assign filenames `Lead_Report_X.txt`.  
   - Connect from JSON parse node.

10. **Add Google Drive Node to Upload Text Files**  
    - Operation: Upload  
    - Folder ID: Set to your Drive folder for collateral (e.g., `1WVD9VVLyV34NCynIizt9R2xnTP5SsTfj`)  
    - Credentials: Google Drive OAuth2  
    - Connect from text reports node.

11. **Add Code Node to Aggregate Uploaded File Metadata**  
    - JavaScript: Create HTML list of uploaded files with links, count total leads.  
    - Connect from Google Drive upload node.

12. **Add Langchain Agent Node to Generate Sales Summary Email**  
    - Input Text: Include total leads and HTML file list.  
    - System Message: Return clean, valid HTML snippet (no <html> or <body>) with `<h2>`, `<ul>`, `<p>` tags in professional style.  
    - Connect from aggregate metadata node.

13. **Add Gmail Node to Send Sales Summary Email**  
    - Send To: Your marketing or sales email (e.g., `newscctv22@gmail.com`)  
    - Subject: “Sales Collateral Summary”  
    - Message: Use AI-generated HTML output  
    - Credentials: Gmail OAuth2  
    - Connect from sales summary email node.

14. **Add Code Node to Map Uploaded Files to Leads**  
    - JavaScript: Match uploaded file links with lead emails and info by index, prepare data for sheet update.  
    - Connect from Google Drive upload node (parallel to aggregation node).

15. **Add Google Sheets Node to Update Lead Records**  
    - Operation: Append or Update  
    - Match on “Email” column  
    - Update “Proposal Link” column with uploaded file links  
    - Credentials: Google Sheets OAuth2  
    - Connect from file-to-lead mapping node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Automates personalized sales collateral creation using advanced AI and integrates tightly with Google Workspace tools.                   | General workflow purpose.                                                                               |
| Uses GPT-4o via Azure OpenAI—requires Azure OpenAI subscription and proper API credentials.                                                | Azure OpenAI GPT-4o model usage.                                                                        |
| Google Sheets and Drive credentials must have read/write access to the specified spreadsheets and folder.                                  | Google OAuth2 credential setup required.                                                                |
| Gmail node requires OAuth2 credentials with permissions to send emails on behalf of the account.                                           | Gmail OAuth2 setup.                                                                                      |
| Parsing AI JSON output includes error handling for malformed or incomplete AI responses to avoid workflow failures.                        | Robustness in AI output handling.                                                                       |
| Workflow is designed for batch processing multiple leads; index-based file-to-lead mapping assumes order is preserved through the flow.   | Important for data integrity and matching.                                                             |
| Sticky notes in the workflow provide detailed explanations for each logical block and node function for maintainability and documentation. | Useful for developers extending or auditing the workflow.                                              |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.