Extract PDF Itineraries to Google Sheets with GPT-4.1-Mini & Gmail Notifications

https://n8nworkflows.xyz/workflows/extract-pdf-itineraries-to-google-sheets-with-gpt-4-1-mini---gmail-notifications-9281


# Extract PDF Itineraries to Google Sheets with GPT-4.1-Mini & Gmail Notifications

### 1. Workflow Overview

This workflow automates the extraction of structured itinerary data from multiple PDF files uploaded via a form. It leverages OpenAI’s GPT-4.1-Mini to analyze and extract specific client and tour details from each PDF, appends this data to a Google Sheets document, and sends confirmation emails notifying the team about the successful processing of the itineraries. The workflow is designed to drastically reduce manual data entry effort (estimated 90% reduction) and provides real-time notifications to ensure transparency and tracking.

The workflow’s logical blocks are:

- **1.1 Input Reception:** Receives multiple PDF file uploads from users through a web form and prepares them for processing.
- **1.2 File Splitting and Looping:** Splits the batch of uploaded PDFs into individual files and loops through each for sequential processing.
- **1.3 AI-Based PDF Data Extraction:** Uses OpenAI GPT-4.1-Mini to extract structured itinerary data (agency, contact, dates, tour details) from each PDF.
- **1.4 Data Storage:** Appends or updates the extracted data into a specified Google Sheet with proper matching on agency name.
- **1.5 Email Generation and Notification:** Creates a confirmation email using GPT-4O-MINI based on extracted data and sends the email notification via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving multiple PDF files submitted through a web form interface.

**Nodes Involved:**  
- Form receives multiple PDF files

**Node Details:**  
- **Name:** Form receives multiple PDF files  
- **Type:** `formTrigger`  
- **Role:** Triggers the workflow upon user form submission containing multiple PDF files.  
- **Configuration:**  
  - Form titled "LOAD MULTIPLE FILES"  
  - Single form field accepting multiple files, restricted to PDF (`*.pdf`) formats only  
  - Mandatory file upload field  
- **Input:** External user form submission  
- **Output:** JSON payload including uploaded PDF files in binary format under the field `files`  
- **Edge Cases:**  
  - User submits no files or unsupported file formats — form validation prevents this  
  - Large file uploads exceeding server limits — may cause trigger failure or timeout

---

#### 2.2 File Splitting and Looping

**Overview:**  
Splits the batch of uploaded files into individual PDF documents and processes each document in sequence.

**Nodes Involved:**  
- Split Files processes each PDF individually  
- Loop Over Items ensures each document

**Node Details:**  

- **Split Files processes each PDF individually**  
  - **Type:** `splitOut`  
  - **Role:** Separates the array of uploaded files into individual items for processing  
  - **Configuration:**  
    - Splits the input array from `files` field into individual outputs  
    - Retains binary data for each file  
  - **Input:** Batch of files from form trigger  
  - **Output:** One JSON item per PDF file  
  - **Edge Cases:** Empty file arrays, corrupted files  

- **Loop Over Items ensures each document**  
  - **Type:** `splitInBatches`  
  - **Role:** Controls processing flow by looping over each split PDF file one at a time  
  - **Configuration:**  
    - Batch size defaults to 1, ensuring sequential processing  
    - No reset of batch counter between executions  
  - **Input:** Individual files from split node  
  - **Output:** Passes each file sequentially downstream for extraction  
  - **Edge Cases:** Interruptions in loop, node timeout on large PDFs

---

#### 2.3 AI-Based PDF Data Extraction

**Overview:**  
Analyzes each PDF file using a specialized OpenAI Information Extractor node that uses GPT-4.1-Mini to extract structured itinerary data fields.

**Nodes Involved:**  
- Analyzes & extract PDF

**Node Details:**  
- **Name:** Analyzes & extract PDF  
- **Type:** `@n8n/n8n-nodes-langchain.informationExtractor`  
- **Role:** Automatically extracts key client and tour itinerary fields from the PDF filename (assumed representation of text) using AI  
- **Configuration:**  
  - Text input is the filename of the PDF (`{{$json.files.filename}}`) — note: usually PDF binary content would be processed, here filename is used, possibly simplified for example  
  - System prompt instructs the AI to extract only relevant attributes, omitting unknowns  
  - Attributes extracted (all required):  
    - Agency Name (client name)  
    - Email  
    - Address  
    - Phone  
    - Date (as date type)  
    - Tour (holiday package)  
    - Departure date  
- **Input:** Single PDF file object from loop  
- **Output:** JSON object with extracted attribute values under `output` key  
- **Edge Cases:**  
  - AI fails to extract due to unclear/incomplete data  
  - Incorrect attribute types or missing required fields  
  - Filename may not contain useful data if actual PDF content is expected  
- **Version Requirements:** Compatible with n8n v1.0.0+ and OpenAI GPT-4.1-Mini model

---

#### 2.4 Data Storage

**Overview:**  
Appends or updates client itinerary data into a Google Sheet, matching on ‘Agency Name’ to avoid duplicates.

**Nodes Involved:**  
- Extracted information to Google Sheets

**Node Details:**  
- **Name:** Extracted information to Google Sheets  
- **Type:** `googleSheets`  
- **Role:** Adds or updates rows in a specific Google Sheet for each extracted itinerary record  
- **Configuration:**  
  - Operation: Append or update, matching on ‘Agency Name’ column  
  - Columns mapped from AI output attributes: Date, tour, Email, Phone, Address, Agency Name, departure date  
  - Sheet ID: `1lg-GRBTQCvM9WC_Mbhe4YXwgyrRcHm8K2JYDHg4XOww`  
  - Sheet tab: `gid=0` (first sheet)  
  - Attempts no type conversion (`attemptToConvertTypes` = false)  
- **Input:** Extracted JSON data from AI extraction node  
- **Output:** Confirmation of sheet update  
- **Edge Cases:**  
  - Credential expiration or invalid OAuth token for Google Sheets  
  - API quota limits  
  - Sheet structure changes causing mapping failures

---

#### 2.5 Email Generation and Notification

**Overview:**  
Creates a customized confirmation email using GPT-4O-MINI and sends it via Gmail to notify relevant parties that the itinerary has been processed and stored.

**Nodes Involved:**  
- Create Email  
- email confirmation sent with results

**Node Details:**  

- **Create Email**  
  - **Type:** `@n8n/n8n-nodes-langchain.openAi`  
  - **Role:** Generates an email subject and message body dynamically from extracted itinerary data  
  - **Configuration:**  
    - Model: GPT-4O-MINI  
    - Messages:  
      - User message template embeds extracted JSON fields (Agency Name, Email, Address, Phone, Date, tour, departure date)  
      - System prompt instructs to create an email confirming receipt, update notification with Google Sheets link  
    - Output JSON parsed for fields `Subject` and `Email`  
  - **Input:** Extracted itinerary JSON from Google Sheets node  
  - **Output:** Email content JSON with separate fields for subject and body  
  - **Edge Cases:**  
    - API call failures, token limits, prompt errors  

- **email confirmation sent with results**  
  - **Type:** `gmail`  
  - **Role:** Sends the generated email to a fixed recipient email address  
  - **Configuration:**  
    - Recipient: fixed Gmail address (redacted here)  
    - Subject and message body dynamically bound from Create Email node output  
    - Email type: text/plain  
    - OAuth2 credentials for Gmail configured  
  - **Input:** Email content JSON from Create Email node  
  - **Output:** Gmail API response confirming send status  
  - **Edge Cases:**  
    - Gmail OAuth token expiration  
    - Sending quota exceeded  
    - Invalid recipient address

---

### 3. Summary Table

| Node Name                          | Node Type                                  | Functional Role                        | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                                  |
|-----------------------------------|--------------------------------------------|-------------------------------------|-----------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Form receives multiple PDF files  | formTrigger                                | Receives uploaded PDF files          | (Trigger)                        | Split Files processes each PDF individually | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| Split Files processes each PDF individually | splitOut                                   | Splits batch into individual PDFs    | Form receives multiple PDF files | Loop Over Items ensures each document | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| Loop Over Items ensures each document | splitInBatches                             | Loops over each PDF for processing   | Split Files processes each PDF individually | Analyzes & extract PDF             | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| Analyzes & extract PDF            | @n8n/n8n-nodes-langchain.informationExtractor | Extracts structured data with GPT   | Loop Over Items ensures each document | Extracted information to Google Sheets | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| Extracted information to Google Sheets | googleSheets                               | Stores extracted data in Sheets     | Analyzes & extract PDF           | Create Email                      | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| Create Email                      | @n8n/n8n-nodes-langchain.openAi           | Generates confirmation email content | Extracted information to Google Sheets | email confirmation sent with results | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| email confirmation sent with results | gmail                                      | Sends confirmation email            | Create Email                    | Loop Over Items ensures each document (start next batch) | See sticky note block 6dd88c5f: Workflow overview, prerequisites, setup, customization, use cases            |
| Sticky Note1                     | stickyNote                                 | Documentation and overview          | -                               | -                                  | # Extract PDF Itineraries to Google Sheets with GPT-4.1-Mini & Gmail Notifications; setup and use notes      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: `formTrigger`  
   - Name: "Form receives multiple PDF files"  
   - Configure form: Title "LOAD MULTIPLE FILES"  
   - Add a required file upload field accepting multiple files with `*.pdf` filter  
   - Save and deploy webhook  

2. **Add Split Out Node**  
   - Type: `splitOut`  
   - Name: "Split Files processes each PDF individually"  
   - Set field to split: `files` (array of uploaded PDFs)  
   - Enable “Include Binary” to pass PDF content  
   - Connect output of form trigger node to this node  

3. **Add Split In Batches Node**  
   - Type: `splitInBatches`  
   - Name: "Loop Over Items ensures each document"  
   - Batch size: 1 (default)  
   - Connect output of splitOut node to this node  

4. **Add Information Extractor Node (Langchain)**  
   - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
   - Name: "Analyzes & extract PDF"  
   - Configure text input: Use expression `{{$json.files.filename}}` (or better, pass actual PDF text if available)  
   - Set system prompt to instruct AI to extract only relevant attributes  
   - Define attributes to extract with descriptions and required flags:  
     - Agency Name (string, required)  
     - Email (string, required)  
     - Address (string, required)  
     - Phone (string, required)  
     - Date (date, required)  
     - tour (string, required)  
     - departure date (string, required)  
   - Connect output of splitInBatches node (main output 2) to this node  

5. **Add Google Sheets Node**  
   - Type: `googleSheets`  
   - Name: "Extracted information to Google Sheets"  
   - Operation: Append or Update  
   - Document ID: Use your Google Sheet ID  
   - Sheet Name: Use the specific sheet tab (e.g., gid=0)  
   - Define columns mapping from AI output:  
     - Date ← `{{$json.output[' Date']}}`  
     - tour ← `{{$json.output.tour}}`  
     - Email ← `{{$json.output.Email}}`  
     - Phone ← `{{$json.output.Phone}}`  
     - Address ← `{{$json.output.Address}}`  
     - Agency Name ← `{{$json.output['Agency Name']}}`  
     - departure date ← `{{$json.output['departure date']}}`  
   - Matching columns: `Agency Name`  
   - Connect output of Information Extractor node to this node  

6. **Add OpenAI Node for Email Creation**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Name: "Create Email"  
   - Model: GPT-4O-MINI  
   - Configure messages:  
     - User message embedding the extracted JSON fields for personalization  
     - System prompt: instruct to create an email confirming receipt and updated database link  
   - Enable JSON output parsing  
   - Connect output of Google Sheets node to this node  

7. **Add Gmail Node to Send Email**  
   - Type: `gmail`  
   - Name: "email confirmation sent with results"  
   - Configure recipient email address  
   - Subject: bind from Create Email node’s JSON output `Subject` field  
   - Message body: bind from Create Email node’s JSON output `Email` field  
   - Set email type to text/plain  
   - Connect output of Create Email node to this node  

8. **Connect Email Node Back to Loop Node**  
   - Connect the output of Gmail node back to the "Loop Over Items ensures each document" node (main output 1) to continue processing next batch  

9. **Credentials Setup**  
   - Add and configure OpenAI API credentials for GPT-4.1-Mini and GPT-4O-MINI models  
   - Add Google Sheets OAuth2 credentials with write access to the target spreadsheet  
   - Add Gmail OAuth2 credentials with send mail permission  

10. **Testing and Activation**  
    - Import or recreate the workflow in n8n  
    - Test form submission with multiple PDF files  
    - Monitor logs for processing, extraction accuracy, sheet updates, and email sends  
    - Adjust prompts or mappings as necessary  
    - Activate workflow for production use  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                  | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Reduces manual data entry by ~90% for PDF itinerary processing using AI and automation.                                                                                                        | Workflow overview                                                                                   |
| Requires OpenAI API key from https://platform.openai.com for GPT-4.1-Mini and GPT-4O-MINI access.                                                                                             | Prerequisite                                                                                       |
| Google Workspace setup needed for Sheets and Gmail with OAuth2 authorization.                                                                                                                  | Prerequisite                                                                                       |
| Workflow tested on n8n version 1.0.0 and above.                                                                                                                                                 | Version compatibility                                                                              |
| Editable AI prompts allow customization for different document types like invoices, resumes, legal documents.                                                                                  | Customization                                                                                      |
| Google Sheets link used in email notification: https://docs.google.com/spreadsheets/d/1lg-GRBTQCvM9WC_Mbhe4YXwgyrRcHm8K2JYDHg4XOww/edit?gid=0                                                    | Included in email system prompt                                                                    |
| For more complex documents, consider extending extraction to binary PDF text parsing rather than filename.                                                                                      | Improvement recommendation                                                                         |
| Sticky Note in workflow provides detailed overview and setup instructions visible inside n8n editor for maintainers.                                                                            | Node: Sticky Note1                                                                                  |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, respecting all current content policies. No illegal or protected content is included. All data processed is legal and public.