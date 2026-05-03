Extract Contacts from Business Cards to Google Sheets With GPT4o

https://n8nworkflows.xyz/workflows/extract-contacts-from-business-cards-to-google-sheets-with-gpt4o-6840


# Extract Contacts from Business Cards to Google Sheets With GPT4o

### 1. Workflow Overview

This workflow automates the extraction of detailed contact information from uploaded business card images and logs the structured data into a Google Sheet. It is designed for sales, recruitment, and event teams who collect business cards and need a fast, accurate way to digitize contact details without manual data entry.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Storage**  
  Handles user form submission of business card images and uploads the files to a designated Google Drive folder for archiving.

- **1.2 AI Processing and Contact Extraction**  
  Uses a smart AI agent with OCR and GPT-4o to analyze the images, extract contact details, and parse the output into structured JSON.

- **1.3 Data Transformation and Validation**  
  Cleans and formats the extracted data, removes invalid or empty records, and prepares contact info for insertion.

- **1.4 Data Appending to Google Sheets**  
  Appends the validated contact records as new rows in a Google Sheet for centralized tracking and easy access.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Storage

**Overview:**  
This block receives business card images uploaded via a user-facing form and archives the files to Google Drive.

**Nodes Involved:**  
- On form submission  
- Upload file

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing user uploads via a form titled "Name Card Uploader Form" accepting JPG/PNG files.  
  - *Config:* Single file upload allowed, with clear form description guiding users to upload business card images.  
  - *Connections:* Outputs to the AI processing node “Neural assistant for contact extraction” and the "Upload file" node.  
  - *Edge Cases:* File type restrictions; missing or corrupted files; large file size may cause timeout or upload errors.

- **Upload file**  
  - *Type:* Google Drive node  
  - *Role:* Saves the uploaded image files to a specific Google Drive folder (SmartSales) with timestamped filenames for archive and audit.  
  - *Config:* Uses OAuth2 credentials for Google Drive, targets folder ID "1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI".  
  - *Input:* Receives file binary data from the form submission node.  
  - *Output:* None connected downstream (archive-only).  
  - *Edge Cases:* Google Drive quota limits; auth token expiry; file naming conflicts.

---

#### 1.2 AI Processing and Contact Extraction

**Overview:**  
This block leverages a LangChain AI agent using GPT-4o with OCR capabilities to analyze the uploaded images and extract structured contact data.

**Nodes Involved:**  
- Neural assistant for contact extraction  
- GPT4o  
- Structured Output Parser

**Node Details:**

- **Neural assistant for contact extraction**  
  - *Type:* LangChain Agent node  
  - *Role:* Core AI logic that processes images and extracts all relevant contact fields such as names, job titles, phones, emails, companies, websites, addresses, LinkedIn, and QR code contents.  
  - *Config:*  
    - Prompt instructs the agent to ignore decorative/branding elements and focus solely on business contact info.  
    - System message sets expectations for logical reading order and output cleanliness for downstream parsing.  
  - *Input:* Receives image data from form submission.  
  - *Output:* Produces raw AI output to be parsed.  
  - *Edge Cases:* OCR failures on low-quality images; model timeouts; unexpected output formats.

- **GPT4o**  
  - *Type:* LangChain OpenAI Chat Model node  
  - *Role:* Language model engine (GPT-4o) powering the AI agent for understanding and extracting contact info.  
  - *Config:* Uses OpenAI API credentials; model set explicitly to “gpt-4o”.  
  - *Input:* Connected as language model backend for the AI agent node.  
  - *Edge Cases:* API rate limits; auth failures; network issues.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses raw AI text output into structured JSON, matching a schema example that includes fields like name, job_title, company, phone, email, website, address, linkedin.  
  - *Config:* JSON schema example provided to ensure consistent output format.  
  - *Input:* Parses output from the AI agent.  
  - *Output:* Clean structured JSON for downstream processing.  
  - *Edge Cases:* Parsing errors if AI output deviates from expected schema; malformed JSON.

---

#### 1.3 Data Transformation and Validation

**Overview:**  
Transforms the structured JSON data by cleaning fields (e.g., removing "+" from phone numbers), normalizes emails, and filters out incomplete records missing a name.

**Nodes Involved:**  
- Transform Output  
- Filter bad data

**Node Details:**

- **Transform Output**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Iterates over all extracted contact entries, cleans phone numbers by removing “+” signs, converts emails to lowercase, and prepares an object matching Google Sheet columns.  
  - *Key Logic:*  
    ```js
    const cleanedPhone = contact.phone ? contact.phone.replace(/\+/g, '') : '';
    output.push({ Name: ..., Phone: cleanedPhone, Email: contact.email.toLowerCase(), ... });
    ```  
  - *Input:* Structured JSON array from the output parser.  
  - *Output:* Array of cleaned contact objects.  
  - *Edge Cases:* Missing fields; null or undefined values; malformed input JSON.

- **Filter bad data**  
  - *Type:* Filter node  
  - *Role:* Removes any contact record where the Name field is empty, ensuring only meaningful contacts proceed.  
  - *Config:* Condition: Name not empty (strict string validation).  
  - *Input:* Cleaned contacts from the code node.  
  - *Output:* Only valid contacts passed to Google Sheets.  
  - *Edge Cases:* Records with empty or whitespace-only names filtered out; potential data loss if name extraction failed.

---

#### 1.4 Data Appending to Google Sheets

**Overview:**  
Appends validated contact records as new rows into a predefined Google Sheet for centralized tracking.

**Nodes Involved:**  
- Add contact to tracking sheet

**Node Details:**

- **Add contact to tracking sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends rows to a target Google Sheet (document ID and sheet name specified) mapping input JSON fields to columns like Name, JobTitle, Company, Phone, Email, Website, Address, LinkedIn.  
  - *Config:*  
    - Operation: Append  
    - Mapping Mode: Auto map input data to sheet columns  
    - Google Sheets OAuth2 credentials configured  
    - Document ID: `1cQtyFzPqq7h9sUsfKj1miAK5gA3de7MKya8HRLD_Pzw`  
    - Sheet Name: `gid=0` (default sheet)  
  - *Input:* Filtered valid contact objects.  
  - *Edge Cases:* API quota limits; auth token expiration; sheet access permissions; data validation errors on Google Sheets side.

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                                  | Input Node(s)                         | Output Node(s)                     | Sticky Note                                                                                                          |
|-----------------------------------|-----------------------------------|-------------------------------------------------|-------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| On form submission                 | Form Trigger                      | Receives uploaded business card images          | —                                   | Neural assistant for contact extraction, Upload file | —                                                                                                                    |
| Upload file                       | Google Drive                      | Archives uploaded images to Google Drive folder | On form submission                  | —                                | ## 1.2. Upload to Google Drive folder for later process                                                               |
| Neural assistant for contact extraction | LangChain Agent                  | Extracts contact info from images using AI       | On form submission                  | Transform Output                  | ## 1.1. Smart agent extract contact from name card picture                                                            |
| GPT4o                             | LangChain OpenAI Chat Model       | Provides GPT-4o model to AI agent                 | — (used inside AI agent)            | Neural assistant for contact extraction (language model) | —                                                                                                                    |
| Structured Output Parser          | LangChain Structured Output Parser | Parses AI raw output into structured JSON         | Neural assistant for contact extraction | Neural assistant for contact extraction (output parser) | —                                                                                                                    |
| Transform Output                  | Code                              | Cleans and formats extracted contact data        | Neural assistant for contact extraction | Filter bad data                  | ## 2. Transform & clean data                                                                                           |
| Filter bad data                  | Filter                            | Filters out invalid contacts (no name)            | Transform Output                   | Add contact to tracking sheet    | —                                                                                                                    |
| Add contact to tracking sheet    | Google Sheets                     | Appends valid contacts to Google Sheet            | Filter bad data                    | —                                | ## 3. Add record to google sheet                                                                                       |
| Sticky Note7                    | Sticky Note                      | Overview and detailed description of workflow    | —                                   | —                                | # 📄 Auto Extract Contacts from Business Cards to Sheet With GPT4o (full description with usage, setup, customization) |
| Sticky Note3                    | Sticky Note                      | Displays an example image of a name card          | —                                   | —                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/namecard.jpg "Optional title text")                    |
| Sticky Note5                    | Sticky Note                      | Explains smart agent extraction block             | —                                   | —                                | ## 1.1. Smart agent extract contact from name card picture                                                             |
| Sticky Note                      | Sticky Note                      | Explains Google Sheets appending step              | —                                   | —                                | ## 3. Add record to google sheet                                                                                       |
| Sticky Note6                    | Sticky Note                      | Screenshot of Google Sheets node configuration    | —                                   | —                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-02+at+1.28.06%E2%80%AFPM.png)       |
| Sticky Note8                    | Sticky Note                      | Screenshot illustrating form submission            | —                                   | —                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-02+at+1.25.32%E2%80%AFPM.png)       |
| Sticky Note9                    | Sticky Note                      | Notes on uploading to Google Drive                 | —                                   | —                                | ## 1.2. Upload to Google Drive folder for later process                                                                |
| Sticky Note10                   | Sticky Note                      | Notes on data transformation and cleaning          | —                                   | —                                | ## 2. Transform & clean data                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add a **Form Trigger** node named “On form submission”.  
   - Configure form title: `Name Card Uploader Form`.  
   - Add one form field: File upload accepting `.jpg, .jpeg, .png` (single file only).  
   - Add form description explaining the purpose (upload business card images to extract contacts).

2. **Add Google Drive Node for Uploading Files**  
   - Add a **Google Drive** node named “Upload file”.  
   - Set operation to upload/save file.  
   - Configure to save the uploaded file in a specific folder by folder ID (`1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI`).  
   - Use OAuth2 credentials for Google Drive.  
   - Connect input from the Form Trigger node (upload file binary).  
   - Set file name as timestamp + original file name for uniqueness.

3. **Add LangChain OpenAI Chat Model Node**  
   - Add a **LangChain LM Chat OpenAI** node named “GPT4o”.  
   - Select model: `gpt-4o`.  
   - Configure with OpenAI API credentials.

4. **Add LangChain Agent Node for Extraction**  
   - Add a **LangChain Agent** node named “Neural assistant for contact extraction”.  
   - Set prompt asking to extract all business card contact details from the image, ignoring decorative elements, and maintaining logical reading order.  
   - Use the GPT4o node as the language model backend.  
   - Enable output parser.  
   - Connect input from “On form submission” node (image binary).  
   - Connect language model input to GPT4o node.

5. **Add LangChain Structured Output Parser Node**  
   - Add a **Structured Output Parser** node named “Structured Output Parser”.  
   - Provide JSON schema example for extracted fields: name, job_title, company, phone, email, website, address, linkedin.  
   - Connect input from the “Neural assistant for contact extraction” node’s output parser.

6. **Add Code Node for Data Transformation**  
   - Add a **Code** node named “Transform Output”.  
   - Use JavaScript to iterate over parsed contacts, clean the phone number by removing “+”, convert emails to lowercase, and map fields to Google Sheet columns.  
   - Connect input from the “Neural assistant for contact extraction” node’s main output.

7. **Add Filter Node to Exclude Invalid Records**  
   - Add a **Filter** node named “Filter bad data”.  
   - Set condition: Field `Name` is not empty.  
   - Connect input from “Transform Output” node.  
   - Only pass contacts with a non-empty Name field.

8. **Add Google Sheets Node to Append Data**  
   - Add a **Google Sheets** node named “Add contact to tracking sheet”.  
   - Set operation to “Append”.  
   - Configure document ID to the Google Sheet where contacts will be stored.  
   - Select sheet by `gid=0`.  
   - Auto map input JSON fields to columns: Name, JobTitle, Company, Phone, Email, Website, Address, LinkedIn.  
   - Connect input from “Filter bad data” node.  
   - Use Google Sheets OAuth2 credentials.

9. **Connect Nodes Sequentially**  
   - On form submission → Neural assistant for contact extraction → Transform Output → Filter bad data → Add contact to tracking sheet.  
   - On form submission → Upload file (parallel path).  
   - Neural assistant for contact extraction → Structured Output Parser (output parser connection).  
   - GPT4o → Neural assistant for contact extraction (language model backend).

10. **Validate Credentials and Permissions**  
    - Ensure Google Drive and Google Sheets nodes have OAuth2 credentials with necessary scopes (file upload, sheet edit).  
    - Ensure OpenAI API key is valid and has access to GPT-4o.  
    - Test the form submission endpoint and verify file upload and AI extraction steps.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow enables seamless and automated digitization of business card contact data, eliminating manual entry errors and saving time for sales and recruitment teams.       | Workflow description sticky note                                                                                 |
| Example business card image used in workflow documentation: ![Example Card](https://wisestackai.s3.ap-southeast-1.amazonaws.com/namecard.jpg)                                    | Sticky Note3 containing image example                                                                             |
| Clear and well-lit images improve OCR and AI extraction accuracy significantly. Avoid blurry or low-contrast photos.                                                             | General best practice                                                                                              |
| Google Drive folder ID and Google Sheets document ID must be set correctly and accessible by the OAuth2 credentials used in the workflow.                                        | Credentials and configuration notes                                                                               |
| To customize, this workflow can be extended to sync data with CRM platforms like HubSpot or Salesforce, or add notification integrations like Slack/email alerts on new entries. | Workflow description sticky note with customization ideas                                                         |
| OpenAI GPT-4o model is used for its image OCR capability combined with large language understanding, ensuring high accuracy in extracting contact info from complex card layouts.| Node GPT4o explanation                                                                                             |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, adhering strictly to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.