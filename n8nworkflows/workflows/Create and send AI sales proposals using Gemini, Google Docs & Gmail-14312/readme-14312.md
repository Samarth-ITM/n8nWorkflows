Create and send AI sales proposals using Gemini, Google Docs & Gmail

https://n8nworkflows.xyz/workflows/create-and-send-ai-sales-proposals-using-gemini--google-docs---gmail-14312


# Create and send AI sales proposals using Gemini, Google Docs & Gmail

# 1. Workflow Overview

This workflow automates the generation and delivery of customized sales proposals when a new lead submission is added to a Google Sheets response sheet. It combines lead intake, AI-generated proposal drafting with Gemini, CRM synchronization in HubSpot, document creation through Google Docs/Drive, PDF export, and email delivery through Gmail.

Typical use cases include:
- Responding automatically to inbound sales inquiry forms
- Creating first-draft proposals for consulting or agency services
- Syncing new leads into a CRM while generating client-facing documents
- Standardizing proposal formatting and delivery

## 1.1 Input Reception

The workflow starts when a new row is added to a Google Sheet. Each row is treated as a proposal request containing client details such as name, email, company, goals, budget, and timestamp.

## 1.2 Iteration Control

The workflow passes incoming rows through a loop node so each submission is processed independently. This makes the design suitable for handling multiple queued items safely.

## 1.3 AI Proposal Generation

Gemini generates a structured proposal draft using the row data. A JavaScript Code node then parses and cleans the AI output into reusable sections for document insertion.

## 1.4 CRM Sync

The workflow creates or updates a contact in HubSpot using the client’s email as the identifier and enriches the record with the client name and company.

## 1.5 Document Generation

A Google Docs proposal template is copied in Google Drive, then placeholders in the copied document are replaced with submission values and AI-generated content. The finished document is then exported as a PDF.

## 1.6 Proposal Delivery

The exported PDF is attached to a Gmail email and sent to the client using a branded HTML message.

## 1.7 Loop Completion

After sending the email, the workflow returns to the loop controller so the next queued row can be processed.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block detects new proposal requests from Google Sheets. It is the workflow entry point and supplies the structured form data used downstream.

### Nodes Involved
- Google Sheets Trigger

### Node Details

#### Google Sheets Trigger
- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`  
  Event-based polling trigger for detecting newly added rows in a Google Sheet.
- **Configuration choices:**
  - Event is set to **rowAdded**
  - Polling frequency is set to **every minute**
  - Targets a specific spreadsheet document and a specific worksheet/tab named **Form Responses 1**
- **Key expressions or variables used:**
  - No complex expressions inside the node itself
  - Produces row fields later referenced as:
    - `Client Name`
    - `Client Email`
    - `Company Name`
    - `Project Goals`
    - `Budget Range`
    - `Timestamp`
- **Input and output connections:**
  - Entry point node, no input
  - Output goes to **Loop Over Items**
- **Version-specific requirements:**
  - Uses **typeVersion 1**
  - Requires a valid Google Sheets credential authorized to access the spreadsheet
- **Edge cases or potential failure types:**
  - OAuth permission issues
  - Spreadsheet or sheet tab not found
  - Polling delay of up to one minute
  - Missing expected columns in the row data
  - Renamed columns causing downstream expression failures
- **Sub-workflow reference:** None

---

## Block 2 — Iteration Control

### Overview
This block ensures each row is processed one at a time. It is important for sequential handling of AI generation, document creation, and email sending.

### Nodes Involved
- Loop Over Items

### Node Details

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Batch/loop controller used here as an iterator over incoming rows.
- **Configuration choices:**
  - No custom batch size or options are explicitly set
  - In this design it acts as the processing loop hub
- **Key expressions or variables used:**
  - Downstream nodes reference the current loop item using:
    - `$('Loop Over Items').item.json[...]`
- **Input and output connections:**
  - Input from **Google Sheets Trigger**
  - Loop processing output goes to **Message a model**
  - Completion/continue path receives input back from **Send a message**
- **Version-specific requirements:**
  - Uses **typeVersion 3**
- **Edge cases or potential failure types:**
  - If downstream nodes fail, the loop stops unless workflow error handling is added
  - Large bursts of rows may queue and process slowly depending on external service latency
  - Expressions that explicitly reference `Loop Over Items` depend on loop context existing
- **Sub-workflow reference:** None

---

## Block 3 — AI Proposal Generation

### Overview
This block generates a persuasive sales proposal with Gemini and converts the response into two clean text sections: executive summary and proposed solution. These outputs are later inserted into the proposal document.

### Nodes Involved
- Message a model
- Code in JavaScript2

### Node Details

#### Message a model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Sends a prompt to the Google Gemini model and returns generated text.
- **Configuration choices:**
  - Model: **models/gemini-2.5-flash**
  - Prompt dynamically includes row values from the current item:
    - Client name
    - Company name
    - Project goals
  - The model is instructed to return two sections:
    - `EXECUTIVE SUMMARY`
    - `PROPOSED SOLUTION`
  - Tone requested: confident, professional, helpful
- **Key expressions or variables used:**
  - `{{ $json['Client Name'] }}`
  - `{{ $json['Company Name'] }}`
  - `{{ $json['Project Goals'] }}`
- **Input and output connections:**
  - Input from **Loop Over Items**
  - Output to **Code in JavaScript2**
- **Version-specific requirements:**
  - Uses **typeVersion 1**
  - Requires Google Gemini / Google AI credentials configured in n8n
  - The exact response structure can vary by node version and provider output format
- **Edge cases or potential failure types:**
  - Credential or quota issues
  - Model output may not follow the requested headings exactly
  - Safety filters or content moderation may alter output
  - Network or API timeout errors
- **Sub-workflow reference:** None

#### Code in JavaScript2
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the Gemini response and extracts normalized text for document placeholders.
- **Configuration choices:**
  - Reads generated text from:
    - `$input.first().json.content.parts[0].text`
  - Splits the text using the marker:
    - `## PROPOSED SOLUTION`
  - Removes `## EXECUTIVE SUMMARY` if present
  - Removes a possible `Dear ...` salutation
  - Cleans formatting characters:
    - `#`
    - `*`
    - `---`
  - Returns:
    - `AI_Summary`
    - `AI_Solution`
- **Key expressions or variables used:**
  - `const fullText = $input.first().json.content.parts[0].text;`
  - `fullText.split("## PROPOSED SOLUTION")`
- **Input and output connections:**
  - Input from **Message a model**
  - Output to **Create or update a contact**
- **Version-specific requirements:**
  - Uses **typeVersion 2**
  - Assumes the Gemini node returns text at `content.parts[0].text`
- **Edge cases or potential failure types:**
  - If Gemini response structure changes, `content.parts[0].text` may be undefined
  - If the model does not return `## PROPOSED SOLUTION`, the solution falls back to `"No solution generated."`
  - If the model returns headings without `##`, parsing will degrade
  - If output contains rich formatting beyond simple markdown markers, cleanup may be incomplete
- **Sub-workflow reference:** None

---

## Block 4 — CRM Sync

### Overview
This block synchronizes lead details into HubSpot before proposal document creation continues. It uses the client email as the main identifier and stores name and company details.

### Nodes Involved
- Create or update a contact

### Node Details

#### Create or update a contact
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Creates or updates a HubSpot contact record.
- **Configuration choices:**
  - Authentication method: **appToken**
  - Email field:
    - `$('Loop Over Items').item.json['Client Email']`
  - Additional fields:
    - First name from `Client Name`
    - Company name from `Company Name`
- **Key expressions or variables used:**
  - `={{ $('Loop Over Items').item.json['Client Email'] }}`
  - `={{ $('Loop Over Items').item.json['Client Name'] }}`
  - `={{ $('Loop Over Items').item.json['Company Name'] }}`
- **Input and output connections:**
  - Input from **Code in JavaScript2**
  - Output to **Copy file**
- **Version-specific requirements:**
  - Uses **typeVersion 2.2**
  - Requires HubSpot Private App Token credentials in n8n
- **Edge cases or potential failure types:**
  - Invalid or missing email address
  - HubSpot auth/token scope problems
  - Field mapping issues if account properties differ
  - API rate limits
- **Sub-workflow reference:** None

---

## Block 5 — Document Generation

### Overview
This block creates a customer-specific proposal document from a template, fills placeholders with form and AI data, and exports the result as a PDF for delivery.

### Nodes Involved
- Copy file
- Update a document
- Download file

### Node Details

#### Copy file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Creates a copy of a master Google Docs template in Google Drive.
- **Configuration choices:**
  - Operation: **copy**
  - Source file is a Google Docs template identified by a fixed file ID placeholder:
    - `YOUR_GOOGLE_DRIVE_FILE_ID`
  - New file name:
    - `Proposal - {{ Company Name }}`
- **Key expressions or variables used:**
  - `=Proposal - {{ $('Loop Over Items').item.json['Company Name'] }}`
- **Input and output connections:**
  - Input from **Create or update a contact**
  - Output to **Update a document**
- **Version-specific requirements:**
  - Uses **typeVersion 3**
  - Requires Google Drive credentials
- **Edge cases or potential failure types:**
  - Template file ID not replaced from placeholder
  - Missing Drive access permission
  - Duplicate file naming may produce many similarly named docs
  - Copying a non-Google-Docs file could affect downstream Google Docs operations
- **Sub-workflow reference:** None

#### Update a document
- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Updates the copied Google Docs file by replacing placeholder tokens.
- **Configuration choices:**
  - Operation: **update**
  - Uses **documentURL** based on prior node output:
    - `{{ $json.id }}`
  - `simple` mode is disabled
  - Performs multiple `replaceAll` actions with exact-case matching
  - Replaces:
    - `{{CompanyName}}`
    - `{{ClientName}}`
    - `{{AI_Summary}}`
    - `{{Budget}}`
    - `{{Date}}`
    - `{{AI_Solution}}`
- **Key expressions or variables used:**
  - `={{ $('Loop Over Items').item.json['Company Name'] }}`
  - `={{ $('Loop Over Items').item.json['Client Name'] }}`
  - `={{ $('Code in JavaScript2').item.json.AI_Summary }}`
  - `={{ $('Loop Over Items').item.json['Budget Range'] }}`
  - `={{ $('Loop Over Items').item.json.Timestamp }}`
  - `={{ $('Code in JavaScript2').item.json.AI_Solution }}`
- **Input and output connections:**
  - Input from **Copy file**
  - Output to **Download file**
- **Version-specific requirements:**
  - Uses **typeVersion 2**
  - Requires Google Docs credentials
  - Assumes the copied file is a valid Google Docs document
- **Edge cases or potential failure types:**
  - Placeholder tokens missing from template
  - Exact-case matching means token case must match perfectly
  - Long AI text could affect document formatting
  - If prior node returns an unexpected `id` shape, document targeting can fail
- **Sub-workflow reference:** None

#### Download file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads the updated Google Docs document as a PDF binary.
- **Configuration choices:**
  - Operation: **download**
  - File ID is taken from:
    - `{{ $json.documentId }}`
  - Google file conversion enabled:
    - Docs to PDF (`application/pdf`)
- **Key expressions or variables used:**
  - `={{ $json.documentId }}`
- **Input and output connections:**
  - Input from **Update a document**
  - Output to **Send a message**
- **Version-specific requirements:**
  - Uses **typeVersion 3**
  - Requires Google Drive credentials with file download/export access
- **Edge cases or potential failure types:**
  - `documentId` missing from previous node output
  - Export failure if file is not a Google Docs document
  - Binary data size issues for large documents
- **Sub-workflow reference:** None

---

## Block 6 — Proposal Delivery

### Overview
This block emails the generated PDF proposal to the client using Gmail. The email body is HTML-formatted and personalized with client and company details.

### Nodes Involved
- Send a message

### Node Details

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email with a binary attachment through Gmail.
- **Configuration choices:**
  - Recipient:
    - `Client Email`
  - Subject:
    - `Proposal for {{ Company Name }}`
  - HTML message body references:
    - Client name
    - Company name
    - Project goals
  - Attachments enabled using binary output from previous node
- **Key expressions or variables used:**
  - `={{ $('Loop Over Items').item.json['Client Email'] }}`
  - `=Proposal for {{ $('Loop Over Items').item.json['Company Name'] }}`
  - HTML body expressions for:
    - `Client Name`
    - `Company Name`
    - `Project Goals`
- **Input and output connections:**
  - Input from **Download file**
  - Output returns to **Loop Over Items**
- **Version-specific requirements:**
  - Uses **typeVersion 2.2**
  - Requires Gmail OAuth2 credentials
- **Edge cases or potential failure types:**
  - Gmail auth expired
  - Attachment may not send if binary property is missing
  - Invalid recipient email
  - HTML contains a malformed tag in the greeting:
    - `<{{ $('Loop Over Items').item.json['Client Name'] }}/span>`
    - This should likely be `<span class="highlight">{{ ... }}</span>`
  - Gmail sending limits or anti-spam restrictions
- **Sub-workflow reference:** None

---

## Block 7 — Documentation / In-Canvas Notes

### Overview
These sticky notes document the workflow purpose, setup requirements, and logical regions. They are not executable but are important for maintainability and reproduction.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note covering the broader workflow.
- **Configuration choices:**
  - Contains overview, flow summary, setup steps, and customization tips
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - Uses **typeVersion 1**
- **Edge cases or potential failure types:**
  - None operational
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the AI/CRM section
- **Input and output connections:** None
- **Version-specific requirements:** typeVersion 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the document generation section
- **Input and output connections:** None
- **Version-specific requirements:** typeVersion 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the delivery/loop section
- **Input and output connections:** None
- **Version-specific requirements:** typeVersion 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | n8n-nodes-base.googleSheetsTrigger | Detects new rows added to the intake sheet |  | Loop Over Items | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterates through incoming rows one by one | Google Sheets Trigger, Send a message | Message a model | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## AI Processing & CRM<br>Generates proposal content and syncs lead data to HubSpot. |
| Message a model | @n8n/n8n-nodes-langchain.googleGemini | Generates proposal text from lead details | Loop Over Items | Code in JavaScript2 | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## AI Processing & CRM<br>Generates proposal content and syncs lead data to HubSpot. |
| Code in JavaScript2 | n8n-nodes-base.code | Parses and cleans AI output into document-ready sections | Message a model | Create or update a contact | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## AI Processing & CRM<br>Generates proposal content and syncs lead data to HubSpot. |
| Create or update a contact | n8n-nodes-base.hubspot | Creates or updates the lead in HubSpot | Code in JavaScript2 | Copy file | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## AI Processing & CRM<br>Generates proposal content and syncs lead data to HubSpot.<br>## Document Generation<br>Creates proposal, replaces placeholders, and exports PDF. |
| Copy file | n8n-nodes-base.googleDrive | Duplicates the master proposal template | Create or update a contact | Update a document | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## Document Generation<br>Creates proposal, replaces placeholders, and exports PDF. |
| Update a document | n8n-nodes-base.googleDocs | Replaces placeholders in the copied proposal document | Copy file | Download file | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## Document Generation<br>Creates proposal, replaces placeholders, and exports PDF. |
| Download file | n8n-nodes-base.googleDrive | Exports the Google Doc as PDF binary | Update a document | Send a message | # AI Proposal Automation Overview<br>This workflow automates the full sales proposal process from lead capture to delivery. It uses AI to generate tailored proposals, updates CRM records, creates documents, and sends them to clients automatically.<br>---<br>### How it works<br>1. A Google Sheets trigger detects new form submissions.<br>2. Gemini AI generates a personalized proposal.<br>3. A Code node cleans and structures the content.<br>4. HubSpot creates or updates the contact.<br>5. Google Docs creates a proposal from a template and converts it to PDF.<br>6. Gmail sends the final proposal to the client.<br>---<br>### Setup steps<br>1. Connect Google Sheets, Drive, Docs, and Gmail accounts.<br>2. Add your proposal template with placeholders like {{AI_Summary}} and {{AI_Solution}}.<br>3. Configure HubSpot with a Private App Token.<br>4. Ensure your sheet includes required fields (Client Name, Email, Goals, etc.).<br>---<br>### Customization tips<br>- Modify the AI prompt to change tone or industry focus.<br>- Adjust email content and branding.<br>- Expand placeholders for pricing, timelines, or services.<br>## Document Generation<br>Creates proposal, replaces placeholders, and exports PDF.<br>## Delivery & Loop<br>Sends proposal via email and handles multiple submissions. |
| Send a message | n8n-nodes-base.gmail | Emails the proposal PDF to the client | Download file | Loop Over Items | ## Delivery & Loop<br>Sends proposal via email and handles multiple submissions. |
| Sticky Note | n8n-nodes-base.stickyNote | In-canvas documentation for overall workflow |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | In-canvas section label for AI and CRM |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | In-canvas section label for document generation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | In-canvas section label for delivery and loop |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it: **Create and send AI sales proposals using Gemini, Google Docs & Gmail**

2. **Add a Google Sheets Trigger node**
   - Node type: **Google Sheets Trigger**
   - Set **Event** to: `rowAdded`
   - Set poll interval to: `every minute`
   - Choose your spreadsheet document
   - Choose the target sheet/tab, for example: `Form Responses 1`
   - Configure Google Sheets credentials with access to the spreadsheet

3. **Prepare the Google Sheet columns**
   - Ensure the sheet includes at least these columns:
     - `Client Name`
     - `Client Email`
     - `Company Name`
     - `Project Goals`
     - `Budget Range`
     - `Timestamp`
   - Keep names exact if you want to reuse the expressions as-is

4. **Add a Loop Over Items node**
   - Node type: **Loop Over Items** / **Split In Batches**
   - Keep default options unless you want a custom batch size
   - Connect:
     - `Google Sheets Trigger -> Loop Over Items`

5. **Add a Google Gemini node**
   - Node type: **Google Gemini Chat Model** / the n8n Gemini node matching `@n8n/n8n-nodes-langchain.googleGemini`
   - Select model: `models/gemini-2.5-flash`
   - Configure Google AI credentials
   - In the message content, use a prompt equivalent to:

     ```text
     You are a world-class Sales Consultant. Write a professional, persuasive sales proposal based on these details:

     Client: {{ $json['Client Name'] }}

     Company: {{ $json['Company Name'] }}

     Goals: {{ $json['Project Goals'] }}

     Structure your response with two headers:

     EXECUTIVE SUMMARY
     (Write a 2-paragraph summary here)

     PROPOSED SOLUTION
     (Write 3 bullet points of how we solve their specific goals)

     Tone: Confident, professional, and helpful.
     ```

   - Connect:
     - `Loop Over Items -> Message a model`

6. **Add a Code node**
   - Node type: **Code**
   - Language: JavaScript
   - Paste the following logic:

     ```javascript
     const fullText = $input.first().json.content.parts[0].text;
     const parts = fullText.split("## PROPOSED SOLUTION");

     let summary = parts[0];

     if (summary.includes("## EXECUTIVE SUMMARY")) {
       summary = summary.split("## EXECUTIVE SUMMARY")[1];
     }

     summary = summary.replace(/Dear .*?,/, "").trim();

     const cleanText = (text) => {
       return text
         .replace(/[#*]/g, '')
         .replace(/---/g, '')
         .replace(/\n\s*\n/g, '\n\n')
         .trim();
     };

     return {
       AI_Summary: cleanText(summary),
       AI_Solution: parts[1] ? cleanText(parts[1]) : "No solution generated."
     };
     ```

   - Connect:
     - `Message a model -> Code in JavaScript2`

7. **Add a HubSpot node**
   - Node type: **HubSpot**
   - Operation: **Create or update contact**
   - Authentication: **Private App Token**
   - Configure HubSpot credentials in n8n
   - Set fields:
     - Email: `{{ $('Loop Over Items').item.json['Client Email'] }}`
     - First Name: `{{ $('Loop Over Items').item.json['Client Name'] }}`
     - Company Name: `{{ $('Loop Over Items').item.json['Company Name'] }}`
   - Connect:
     - `Code in JavaScript2 -> Create or update a contact`

8. **Create a master Google Docs proposal template**
   - In Google Docs, create a base proposal file
   - Include placeholders exactly matching:
     - `{{CompanyName}}`
     - `{{ClientName}}`
     - `{{AI_Summary}}`
     - `{{Budget}}`
     - `{{Date}}`
     - `{{AI_Solution}}`
   - Save the file in Google Drive
   - Copy the document ID from its URL

9. **Add a Google Drive node to copy the template**
   - Node type: **Google Drive**
   - Operation: **Copy**
   - File ID: your master proposal template ID
   - Name:
     - `Proposal - {{ $('Loop Over Items').item.json['Company Name'] }}`
   - Configure Google Drive credentials
   - Connect:
     - `Create or update a contact -> Copy file`

10. **Add a Google Docs node to replace placeholders**
    - Node type: **Google Docs**
    - Operation: **Update**
    - Disable simple mode if required by your node version
    - Set the target document using the copied file output
    - Use the copied file identifier from the previous node as the document reference
    - Add replace actions:
      1. Replace `{{CompanyName}}` with `{{ $('Loop Over Items').item.json['Company Name'] }}`
      2. Replace `{{ClientName}}` with `{{ $('Loop Over Items').item.json['Client Name'] }}`
      3. Replace `{{AI_Summary}}` with `{{ $('Code in JavaScript2').item.json.AI_Summary }}`
      4. Replace `{{Budget}}` with `{{ $('Loop Over Items').item.json['Budget Range'] }}`
      5. Replace `{{Date}}` with `{{ $('Loop Over Items').item.json.Timestamp }}`
      6. Replace `{{AI_Solution}}` with `{{ $('Code in JavaScript2').item.json.AI_Solution }}`
    - Enable exact matching if you want behavior identical to this workflow
    - Connect:
      - `Copy file -> Update a document`

11. **Add a Google Drive node to export the file as PDF**
    - Node type: **Google Drive**
    - Operation: **Download**
    - File ID:
      - Use the document ID from the Google Docs update output
    - Enable Google file conversion
    - Choose conversion:
      - Google Docs to `application/pdf`
    - Connect:
      - `Update a document -> Download file`

12. **Add a Gmail node**
    - Node type: **Gmail**
    - Operation: **Send message**
    - Configure Gmail OAuth2 credentials
    - Set recipient:
      - `{{ $('Loop Over Items').item.json['Client Email'] }}`
    - Set subject:
      - `Proposal for {{ $('Loop Over Items').item.json['Company Name'] }}`
    - Set email body as HTML
    - Use the workflow’s HTML body or a corrected version
    - Attach the binary file coming from the previous node
    - Connect:
      - `Download file -> Send a message`

13. **Fix the HTML greeting before production use**
    - The current workflow contains malformed HTML:
      - `<{{ $('Loop Over Items').item.json['Client Name'] }}/span>`
    - Replace it with:
      ```html
      <span class="highlight">{{ $('Loop Over Items').item.json['Client Name'] }}</span>
      ```

14. **Close the processing loop**
    - Connect:
      - `Send a message -> Loop Over Items`
    - This allows the next incoming item to be processed

15. **Add optional sticky notes for maintainability**
    - Create one overview note describing the full process
    - Create section notes for:
      - AI Processing & CRM
      - Document Generation
      - Delivery & Loop

16. **Test each integration independently**
    - Test Google Sheets trigger with a sample row
    - Test Gemini output formatting
    - Test HubSpot contact creation/update
    - Test document copy and replacement against your template
    - Test PDF export
    - Test Gmail delivery with attachment

17. **Run an end-to-end validation**
    - Add a new row to the sheet
    - Confirm:
      - Gemini generates content
      - HubSpot contact appears or updates
      - Proposal doc is copied and filled
      - PDF is generated
      - Email is delivered to the client

18. **Recommended hardening improvements**
    - Add an IF node before Gemini to validate required fields
    - Add fallback parsing if Gemini headings vary
    - Add error handling or separate failure notifications
    - Log created document URLs for traceability
    - Avoid exact-case placeholder matching unless your template is strictly controlled

### Credential configuration summary
- **Google Sheets Trigger:** Google account with spreadsheet read access
- **Google Drive:** Google account with template copy/export permissions
- **Google Docs:** Google account with document edit permissions
- **Gmail:** Gmail OAuth2 account allowed to send emails
- **HubSpot:** Private App Token with contact write permissions
- **Gemini:** Google AI / Gemini API credential with model access

### Sub-workflow setup
- This workflow does **not** use any Execute Workflow or other sub-workflow nodes.
- There are **no sub-workflow dependencies** to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow assumes a Google Form or similar intake process is writing new lead rows into Google Sheets. | Operational assumption |
| The document template must contain exact placeholder strings for successful replacement. | Google Docs template design |
| The AI parsing logic currently expects markdown-style headings like `## EXECUTIVE SUMMARY` and `## PROPOSED SOLUTION`, even though the prompt text does not explicitly force the `##` prefix. This mismatch can cause parsing issues. | Parsing/design caution |
| The Gmail HTML body contains a malformed client name span tag and should be corrected before production use. | Email formatting caution |
| The source JSON includes placeholder values such as `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`, `YOUR_GOOGLE_DRIVE_FILE_ID`, and `YOUR_GMAIL_WEBHOOK_ID`; these must be replaced with real values in a live environment. | Deployment requirement |
| Branding in the email body references `iTechNotion` and sender name `Krish`. Adjust these to your own organization if needed. | Branding customization |