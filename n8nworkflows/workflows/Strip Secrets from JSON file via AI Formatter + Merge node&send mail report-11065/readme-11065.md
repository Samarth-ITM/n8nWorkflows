Strip Secrets from JSON file via AI Formatter + Merge node&send mail report

https://n8nworkflows.xyz/workflows/strip-secrets-from-json-file-via-ai-formatter---merge-node-send-mail-report-11065


# Strip Secrets from JSON file via AI Formatter + Merge node&send mail report

### 1. Workflow Overview

This workflow is designed to sanitize an uploaded n8n workflow JSON file by removing all sensitive information such as credentials, webhook IDs, tokens, and user-specific metadata. It then generates a sanitized JSON version that is safe to share or publish, produces a detailed AI-generated change log comparing the original and sanitized workflows, and finally emails both the sanitized JSON file and the change report to the user.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Collect user email and JSON file upload via a form.
- **1.2 JSON Extraction and Preparation:** Extract and prepare the uploaded JSON for sanitization.
- **1.3 AI-Based Sanitization:** Use OpenAI to remove secrets and sensitive data, producing sanitized JSON.
- **1.4 Formatting and Comparison:** Format the sanitized JSON and generate a detailed change log comparing original vs sanitized workflows.
- **1.5 File Creation and Email Sending:** Convert sanitized JSON to a file, merge with the report, and send an email to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects the user’s email and the JSON file to be sanitized through a web form trigger.

**Nodes Involved:**  
- Upload Workflow Json  
- Sticky Note (Input Collection)

**Node Details:**  

- **Upload Workflow Json**  
  - Type: Form Trigger  
  - Role: Entry point for workflow, triggers on form submission  
  - Configuration: Form with two fields: "Mailid" (email) and "Upload JSON file" (file upload)  
  - Inputs: HTTP form submission  
  - Outputs: Contains email and uploaded JSON file as binary data  
  - Edge Cases: Form validation errors, missing file or email, upload size limits  
  - Sticky Note: "Input Collection - Form node collects MailId and uploaded JSON."

---

#### 2.2 JSON Extraction and Preparation

**Overview:**  
Extracts the JSON content from the uploaded file, wraps it for processing, and strips known heavy or unnecessary fields before AI sanitization.

**Nodes Involved:**  
- Extract JSON content  
- Prepare Original Workflow Structure  
- Format Original Workflow(JS)  
- Sticky Note (Extract + Prepare Original Workflow)

**Node Details:**  

- **Extract JSON content**  
  - Type: Extract From File  
  - Role: Parses the uploaded binary JSON file to a JSON object  
  - Configuration: Operation "fromJson"; reads from binary property "Upload_JSON_file"; output key "workflowJson"  
  - Inputs: Upload Workflow Json  
  - Outputs: Parsed JSON of the workflow file  
  - Edge Cases: Invalid JSON file, corrupted upload, empty file  

- **Prepare Original Workflow Structure**  
  - Type: Set  
  - Role: Wraps parsed JSON under "originalWorkflowJson" key for clarity  
  - Configuration: Assigns parsed JSON to "originalWorkflowJson"  
  - Inputs: Extract JSON content  
  - Outputs: Payload with original workflow JSON under a single key  

- **Format Original Workflow(JS)**  
  - Type: Code  
  - Role: Recursively removes fields known to be heavy or sensitive (credentials, webhookId, root-level meta, versionId, id) before AI sanitization  
  - Configuration: JavaScript function `walk` recursively cleans the object  
  - Inputs: Prepare Original Workflow Structure  
  - Outputs: JSON containing original and pre-trimmed workflow JSON  
  - Edge Cases: Unexpected JSON structures, large workflows causing performance overhead  
  - Sticky Note: "Extract + Prepare Original Workflow - Extracts uploaded file into structured JSON. Wraps parsed JSON into originalWorkflowJson. Stringifies and prepares payload for sanitization."

---

#### 2.3 AI-Based Sanitization

**Overview:**  
Uses OpenAI GPT-4.1-mini to sanitize the workflow JSON by removing all secrets, credentials, webhook IDs, and user metadata, strictly following defined rules. Parses the AI response back into usable JSON.

**Nodes Involved:**  
- AI Sanitize Workflow JSON  
- Format Sanitized Workflow (JS)  
- Sticky Note (AI Sanitization Engine)

**Node Details:**  

- **AI Sanitize Workflow JSON**  
  - Type: OpenAI (Langchain integration)  
  - Role: Sends the pre-trimmed JSON to AI with detailed system instructions to sanitize secrets  
  - Configuration: Model "gpt-4.1-mini"; prompt includes strict sanitization rules (remove credentials, webhookId, replace secrets with placeholders, preserve sticky notes intact, etc.)  
  - Inputs: Format Original Workflow(JS) output (preTrimmedWorkflowJson)  
  - Outputs: Raw AI response containing sanitized workflow JSON as plain JSON string  
  - Credentials: OpenAI API credentials required  
  - Edge Cases: AI API errors, timeouts, malformed AI output, prompt length limits, malformed JSON in response  

- **Format Sanitized Workflow (JS)**  
  - Type: Code  
  - Role: Parses AI's JSON string output into an object and prepares a pretty-printed string for file export  
  - Configuration: JavaScript to parse and JSON.stringify with indentation  
  - Inputs: AI Sanitize Workflow JSON  
  - Outputs: sanitizedWorkflowJson (object), sanitizedText (string)  
  - Edge Cases: Invalid JSON from AI, parsing errors  

---

#### 2.4 Formatting and Comparison

**Overview:**  
Combines the original and sanitized JSONs, then uses AI to generate a structured HTML change log highlighting modifications, placeholders inserted, and summary of removed secrets.

**Nodes Involved:**  
- Combine Original & Sanitized JSON  
- Generate Workflow Change log (AI)  
- Sticky Note (Comparison Engine)

**Node Details:**  

- **Combine Original & Sanitized JSON**  
  - Type: Merge  
  - Role: Combines the original and sanitized JSON objects side-by-side for comparison input  
  - Configuration: Mode "combine", include unpaired nodes, combine by position  
  - Inputs: Format Sanitized Workflow (JS) and Format Original Workflow(JS)  
  - Outputs: Combined JSON with both versions  

- **Generate Workflow Change log (AI)**  
  - Type: OpenAI (Langchain integration)  
  - Role: Compares the original and sanitized workflows and generates an HTML-formatted change log detailing node-level changes, placeholders, and summary  
  - Configuration: Model "gpt-4.1-mini"; prompt requesting structured HTML output with headings and lists (no markdown, no <html>/<body> tags)  
  - Inputs: Combine Original & Sanitized JSON  
  - Outputs: HTML content of the change log  
  - Credentials: OpenAI API credentials  
  - Edge Cases: AI API errors, malformed HTML output, prompt length limits  

---

#### 2.5 File Creation and Email Sending

**Overview:**  
Converts the sanitized JSON string into a downloadable JSON file, merges it with the AI-generated report, and sends an HTML email with the report and sanitized file attached to the user’s email.

**Nodes Involved:**  
- Create Sanitized JSON File  
- Assemble Email Content & Attachment  
- Email Sanitized Workflow + Report  
- Sticky Note (Create Sanitized JSON File, Assemble Email & Send)

**Node Details:**  

- **Create Sanitized JSON File**  
  - Type: Convert To File  
  - Role: Converts the pretty-printed sanitized JSON string into a text file named "sanitized_workflow.json" for attachment  
  - Configuration: Operation "toText"; sourceProperty is sanitizedText; output binaryProperty "sanitizedFile"  
  - Inputs: Generate Workflow Change log (AI) via Combine Original & Sanitized JSON  
  - Outputs: Binary file data for attachment  
  - Edge Cases: Extremely large JSON causing memory issues  

- **Assemble Email Content & Attachment**  
  - Type: Merge  
  - Role: Combines the HTML change log report and the binary sanitized JSON file into one payload for sending  
  - Configuration: Mode "combine", include unpaired nodes, combine by position  
  - Inputs: Create Sanitized JSON File (binary file) and Generate Workflow Change log (AI) (HTML report)  
  - Outputs: Combined data for email  

- **Email Sanitized Workflow + Report**  
  - Type: Gmail  
  - Role: Sends the email to the user with the sanitized workflow JSON file attached and the HTML report as the email body  
  - Configuration:  
    - SendTo: user's email from Upload Workflow Json node  
    - Subject: "Sanitized n8n workflow JSON: [original filename]"  
    - Message: HTML content from AI change log  
    - Attachments: binary sanitized JSON file  
    - OAuth2 credentials for Gmail  
  - Inputs: Assemble Email Content & Attachment  
  - Edge Cases: Authentication errors, email delivery failures, large attachments, invalid email addresses  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                          | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                  |
|-------------------------------|--------------------------------|----------------------------------------|--------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------|
| Upload Workflow Json           | Form Trigger                   | Collect email and JSON file upload     | Trigger (HTTP form submission) | Extract JSON content                   | Input Collection - Form node collects MailId and uploaded JSON.                             |
| Extract JSON content           | Extract From File              | Parse uploaded JSON file to JSON       | Upload Workflow Json           | Prepare Original Workflow Structure    | Extract + Prepare Original Workflow - Extracts uploaded file into structured JSON.          |
| Prepare Original Workflow Structure | Set                          | Wrap JSON under originalWorkflowJson   | Extract JSON content           | Format Original Workflow(JS)            | Extract + Prepare Original Workflow - Wraps parsed JSON into originalWorkflowJson.          |
| Format Original Workflow(JS)   | Code                          | Remove credentials and metadata fields | Prepare Original Workflow Structure | AI Sanitize Workflow JSON           | Extract + Prepare Original Workflow - Stringifies and prepares payload for sanitization.    |
| AI Sanitize Workflow JSON      | OpenAI (Langchain)             | AI sanitization of workflow JSON       | Format Original Workflow(JS)   | Format Sanitized Workflow (JS)          | AI Sanitization Engine - Removes secrets & metadata while preserving structure.              |
| Format Sanitized Workflow (JS) | Code                          | Parse AI output and prepare file string | AI Sanitize Workflow JSON      | Combine Original & Sanitized JSON       | AI Sanitization Engine - Parses sanitized JSON into sanitizedWorkflowJson                    |
| Combine Original & Sanitized JSON | Merge                        | Combine original and sanitized JSON    | Format Sanitized Workflow (JS), Format Original Workflow(JS) | Generate Workflow Change log (AI), Create Sanitized JSON File | Comparison Engine - Merge node pairs both versions for comparison.                          |
| Generate Workflow Change log (AI) | OpenAI (Langchain)           | Generate HTML change log comparing JSONs | Combine Original & Sanitized JSON | Assemble Email Content & Attachment    | Comparison Engine - Produces Markdown/HTML-ready diff report.                               |
| Create Sanitized JSON File     | Convert To File               | Convert sanitized JSON string to file  | Generate Workflow Change log (AI) | Assemble Email Content & Attachment    | Create Sanitized JSON File - Converts sanitized JSON string into sanitized_workflow.json     |
| Assemble Email Content & Attachment | Merge                        | Combine report and file for email      | Generate Workflow Change log (AI), Create Sanitized JSON File | Email Sanitized Workflow + Report    | Assemble Email & Send - Merges report and file. Gmail node sends HTML email and attachment. |
| Email Sanitized Workflow + Report | Gmail                        | Send email with report and attachment  | Assemble Email Content & Attachment | None                                  | Assemble Email & Send - Sends HTML email and attachment to user-entered address.            |
| Sticky Note                   | Sticky Note                   | Documentation and explanation notes    | None                          | None                                   | Various contextual notes throughout the workflow.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: "Upload Workflow Json"  
   - Type: Form Trigger  
   - Configure a form with two fields:  
     - Email field labeled "Mailid"  
     - File upload field labeled "Upload JSON file"  
   - Set webhook to receive form data  

2. **Add Extract From File Node**  
   - Name: "Extract JSON content"  
   - Type: Extract From File  
   - Operation: fromJson  
   - Binary Property Name: "Upload_JSON_file" (matches file field)  
   - Destination Key: "workflowJson"  
   - Connect from "Upload Workflow Json"  

3. **Add Set Node**  
   - Name: "Prepare Original Workflow Structure"  
   - Type: Set  
   - Assign a new JSON key "originalWorkflowJson" with the value of the parsed JSON (`{{$json.workflowJson}}`)  
   - Connect from "Extract JSON content"  

4. **Add Code Node for JSON Cleaning**  
   - Name: "Format Original Workflow(JS)"  
   - Type: Code (JavaScript)  
   - Paste the recursive function to remove `credentials`, `webhookId`, and root-level keys like `meta`, `versionId`, `id` (as per original code)  
   - Return two keys: `originalWorkflowJson` and `preTrimmedWorkflowJson`  
   - Connect from "Prepare Original Workflow Structure"  

5. **Add OpenAI Node for Sanitization**  
   - Name: "AI Sanitize Workflow JSON"  
   - Type: OpenAI Node (Langchain)  
   - Model: GPT-4.1-mini  
   - Prompt: System prompt instructing to remove credentials, webhookIds, user metadata; keep sticky notes untouched; replace secrets with placeholders; output JSON only  
   - Pass `preTrimmedWorkflowJson` or fallback to `originalWorkflowJson` stringified as input  
   - Connect from "Format Original Workflow(JS)"  
   - Set OpenAI API credentials  

6. **Add Code Node to Parse AI Output**  
   - Name: "Format Sanitized Workflow (JS)"  
   - Type: Code  
   - Parse the AI JSON string output from previous node  
   - Return keys: `sanitizedWorkflowJson` (object), `sanitizedText` (pretty JSON string)  
   - Connect from "AI Sanitize Workflow JSON"  

7. **Add Merge Node to Combine JSONs**  
   - Name: "Combine Original & Sanitized JSON"  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Include Unpaired: True  
   - Connect inputs from both "Format Sanitized Workflow (JS)" and "Format Original Workflow(JS)"  

8. **Add OpenAI Node for Change Log Generation**  
   - Name: "Generate Workflow Change log (AI)"  
   - Type: OpenAI Node (Langchain)  
   - Model: GPT-4.1-mini  
   - Prompt: Compare original and sanitized JSONs; output HTML change log with headings and bullet points; no markdown or HTML/body wrapping  
   - Connect from "Combine Original & Sanitized JSON"  
   - Use same OpenAI credentials  

9. **Add Convert To File Node**  
   - Name: "Create Sanitized JSON File"  
   - Type: Convert To File  
   - Operation: toText  
   - Source Property: `sanitizedText`  
   - File Name: "sanitized_workflow.json"  
   - Connect from "Generate Workflow Change log (AI)" (output on the appropriate merge input)  

10. **Add Merge Node to Assemble Email Content & Attachment**  
    - Name: "Assemble Email Content & Attachment"  
    - Type: Merge  
    - Mode: Combine  
    - Combine By: Position  
    - Include Unpaired: True  
    - Connect inputs from "Generate Workflow Change log (AI)" (HTML report) and "Create Sanitized JSON File" (binary file)  

11. **Add Gmail Node to Send Email**  
    - Name: "Email Sanitized Workflow + Report"  
    - Type: Gmail  
    - Send To: Expression referencing `Mailid` from "Upload Workflow Json" node  
    - Subject: "Sanitized n8n workflow JSON: " + original uploaded file name  
    - Message: Use HTML content from the "Generate Workflow Change log (AI)" node  
    - Attachments: Attach the binary file from "Create Sanitized JSON File" node  
    - Configure OAuth2 Gmail credentials  
    - Connect from "Assemble Email Content & Attachment"  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow sanitizes any uploaded n8n workflow JSON by removing credentials, webhook IDs, tokens, and sensitive metadata. It generates a clean, secure version, produces a detailed change-log, and emails both the sanitized file and report to the user. | Sticky Note at workflow start |
| Join n8n Discord https://discord.com/invite/n8n or Community Forum https://community.n8n.io/ for support. | Community links |
| README file available at https://bit.ly/GeneratesanitizedJSONfile | Documentation link |
| Requires OpenAI API credentials and Gmail OAuth2 credentials for email sending | Setup requirements |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. All processing complies strictly with content policies, contains no illegal or offensive material, and manipulates only legal and publicly accessible data.