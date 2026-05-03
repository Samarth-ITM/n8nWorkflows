Extract and Organize Contract Details from PDFs with Slack, GPT-4o, and Google Sheets

https://n8nworkflows.xyz/workflows/extract-and-organize-contract-details-from-pdfs-with-slack--gpt-4o--and-google-sheets-8440


# Extract and Organize Contract Details from PDFs with Slack, GPT-4o, and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction and organization of key contract details from PDF or Word files uploaded to a specific Slack channel. It is designed for operations, legal teams, or businesses managing multiple contracts, aiming to reduce manual data entry and improve contract management accuracy.

The workflow is logically structured into these blocks:

- **1.1 Input Reception:** Receives contract files uploaded on Slack in a designated channel.
- **1.2 File Format Validation and Routing:** Checks the uploaded file format to route PDFs and Word documents correctly, or reject unsupported formats.
- **1.3 File Download and Text Extraction:** Downloads the contract file, converts Word files to PDF if necessary, and extracts text content.
- **1.4 AI-Powered Contract Analysis:** Uses GPT-4o to parse extracted text and extract structured contract data fields.
- **1.5 Data Storage:** Appends the extracted contract details into a Google Sheets spreadsheet.
- **1.6 Notification:** Posts a summary message back to Slack confirming successful data capture.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens to a Slack channel for incoming messages containing contract files and triggers the workflow upon detection.

- **Nodes Involved:**  
  - Receive Contract File

- **Node Details:**

  - **Receive Contract File**  
    - Type: Slack Trigger  
    - Role: Initiates workflow on new message event in a specific Slack channel  
    - Configuration:  
      - Trigger on message event  
      - Channel ID set to the contract submission channel (e.g., `C09EG2EN9AA`)  
      - Uses Slack API credentials for authentication  
    - Inputs: Slack event webhook  
    - Outputs: JSON with file metadata (including filetype and download URL)  
    - Edge Cases:  
      - Missing or malformed file data in Slack message  
      - Slack API rate limits or authorization errors  

#### 2.2 File Format Validation and Routing

- **Overview:**  
  Validates the uploaded file format and routes it for appropriate processing: PDF files are downloaded directly, Word documents are converted to PDF, and unsupported formats trigger an error message.

- **Nodes Involved:**  
  - Check File Format  
  - Send Error Message  
  - Download PDF  
  - Convert Word to PDF  

- **Node Details:**

  - **Check File Format**  
    - Type: Switch  
    - Role: Branches workflow based on file extension (`pdf`, `docx`, or others)  
    - Configuration:  
      - Checks `$json.files[0].filetype` equality against `pdf` or `docx`  
      - Outputs: Routes to “PDF”, “WORD”, or “Others”  
    - Inputs: File metadata from Slack trigger  
    - Outputs: Conditional routing to Download PDF, Convert Word to PDF, or Send Error Message nodes  
    - Edge Cases:  
      - Unsupported file types (e.g., images, txt)  
      - Missing `filetype` property  

  - **Send Error Message**  
    - Type: Slack  
    - Role: Sends notification to Slack channel if unsupported file type is uploaded  
    - Configuration:  
      - Sends a fixed message “Only PDF or Word format contracts can be uploaded.”  
      - Posts to the same Slack channel as input  
      - Authenticated with Slack API  
    - Inputs: Triggered if file format is unsupported  
    - Outputs: Ends workflow branch  
    - Edge Cases: Slack API errors, message posting failures  

  - **Download PDF**  
    - Type: HTTP Request  
    - Role: Downloads the PDF file using private Slack file URL  
    - Configuration:  
      - URL parameter set dynamically to `$json.files[0].url_private_download`  
      - Response format set to file (binary)  
      - Uses Slack API authentication to access private file  
    - Inputs: PDF file metadata from switch node  
    - Outputs: Binary file data passed downstream  
    - Edge Cases: URL expiration, Slack API auth errors, network timeouts  

  - **Convert Word to PDF**  
    - Type: HTTP Request  
    - Role: Downloads Word document converted to PDF (assumes Slack provides a converted PDF URL)  
    - Configuration:  
      - URL parameter set dynamically to `$json.files[0].converted_pdf`  
      - Response format set to file (binary)  
      - Uses Slack API authentication  
    - Inputs: Word file metadata from switch node  
    - Outputs: Binary PDF file data for further processing  
    - Edge Cases: Missing `converted_pdf` URL, Slack API errors  

#### 2.3 File Download and Text Extraction

- **Overview:**  
  Extracts text from the binary PDF file, whether originally uploaded as PDF or converted from Word.

- **Nodes Involved:**  
  - Extract Text from PDF  
  - Extract Text from PDF1  

- **Node Details:**

  - **Extract Text from PDF**  
    - Type: Extract from File  
    - Role: Extracts plain text from PDF binary data  
    - Configuration:  
      - Operation set to `pdf`  
      - Binary property name set to `data` (the binary file input)  
    - Inputs: PDF binary from Download PDF node  
    - Outputs: Extracted text as JSON property `text`  
    - Edge Cases: Corrupt or encrypted PDFs, extraction failures  

  - **Extract Text from PDF1**  
    - Same configuration and role as above but processes PDFs converted from Word files  
    - Inputs: PDF binary from Convert Word to PDF node  

#### 2.4 AI-Powered Contract Analysis

- **Overview:**  
  Uses GPT-4o to analyze extracted text and extract structured contract details such as client, service provider, dates, and contract value.

- **Nodes Involved:**  
  - AI model  
  - Structure Output  
  - Analyze Contract Content  

- **Node Details:**

  - **AI model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Performs GPT-4o language model inference  
    - Configuration:  
      - Model set to `gpt-4o`  
      - Uses OpenAI API credentials  
    - Inputs: Text prompt from Analyze Contract Content node  
    - Outputs: Raw AI response JSON  

  - **Structure Output**  
    - Type: Langchain Output Parser (Structured)  
    - Role: Parses AI output into structured JSON according to schema  
    - Configuration:  
      - JSON schema example provided with keys: Client, Service Provider, Effective Date, Expiration Date, Signature Date, Contract Value  
    - Inputs: Raw AI output from AI model node  
    - Outputs: Parsed structured contract data  
    - Edge Cases: Parsing errors if AI output deviates from schema  

  - **Analyze Contract Content**  
    - Type: Langchain Agent  
    - Role: Main node orchestrating prompt and output parsing  
    - Configuration:  
      - Prompt instructs AI to extract specific contract fields from input text  
      - Input text injected dynamically from previous text extraction nodes  
      - Output parser enabled  
    - Inputs: Extracted text from PDF extraction nodes  
    - Outputs: Parsed contract details JSON  
    - Edge Cases: AI understanding errors, empty or malformed text input  

#### 2.5 Data Storage

- **Overview:**  
  Saves parsed contract data as a new row in a Google Sheets spreadsheet for contract tracking.

- **Nodes Involved:**  
  - Save to Google Sheets  

- **Node Details:**

  - **Save to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends new row with contract data fields  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet ID targeting specific contract tracker spreadsheet  
      - Maps fields: Client, Service Provider, Effective Date, Expiration Date, Signature Date, Contract Value  
      - Uses Google OAuth2 credentials  
    - Inputs: Structured JSON from Analyze Contract Content node  
    - Outputs: Confirmation of append operation  
    - Edge Cases: API access errors, permission issues, schema mismatch  

#### 2.6 Notification

- **Overview:**  
  Posts a summary message back to the Slack channel confirming the contract data has been saved.

- **Nodes Involved:**  
  - Notify on Slack  

- **Node Details:**

  - **Notify on Slack**  
    - Type: Slack  
    - Role: Sends a formatted summary of extracted contract data to Slack channel  
    - Configuration:  
      - Message text includes key fields: Client, Service Provider, Expiration Date, Effective Date, Signature Date, Contract Value  
      - Posts to the same Slack channel as input  
      - Uses Slack API credentials  
    - Inputs: Data from Save to Google Sheets node  
    - Outputs: Ends workflow  
    - Edge Cases: Slack API posting errors  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                       |
|-----------------------|----------------------------------|---------------------------------------|-------------------------|----------------------------|--------------------------------------------------------------------------------------------------|
| Receive Contract File  | Slack Trigger                    | Trigger workflow on Slack file upload | —                       | Check File Format           | Receives contract file uploaded via Slack and triggers workflow                                  |
| Check File Format      | Switch                          | Routes by file extension               | Receive Contract File    | Download PDF, Convert Word to PDF, Send Error Message | Checks file format: PDF → Download PDF, Word → Convert Word to PDF, Others → Send Error Message  |
| Download PDF          | HTTP Request                    | Downloads contract PDF file            | Check File Format (PDF)  | Extract Text from PDF       |                                                                                                  |
| Convert Word to PDF    | HTTP Request                    | Downloads converted Word to PDF        | Check File Format (WORD) | Extract Text from PDF1      |                                                                                                  |
| Extract Text from PDF  | Extract from File               | Extracts text from PDF binary          | Download PDF             | Analyze Contract Content    |                                                                                                  |
| Extract Text from PDF1 | Extract from File               | Extracts text from PDF converted from Word | Convert Word to PDF      | Analyze Contract Content    |                                                                                                  |
| Send Error Message     | Slack                          | Sends error message for unsupported formats | Check File Format (Others) | —                          | Only PDF or Word format contracts can be uploaded.                                              |
| Analyze Contract Content| Langchain Agent                | Extracts structured contract data via GPT | Extract Text from PDF, Extract Text from PDF1 | Save to Google Sheets        | Analyzes contract content to extract key fields                                                 |
| AI model              | Langchain OpenAI Chat Model    | Runs GPT-4o inference                  | Analyze Contract Content | Structure Output            |                                                                                                  |
| Structure Output      | Langchain Output Parser        | Parses GPT output into structured JSON | AI model                 | Analyze Contract Content    |                                                                                                  |
| Save to Google Sheets | Google Sheets                  | Appends contract data to spreadsheet  | Analyze Contract Content | Notify on Slack             |                                                                                                  |
| Notify on Slack       | Slack                         | Posts confirmation message to Slack   | Save to Google Sheets    | —                          |                                                                                                  |
| Sticky Note           | Sticky Note                   | Documentation                         | —                       | —                          | See respective detailed sticky notes for block explanations                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger node: "Receive Contract File"**  
   - Type: Slack Trigger  
   - Trigger on message event in Slack channel for contract uploads  
   - Select Slack credentials with API access  
   - Set Channel ID to your contracts Slack channel  

2. **Add Switch node: "Check File Format"**  
   - Evaluate file extension: `$json.files[0].filetype`  
   - Add rules:  
     - If `pdf` → output "PDF"  
     - If `docx` → output "WORD"  
     - Else → output "Others"  

3. **Add Slack node: "Send Error Message"**  
   - Triggered by "Others" output  
   - Message text: "Only PDF or Word format contracts can be uploaded."  
   - Post to the contracts Slack channel using Slack credentials  

4. **Add HTTP Request node: "Download PDF"**  
   - Triggered by "PDF" output  
   - URL: set to `$json.files[0].url_private_download`  
   - Response format: file (binary)  
   - Authentication: Slack API credentials  

5. **Add HTTP Request node: "Convert Word to PDF"**  
   - Triggered by "WORD" output  
   - URL: set to `$json.files[0].converted_pdf` (assumes Slack provides converted PDF URL)  
   - Response format: file (binary)  
   - Authentication: Slack API credentials  

6. **Add Extract from File node: "Extract Text from PDF"**  
   - Triggered from "Download PDF"  
   - Operation: PDF  
   - Binary property: `data`  

7. **Add Extract from File node: "Extract Text from PDF1"**  
   - Triggered from "Convert Word to PDF"  
   - Operation: PDF  
   - Binary property: `data`  

8. **Add Langchain Agent node: "Analyze Contract Content"**  
   - Triggered from both PDF extraction nodes  
   - Prompt:  
     ```
     please read and understand the input data({{ $json.text }}). I would like you to extract Client, Service Provider, Effective Date, Expiration Date, Signature Date and Contract Value.
     ```  
   - Enable output parser  

9. **Add Langchain OpenAI Chat Model node: "AI model"**  
   - Model: `gpt-4o`  
   - Credentials: OpenAI API key  
   - Connected as AI language model for "Analyze Contract Content"  

10. **Add Langchain Output Parser node: "Structure Output"**  
    - JSON schema example:  
      ```json
      [{
        "Client": "XYZ Inc",
        "Service Provider": "ABC Inc",
        "Effective Date": "2025/04/29",
        "Expiration Date": "2025/05/29",
        "Signature Date": "2025/05/29",
        "Contract Value": "11,000"
      }]
      ```  
    - Connected as AI output parser for "Analyze Contract Content"  

11. **Add Google Sheets node: "Save to Google Sheets"**  
    - Operation: Append  
    - Document ID: your target Google Sheets document ID  
    - Sheet Name: target sheet's GID or name  
    - Map columns: Client, Service Provider, Effective Date, Expiration Date, Signature Date, Contract Value from AI parsed output  
    - Credentials: Google Sheets OAuth2  

12. **Add Slack node: "Notify on Slack"**  
    - Triggered after Google Sheets append  
    - Message text:  
      ```
      ---
      Client: {{ $json.Client }}
      Service Provider: {{ $json['Service Provider'] }}
      Expiration Date: {{ $json['Expiration Date'] }}
      Effective Date: {{ $json['Effective Date'] }}
      Signature Date: {{ $json['Signature Date'] }}
      Contract Value: {{ $json['Contract Value'] }}
      ```  
    - Post to the contracts Slack channel  
    - Credentials: Slack API  

13. **Connect nodes in sequence:**  
    - Receive Contract File → Check File Format  
    - Check File Format → Download PDF / Convert Word to PDF / Send Error Message  
    - Download PDF → Extract Text from PDF → Analyze Contract Content  
    - Convert Word to PDF → Extract Text from PDF1 → Analyze Contract Content  
    - Analyze Contract Content → AI model → Structure Output (as AI output parser)  
    - Analyze Contract Content → Save to Google Sheets → Notify on Slack  

14. **Test with sample contract files uploaded in Slack channel.**  
15. **Adjust GPT prompt or Google Sheets mapping if needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                                                                                                                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Managing contracts manually is error-prone. This workflow automates contract data capture from Slack uploads using GPT-4o and stores results in Google Sheets, notifying teams via Slack. Ideal for operations, legal, and growing businesses.                                                                                                                                                                | Sticky Note6 content in the workflow JSON (detailed project description and setup instructions).                                                                                                                                                                                                                                                               |
| Only PDF and Word file formats are supported for contract uploads. Attempts to upload other formats trigger an error message in Slack.                                                                                                                                                                                                                                                                    | Sticky Note content near the "Check File Format" node.                                                                                                                                                                                                                                                                                                         |
| The GPT prompt can be customized to extract additional or alternative contract fields as per organizational needs.                                                                                                                                                                                                                                                                                          | Editable in "Analyze Contract Content" node prompt parameter.                                                                                                                                                                                                                                                                                                  |
| Google Sheets requires proper column headers matching the contract fields for data appending to work correctly.                                                                                                                                                                                                                                                                                              | Defined in "Save to Google Sheets" node mapping.                                                                                                                                                                                                                                                                                                               |
| Slack API credentials must have permissions to read messages, download files, and post messages in the relevant channels.                                                                                                                                                                                                                                                                                    | Credentials used in Slack Trigger, HTTP Request (for download), Slack message nodes.                                                                                                                                                                                                                                                                            |
| OpenAI API key with access to GPT-4o is required for reliable contract data extraction.                                                                                                                                                                                                                                                                                                                      | Credentials used in "AI model" node.                                                                                                                                                                                                                                                                                                                           |

---

This completes the comprehensive reference documentation for the "Extract and Organize Contract Details from PDFs with Slack, GPT-4o, and Google Sheets" workflow. It enables understanding, reproduction, and customization by both advanced users and automation agents.