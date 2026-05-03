Transform Support Tickets into AI Documentation with GPT-4, PDFs and Google Drive

https://n8nworkflows.xyz/workflows/transform-support-tickets-into-ai-documentation-with-gpt-4--pdfs-and-google-drive-9260


# Transform Support Tickets into AI Documentation with GPT-4, PDFs and Google Drive

### 1. Workflow Overview

This workflow automates the transformation of resolved customer support tickets into professionally formatted AI-generated documentation. It captures ticket data via a webhook, extracts and normalizes the ticket details, uses GPT-4 to summarize the case into a knowledge base style document, formats this summary into a branded HTML report, converts it to PDF, stores the document in Google Drive, logs metadata in Google Sheets, and notifies the support team in Slack. Comprehensive error handling ensures failures in PDF generation or cloud uploads are caught and alerted.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception:** Receives resolved ticket data via webhook.
- **1.2 Data Extraction:** Normalizes ticket details from various support tools.
- **1.3 AI Summarization:** Uses GPT-4 to generate structured case study summaries.
- **1.4 HTML Formatting:** Combines ticket data and AI summary into branded HTML.
- **1.5 PDF Conversion:** Converts HTML to PDF using an external API.
- **1.6 PDF Retrieval & Validation:** Downloads PDF binary and validates success.
- **1.7 Google Drive Upload:** Uploads PDF file to a specified Drive folder.
- **1.8 Upload Validation:** Checks Google Drive upload success.
- **1.9 Database Update:** Logs documentation metadata into a Google Sheet.
- **1.10 Slack Notification:** Notifies support team of successful documentation.
- **1.11 Error Handling:** Sends Slack alerts on failures in PDF generation or Drive upload.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Captures incoming resolved support tickets through an HTTP POST webhook for flexible integration with any support tool.

- **Nodes Involved:**  
  - Webhook - Receive Ticket  
  - 📝 Sticky Note - Trigger Setup  

- **Node Details:**

  - **Webhook - Receive Ticket**  
    - Type: Webhook  
    - Role: Entry point, listens for POST requests on path `support-ticket-resolved`  
    - Config: HTTP POST method, responds immediately after processing  
    - Input: Incoming JSON payload with ticket details  
    - Output: Passes raw payload downstream  
    - Edge Cases: Missing or malformed payload, HTTP errors, webhook misconfiguration  
    - Sticky Note: Explains expected JSON payload structure and setup instructions with sample data  

  - **📝 Sticky Note - Trigger Setup**  
    - Type: Sticky Note (documentation only)  
    - Role: Explains trigger options, payload expectations, and testing tips  

---

#### 1.2 Data Extraction

- **Overview:**  
Normalizes and extracts key ticket details from various possible payload formats, providing fallback defaults for missing fields.

- **Nodes Involved:**  
  - Extract Ticket Details (Code)  
  - 📝 Sticky Note - Data Extraction  

- **Node Details:**

  - **Extract Ticket Details**  
    - Type: Code (JavaScript)  
    - Role: Parses raw webhook payload, extracts normalized fields such as ticket ID, subject, description, resolution notes, agent and customer info, priority, timestamps, tags, and category  
    - Key Expressions: Custom extraction functions handle multiple data structures (Zendesk, Freshdesk, etc.), fallback values for missing data  
    - Input: Raw JSON from webhook  
    - Output: Structured JSON object with normalized fields and original raw ticket included  
    - Edge Cases: Variations in field naming, missing resolution notes, empty tags, inconsistent dates  
    - Debug Tips: Console logs can be added to troubleshoot data mapping  

  - **📝 Sticky Note - Data Extraction**  
    - Type: Sticky Note  
    - Role: Documents purpose, fields extracted, error handling, and troubleshooting advice  

---

#### 1.3 AI Summarization

- **Overview:**  
Uses OpenAI GPT-4 to generate a clear, professional, and structured case study summary from the extracted ticket details.

- **Nodes Involved:**  
  - AI Summarization (OpenAI)  
  - 📝 Sticky Note - AI Summarization  

- **Node Details:**

  - **AI Summarization (OpenAI)**  
    - Type: OpenAI (Langchain)  
    - Role: Sends prompt with ticket info to GPT-4 to generate HTML-formatted summary with sections: Issue Reported, Troubleshooting Steps, Final Resolution, Key Takeaways  
    - Configuration:  
      - Model: GPT-4  
      - Temperature: 0.3 (for factual consistency)  
      - Max Tokens: 1500  
      - Prompt includes system and user messages with detailed instructions and ticket data interpolated dynamically  
    - Input: Normalized ticket JSON  
    - Output: AI-generated HTML summary text  
    - Edge Cases: API timeouts, rate limits, incomplete or vague summaries, prompt formatting errors  
    - Alternative: Can replace with Anthropic Claude via HTTP Request node  

  - **📝 Sticky Note - AI Summarization**  
    - Type: Sticky Note  
    - Role: Describes AI prompt design, output format, troubleshooting tips  

---

#### 1.4 HTML Formatting

- **Overview:**  
Combines ticket metadata and AI summary into a polished, branded, and print-ready HTML document with professional styling.

- **Nodes Involved:**  
  - Format HTML Document (Code)  
  - 📝 Sticky Note - HTML Formatting  

- **Node Details:**

  - **Format HTML Document**  
    - Type: Code (JavaScript)  
    - Role:  
      - Extracts AI summary and ticket data  
      - Calculates resolution time (days/hours)  
      - Formats dates for readability  
      - Generates HTML with embedded CSS styling for branding (green accents, badges, layout)  
      - Creates descriptive filename based on ticket ID, sanitized subject, and date  
    - Input: Normalized ticket JSON + AI summary JSON  
    - Output: Full HTML content string and metadata (filename, resolution time)  
    - Edge Cases: Special characters causing HTML breakage, missing fields, date parsing errors  
    - Tips: Test HTML output in browser before live usage  

  - **📝 Sticky Note - HTML Formatting**  
    - Type: Sticky Note  
    - Role: Explains design choices, file naming conventions, customization points, troubleshooting  

---

#### 1.5 PDF Conversion

- **Overview:**  
Converts the formatted HTML document into a PDF file using the external HTML/CSS to PDF API (PDFMunk).

- **Nodes Involved:**  
  - HTML to PDF  
  - 📝 Sticky Note - PDF Conversion  

- **Node Details:**

  - **HTML to PDF**  
    - Type: HTML/CSS to PDF node (htmlcsstopdf)  
    - Role: Converts the HTML content string into a PDF document  
    - Input: HTML content from previous node  
    - Output: PDF URL (hosted by PDFMunk)  
    - Edge Cases: Conversion failures due to invalid HTML, timeout, large content size  
    - Requires: PDFMunk API credentials  
      
  - **📝 Sticky Note - PDF Conversion**  
    - Type: Sticky Note  
    - Role: Details PDF node configuration, common errors, and troubleshooting  

---

#### 1.6 PDF Retrieval & Validation

- **Overview:**  
Downloads the PDF file from the URL returned by the conversion API and validates its successful generation before proceeding.

- **Nodes Involved:**  
  - Check PDF Success (If)  
  - HTTP Request (Download PDF)  
  - Sticky Note1 (PDF to Binary Conversion)  
  - 📝 Sticky Note - Error Check  

- **Node Details:**

  - **Check PDF Success**  
    - Type: If node  
    - Role: Checks if `success` field (PDF URL) is not empty, routes success or failure  
    - Input: PDF generation response  
    - Output: Routes to download or error Slack notification  

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Downloads PDF file from URL, outputting binary data  
    - Config: GET method, response format file, binary output to property `data`  
    - Input: PDF URL from previous node  
    - Output: Binary PDF data  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains purpose and configuration of HTTP Request node for PDF binary download  

  - **📝 Sticky Note - Error Check**  
    - Type: Sticky Note  
    - Role: Describes error checking logic, failure causes (invalid HTML, memory issues), and recovery steps including Slack alerts  

---

#### 1.7 Google Drive Upload

- **Overview:**  
Uploads the downloaded PDF binary file to a designated Google Drive folder for organized cloud storage.

- **Nodes Involved:**  
  - Upload to Google Drive  
  - Capture Drive URL (Set)  
  - Check Upload Success (If)  
  - 📝 Sticky Note - Drive Upload  
  - 📝 Sticky Note - Upload Check  

- **Node Details:**

  - **Upload to Google Drive**  
    - Type: Google Drive node  
    - Role: Uploads PDF binary data as a file with generated filename  
    - Config:  
      - Resource: File  
      - Operation: Upload  
      - Folder ID: Configured with Drive folder ID  
      - Filename: From previous node output  
    - Credential: Google Drive OAuth2  
    - Input: Binary PDF data  
    - Output: Metadata of uploaded file including ID and shareable URLs  

  - **Capture Drive URL**  
    - Type: Set node  
    - Role: Extracts and stores file ID, web view link, and name for downstream use  

  - **Check Upload Success**  
    - Type: If node  
    - Role: Validates presence of Drive file URL, routes success or failure  

  - **📝 Sticky Note - Drive Upload**  
    - Type: Sticky Note  
    - Role: Explains upload setup, folder ID extraction, folder organization ideas, permissions, and output data  

  - **📝 Sticky Note - Upload Check**  
    - Type: Sticky Note  
    - Role: Describes error checks for upload, common failure reasons, recovery, and monitoring best practices  

---

#### 1.8 Database Update

- **Overview:**  
Appends a new row to a Google Sheet tracking all documented tickets and their metadata for centralized logging and reporting.

- **Nodes Involved:**  
  - Update Google Sheets  
  - 📝 Sticky Note - Database  

- **Node Details:**

  - **Update Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends a row with ticket metadata, resolution time, PDF link, and status "Documented"  
    - Config:  
      - Operation: Append  
      - Sheet Name: "Sheet1"  
      - Document ID: Configured Google Sheet ID  
      - Columns mapped dynamically from extracted ticket and Drive upload data  
    - Credential: Google Sheets OAuth2  

  - **📝 Sticky Note - Database**  
    - Type: Sticky Note  
    - Role: Describes purpose, setup instructions for Google Sheet, data tracked, benefits, alternative databases, and reporting ideas  

---

#### 1.9 Slack Notification

- **Overview:**  
Sends a formatted Slack message to the support team channel informing them of successful ticket documentation with key details and a PDF link.

- **Nodes Involved:**  
  - Send Slack Notification  
  - 📝 Sticky Note - Slack  

- **Node Details:**

  - **Send Slack Notification**  
    - Type: Slack node  
    - Role: Posts markdown-formatted message summarizing ticket ID, subject, customer, priority, resolved by, resolution time, and PDF link  
    - Config:  
      - Channel: Configured Slack channel ID  
      - Authentication: OAuth2  
      - Message text uses n8n expressions to inject dynamic data  
    - Edge Cases: Slack API rate limits, invalid channel ID, auth expiration  

  - **📝 Sticky Note - Slack**  
    - Type: Sticky Note  
    - Role: Explains notification purpose, message content, customization, alternatives, and best practices  

---

#### 1.10 Error Handling

- **Overview:**  
Handles failures in PDF generation or Google Drive upload by sending detailed Slack alerts to admins for manual intervention.

- **Nodes Involved:**  
  - Error - PDF Failed (Slack)  
  - Error - Upload Failed (Slack)  
  - 📝 Sticky Note - Error Handling  

- **Node Details:**

  - **Error - PDF Failed**  
    - Type: Slack node  
    - Role: Sends alert when PDF generation fails, includes ticket ID, subject, timestamp, and action instructions  

  - **Error - Upload Failed**  
    - Type: Slack node  
    - Role: Sends alert when Drive upload fails, includes ticket info, possible causes, and next steps  

  - **📝 Sticky Note - Error Handling**  
    - Type: Sticky Note  
    - Role: Documents error scenarios covered, notification contents, recovery process, advanced error handling options, and monitoring best practices  

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                          | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                       |
|---------------------------|----------------------------------|----------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Webhook - Receive Ticket   | Webhook                          | Receives resolved ticket payload       |                              | Extract Ticket Details       | ## 🎯 STEP 1: TRIGGER - Explains webhook setup and expected JSON payload                        |
| Extract Ticket Details     | Code                             | Normalizes and extracts ticket data    | Webhook - Receive Ticket      | AI Summarization (OpenAI)    | ## 📋 STEP 2: DATA EXTRACTION - Details data normalization and fallback logic                   |
| AI Summarization (OpenAI)  | OpenAI (Langchain)               | Generates AI summary of the ticket     | Extract Ticket Details        | Format HTML Document         | ## 🤖 STEP 3: AI SUMMARIZATION - Describes AI prompt, output format, and troubleshooting        |
| Format HTML Document       | Code                             | Creates branded HTML report             | AI Summarization (OpenAI)     | HTML to PDF                 | ## 🎨 STEP 4: HTML FORMATTING - Explains styling, layout, file naming, and customization points |
| HTML to PDF               | HTML/CSS to PDF (htmlcsstopdf)   | Converts HTML to PDF                    | Format HTML Document          | Check PDF Success            | ## 📄 STEP 5: PDF CONVERSION - Details PDF conversion setup and common issues                    |
| Check PDF Success          | If                               | Validates PDF generation success       | HTML to PDF                  | HTTP Request / Error - PDF Failed | ## ⚠️ STEP 6: ERROR CHECK #1 - Describes PDF validation and failure handling                |
| HTTP Request               | HTTP Request                     | Downloads PDF as binary data            | Check PDF Success (true)      | Upload to Google Drive       | # PDF to Binary Conversion - Explains download config for binary data                           |
| Upload to Google Drive     | Google Drive                     | Uploads PDF to Drive folder             | HTTP Request                 | Capture Drive URL            | ## ☁️ STEP 7: GOOGLE DRIVE UPLOAD - Explains upload config, folder ID, and permissions         |
| Capture Drive URL          | Set                              | Extracts Drive file metadata            | Upload to Google Drive        | Check Upload Success         |                                                                                                 |
| Check Upload Success       | If                               | Validates Drive upload success          | Capture Drive URL             | Update Google Sheets / Error - Upload Failed | ## ⚠️ STEP 8: ERROR CHECK #2 - Upload validation and error alert                         |
| Update Google Sheets       | Google Sheets                    | Logs ticket metadata and links          | Check Upload Success (true)   | Send Slack Notification      | ## STEP 9: DATABASE - Describes logging setup, data tracked, and benefits                      |
| Send Slack Notification    | Slack                           | Notifies team of successful documentation | Update Google Sheets         |                             | ## 💬 STEP 10: SLACK NOTIFICATION - Details message format and customization                   |
| Error - PDF Failed         | Slack                           | Alerts on PDF generation failure        | Check PDF Success (false)     |                             | ## ERROR HANDLING - Covers error scenarios and recovery steps                                 |
| Error - Upload Failed      | Slack                           | Alerts on Drive upload failure           | Check Upload Success (false)  |                             |                                                                                                 |
| 📝 Sticky Note nodes       | Sticky Note                     | Documentation and instructions          | Various                      |                             | Multiple notes provide detailed explanations per workflow step                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: "Webhook - Receive Ticket"  
   - HTTP Method: POST  
   - Path: `support-ticket-resolved`  
   - Response Mode: `responseNode`  
   - Purpose: Receive JSON payload of resolved ticket  

2. **Add Code Node to Extract Ticket Details:**  
   - Name: "Extract Ticket Details"  
   - Paste JavaScript code to normalize ticket data from webhook payload (handle various support systems, fallback values for missing data)  
   - Output: JSON with keys like ticketId, subject, description, resolution, agentName, customerName, customerEmail, priority, createdAt, resolvedAt, tags, category, rawTicket  

3. **Add OpenAI Node for AI Summarization:**  
   - Name: "AI Summarization (OpenAI)"  
   - Model: GPT-4 (or GPT-3.5 as fallback)  
   - Temperature: 0.3  
   - Max Tokens: 1500  
   - Messages:  
     - System message: Define role as technical support documentation specialist  
     - User message: Provide ticket data using expressions for dynamic insertion; ask for structured HTML output with sections  
   - Credentials: Configure OpenAI API key credentials  

4. **Add Code Node to Format HTML Document:**  
   - Name: "Format HTML Document"  
   - JavaScript code to:  
     - Extract AI summary text  
     - Format dates and calculate resolution time  
     - Generate branded HTML document with embedded CSS and badges  
     - Create sanitized descriptive filename for PDF  
   - Input: Output from AI Summarization and Extract Ticket Details nodes  

5. **Add HTML to PDF Node:**  
   - Name: "HTML to PDF"  
   - Operation: Convert HTML to PDF  
   - Input: HTML content from "Format HTML Document"  
   - Credentials: PDFMunk API key (HtmlcsstopdfApi)  

6. **Add If Node to Check PDF Success:**  
   - Name: "Check PDF Success"  
   - Condition: Check if PDF generation response `success` field is not empty  
   - True: Proceed to download PDF  
   - False: Trigger Slack error notification (PDF Failed)  

7. **Add HTTP Request Node to Download PDF:**  
   - Name: "HTTP Request"  
   - Method: GET  
   - URL: Use PDF URL from previous node output  
   - Response Format: File  
   - Output Binary Property: `data`  
   - No authentication required  

8. **Add Google Drive Upload Node:**  
   - Name: "Upload to Google Drive"  
   - Resource: File  
   - Operation: Upload  
   - File Binary Property: `data` from HTTP Request node  
   - Filename: Use generated filename from HTML formatting node  
   - Folder ID: Set your Google Drive target folder ID  
   - Credentials: Google Drive OAuth2  

9. **Add Set Node to Capture Drive URL and Metadata:**  
   - Name: "Capture Drive URL"  
   - Fields: Extract `id`, `webViewLink`, and `name` from Drive upload response  

10. **Add If Node to Check Upload Success:**  
    - Name: "Check Upload Success"  
    - Condition: Drive file URL is not empty  
    - True: Proceed to update Google Sheets  
    - False: Trigger Slack error notification (Upload Failed)  

11. **Add Google Sheets Append Node:**  
    - Name: "Update Google Sheets"  
    - Operation: Append row  
    - Sheet Name: "Sheet1"  
    - Document ID: Your Google Sheet ID  
    - Map columns for Ticket ID, Subject, Customer Name/Email, Agent Name, Priority, Category, Resolved Date, Resolution Time, PDF Link, Document Generated, Status = "Documented"  
    - Credentials: Google Sheets OAuth2  

12. **Add Slack Node to Notify Team:**  
    - Name: "Send Slack Notification"  
    - Channel: Your support team's Slack channel ID  
    - Message: Markdown with ticket ID, subject, customer, priority, agent, resolution time, and PDF link  
    - Credentials: Slack OAuth2 API  

13. **Add Slack Nodes for Error Notifications:**  
    - Names: "Error - PDF Failed" and "Error - Upload Failed"  
    - Configure messages detailing failure cause, ticket info, timestamp, and action required  
    - Same Slack channel as success notification or dedicated admin channel  
    - Triggered from respective If nodes on failure branches  

14. **Add Sticky Notes for Documentation:**  
    - Add descriptive sticky notes on each major block explaining purpose, setup, troubleshooting, and configuration references  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow credentials setup includes OpenAI API, PDFMunk API (for HTML to PDF), Google Drive OAuth2, Google Sheets OAuth2, Slack OAuth2                | See sticky note titled "Workflow Credentials Setup"                                                    |
| Folder ID and Sheet ID must be extracted from respective Google URLs for Drive upload and Sheets logging                                           | Folder ID example: `drive.google.com/drive/folders/1ABC123XYZ`                                        |
| Slack Channel IDs format: public channels start with `C`, private with `G`, direct messages with `D`                                                | Use Slack UI to copy exact channel ID                                                                  |
| Alternative AI summarization can be implemented using Anthropic Claude model via HTTP Request node                                                  | See sticky note in AI Summarization block                                                             |
| PDF conversion may take several seconds; test with sample data to ensure formatting and content correctness                                        | See sticky note "PDF Conversion"                                                                       |
| Error handling best practices include retry logic, daily error summaries, and centralized error workflows                                         | See sticky note "Error Handling"                                                                       |
| Output PDF filenames sanitize special characters and limit subject length to 50 characters                                                         | See code in "Format HTML Document" node                                                               |
| HTML formatting includes embedded CSS with branding colors (#4CAF50 green) and can be customized for company branding                             | See sticky note "HTML Formatting"                                                                      |
| Slack notifications use rich markdown for clarity and clickable PDF links                                                                           | See sticky note "Slack Notification"                                                                   |
| Google Sheets logging enables searchable ticket documentation, reporting, and analytics                                                             | See sticky note "Database"                                                                              |

---

This detailed reference document enables understanding, modification, and reconstruction of the "Customer Support Ticket Workflow Automation" in n8n, supporting robust AI-powered documentation from support tickets.