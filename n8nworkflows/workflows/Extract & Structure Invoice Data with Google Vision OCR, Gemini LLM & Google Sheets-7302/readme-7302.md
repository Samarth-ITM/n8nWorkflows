Extract & Structure Invoice Data with Google Vision OCR, Gemini LLM & Google Sheets

https://n8nworkflows.xyz/workflows/extract---structure-invoice-data-with-google-vision-ocr--gemini-llm---google-sheets-7302


# Extract & Structure Invoice Data with Google Vision OCR, Gemini LLM & Google Sheets

### 1. Workflow Overview

This workflow automates the extraction and structuring of invoice data from uploaded image files using Google Vision OCR and a Gemini LLM model, before saving the processed data in Google Sheets and notifying via Telegram. It is designed for accounting, finance, or administrative teams seeking to digitize invoice processing workflows, eliminating manual data entry and enabling real-time monitoring.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Storage**: Capturing invoice images via a user form, uploading them to Google Drive, and downloading the stored files for processing.
- **1.2 OCR Processing**: Converting invoice images to text through Google Vision API.
- **1.3 AI Text Structuring**: Using a Gemini LLM to parse OCR text into a structured JSON format with key invoice details.
- **1.4 Data Cleaning & Formatting**: Normalizing numeric fields and ensuring consistent data types.
- **1.5 Data Persistence & Notification**: Appending or updating structured data in Google Sheets and sending a transaction summary notification to Telegram.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Storage

**Overview:**  
This block handles the initial user input of invoice images, uploading the files to a Google Drive folder, and then downloading the uploaded file for subsequent processing.

**Nodes Involved:**  
- On form submission  
- Upload file  
- Download file  
- Code (Base64 conversion)  

**Node Details:**

- **On form submission**  
  - *Type & Role:* Form Trigger node; initiates workflow on user upload.  
  - *Config:* Listens for submissions on a form titled "Upload Purchase Invoice" with a required file field accepting JPG, PNG, JPEG formats.  
  - *Expressions:* None.  
  - *Connections:* Output → Upload file.  
  - *Edge Cases:* User uploads unsupported file types; no file submitted; form submission errors.  

- **Upload file**  
  - *Type & Role:* Google Drive node; uploads invoice image to specified Drive folder.  
  - *Config:* Uses `My Drive` drive setting, uploads to folder ID set by user; file name derived from form input filename.  
  - *Expressions:* Filename taken from form field (`Image[0].filename`).  
  - *Connections:* Input from form submission; output → Download file.  
  - *Edge Cases:* Drive quota exceeded, invalid folder ID, OAuth token expiration.  

- **Download file**  
  - *Type & Role:* Google Drive node; downloads the uploaded file binary data using file ID from upload.  
  - *Config:* Operation: download by file ID (`{{$json.id}}`).  
  - *Connections:* Input from Upload file; output → Code (Base64 conversion).  
  - *Edge Cases:* File not found, permission denied, API rate limits.  

- **Code (Base64 conversion)**  
  - *Type & Role:* Code node; converts binary file data into Base64 string for Vision API.  
  - *Config:* Runs once per item; extracts binary data and encodes to Base64.  
  - *Code:* Uses `item.binary.data.data` field.  
  - *Connections:* Input from Download file; output → Set Vision API.  
  - *Edge Cases:* Missing binary data, encoding errors.  

---

#### 1.2 OCR Processing

**Overview:**  
This block sends the Base64-encoded invoice image to Google Vision API for OCR text extraction.

**Nodes Involved:**  
- Set Vision API  
- HTTP Request  

**Node Details:**

- **Set Vision API**  
  - *Type & Role:* Set node; injects API key for Google Vision API into workflow JSON data.  
  - *Config:* Assigns string variable `visionAPI` with user’s API key placeholder.  
  - *Connections:* Input from Code; output → HTTP Request.  
  - *Edge Cases:* Missing or invalid API key.  

- **HTTP Request**  
  - *Type & Role:* HTTP Request node; calls Google Vision API's `images:annotate` endpoint with OCR request.  
  - *Config:*  
    - URL dynamically composed with API key: `https://vision.googleapis.com/v1/images:annotate?key={{ $json.visionAPI }}`  
    - Method: POST  
    - JSON body includes Base64 image content and `TEXT_DETECTION` feature.  
  - *Expressions:* Image content passed from `Code` node’s Base64 output.  
  - *Connections:* Input from Set Vision API; output → Basic LLM Chain.  
  - *Edge Cases:* Network errors, invalid API key, rate limiting, malformed JSON, large images causing timeouts.  

---

#### 1.3 AI Text Structuring

**Overview:**  
Processes OCR text output using Gemini LLM, enforcing structured JSON output with specific invoice fields.

**Nodes Involved:**  
- Basic LLM Chain  
- OpenRouter Chat Model (language model)  
- Structured Output Parser (output parser)  

**Node Details:**

- **Basic LLM Chain**  
  - *Type & Role:* Langchain LLM Chain node; main orchestrator for LLM text processing.  
  - *Config:*  
    - Input text: OCR extracted text from Vision API response (`{{$json.responses[0].fullTextAnnotation.text}}`).  
    - Message prompt instructs LLM to parse invoice fields: DATE (YYYY-MM-DD), NO VOUCHER, TRANSACTION DETAIL, VENDOR, VALUE.  
    - Output parser enabled to enforce structured JSON output.  
  - *Connections:* Input from HTTP Request; output → Code1 (data cleaning).  
  - *Edge Cases:* LLM API errors, incomplete/ambiguous OCR text, prompt misinterpretation, output parsing failures.  

- **OpenRouter Chat Model**  
  - *Type & Role:* Langchain language model node; configured with Gemini 2.0 Flash model.  
  - *Config:* Model `google/gemini-2.0-flash-exp:free`.  
  - *Connections:* Connected as AI language model node to Basic LLM Chain.  
  - *Edge Cases:* Model rate limits, API key issues, network errors.  

- **Structured Output Parser**  
  - *Type & Role:* Langchain output parser; enforces JSON schema on LLM output.  
  - *Config:* Example schema specifying fields DATE, NO VOUCHER, TRANSACTION DETAIL, VENDOR, VALUE with sample data.  
  - *Connections:* Connected as AI output parser to Basic LLM Chain.  
  - *Edge Cases:* Parsing errors if LLM output deviates from schema.  

---

#### 1.4 Data Cleaning & Formatting

**Overview:**  
Normalizes and cleans numeric values in parsed data to ensure proper numeric types without currency symbols or formatting.

**Nodes Involved:**  
- Code1  

**Node Details:**

- **Code1**  
  - *Type & Role:* Code node; cleans numbers by removing periods and commas, converting to integers.  
  - *Config:*  
    - Defines `cleanNumber` helper function.  
    - Cleans numeric fields in output JSON (Quantity, Unit Price, Total, Total Amount).  
    - Returns cleaned output JSON.  
  - *Connections:* Input from Basic LLM Chain; output → Append or update row in sheet.  
  - *Edge Cases:* Unexpected data structures, missing numeric fields, conversion failures.  

---

#### 1.5 Data Persistence & Notification

**Overview:**  
Writes the cleaned data into a Google Sheets document, either inserting a new row or updating an existing one by matching file ID. Sends a Telegram notification summarizing the transaction.

**Nodes Involved:**  
- Append or update row in sheet  
- Send a text message  

**Node Details:**

- **Append or update row in sheet**  
  - *Type & Role:* Google Sheets node; appends or updates invoice data in a spreadsheet.  
  - *Config:*  
    - Document ID and sheet name specified by user.  
    - Mapping of columns includes ID (file ID), IMG (preview link), DATE, VALUE, VENDOR, FILE NAME, NO VOUCHER, TRANSACTION DETAIL.  
    - Matches rows by "ID" column to prevent duplicates.  
  - *Connections:* Input from Code1; output → Send a text message.  
  - *Edge Cases:* Sheet access errors, invalid document ID, network errors, concurrency conflicts.  

- **Send a text message**  
  - *Type & Role:* Telegram node; sends formatted transaction summary to Telegram chat/group.  
  - *Config:*  
    - Text message includes date, transaction detail, vendor, and value.  
    - Chat ID placeholder requires user configuration.  
  - *Connections:* Input from Append or update row in sheet.  
  - *Edge Cases:* Invalid bot token, wrong chat ID, Telegram API limits.  

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                                  | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                                                            |
|-------------------------|---------------------------------|-------------------------------------------------|------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                    | Entry point; triggers workflow on invoice upload | —                      | Upload file                 | - This node triggers the workflow when a user submits a form titled "Upload Purchase Invoice".                                                         |
| Upload file             | Google Drive                   | Uploads uploaded invoice image to Google Drive   | On form submission     | Download file               | - Uploads the file received from the form into a specific Google Drive folder.                                                                         |
| Download file           | Google Drive                   | Downloads uploaded file from Drive                | Upload file            | Code                       | - Downloads the file from Google Drive using the file ID obtained from the previous Upload file node.                                                  |
| Code                    | Code                           | Converts downloaded file binary to Base64 string | Download file          | Set Vision API              | - Converts the downloaded invoice file (binary) into Base64 format so that it can be sent to the Google Vision API.                                   |
| Set Vision API          | Set                            | Injects Google Vision API key into workflow data | Code                   | HTTP Request                | - Adds the Google Vision API key into the workflow data so it can be used in the HTTP request.                                                        |
| HTTP Request            | HTTP Request                   | Sends OCR request to Google Vision API           | Set Vision API         | Basic LLM Chain             | - Sends the Base64-encoded image to the Google Vision API for OCR processing.                                                                          |
| Basic LLM Chain         | Langchain LLM Chain            | Parses OCR text with LLM into structured data     | HTTP Request           | Code1                      | - Processes OCR text through an LLM to extract and structure key invoice details. <br> - Forces the LLM to produce output in a fixed JSON schema. <br> - Cleans and normalizes numeric fields from the LLM output. |
| OpenRouter Chat Model   | Langchain Language Model       | Provides Gemini Flash LLM for text processing     | — (used by Basic LLM Chain) | Basic LLM Chain (AI language model) |                                                                                                                                                        |
| Structured Output Parser| Langchain Output Parser        | Enforces JSON schema on LLM output                | — (used by Basic LLM Chain) | Basic LLM Chain (AI output parser) |                                                                                                                                                        |
| Code1                   | Code                           | Cleans and formats numeric fields from LLM output| Basic LLM Chain        | Append or update row in sheet |                                                                                                                                                        |
| Append or update row in sheet | Google Sheets                | Inserts or updates invoice data in Google Sheets | Code1                  | Send a text message         | - Writes the extracted and cleaned invoice data into a Google Sheets document, either adding a new row or updating an existing one.                  |
| Send a text message     | Telegram                       | Sends summary notification to Telegram chat      | Append or update row in sheet | —                         | - Sends a Telegram message summarizing the new or updated transaction entry.                                                                           |
| Sticky Note1            | Sticky Note                    | Documentation block for overall workflow          | —                      | —                           | # Automated Invoice Collection & Data Extraction Using Vision API and LLM<br>Summary of purpose, features, credentials, and benefits.                |
| Sticky Note2            | Sticky Note                    | Documentation for input reception nodes           | —                      | —                           | - This node triggers the workflow when a user submits a form titled "Upload Purchase Invoice".<br>- Uploads the file received from the form into a specific Google Drive folder.<br>- Downloads the file from Google Drive using the file ID obtained from the previous Upload file node. |
| Sticky Note             | Sticky Note                    | Documentation for OCR processing nodes             | —                      | —                           | - Converts the downloaded invoice file (binary) into Base64 format so that it can be sent to the Google Vision API.<br>- Adds the Google Vision API key into the workflow data so it can be used in the HTTP request.<br>- Sends the Base64-encoded image to the Google Vision API for OCR processing. |
| Sticky Note3            | Sticky Note                    | Documentation for AI parsing nodes                 | —                      | —                           | - Processes OCR text through an LLM to extract and structure key invoice details.<br>- Forces the LLM to produce output in a fixed JSON schema.<br>- Cleans and normalizes numeric fields from the LLM output. |
| Sticky Note4            | Sticky Note                    | Documentation for data persistence and notification| —                      | —                           | - Writes the extracted and cleaned invoice data into a Google Sheets document, either adding a new row or updating an existing one.<br>- Sends a Telegram message summarizing the new or updated transaction entry. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission`  
   - Set form title: "Upload Purchase Invoice"  
   - Add a file upload field labeled "Image"  
   - Accept file types: `.jpg, .png, .jpeg`  
   - Configure response mode as "lastNode".

2. **Create a Google Drive node to upload files**  
   - Name: `Upload file`  
   - Operation: Upload  
   - Drive: Select "My Drive" or your preferred drive  
   - Folder ID: Enter your Google Drive folder ID for invoice storage  
   - File name: Use expression `{{$json.Image[0].filename}}`  
   - Input data field: `Image` (matches form upload)  
   - Connect `On form submission` output to this node.

3. **Create a Google Drive node to download files**  
   - Name: `Download file`  
   - Operation: Download by File ID  
   - File ID: Use expression `{{$json.id}}` from Upload file output  
   - Connect `Upload file` output to this node.

4. **Create a Code node to convert file to Base64**  
   - Name: `Code`  
   - Mode: Run once for each item  
   - JavaScript code:
     ```javascript
     const base64 = item.binary.data.data;
     return { json: { base64 } };
     ```
   - Connect `Download file` output to this node.

5. **Create a Set node to inject Vision API key**  
   - Name: `Set Vision API`  
   - Add a string field `visionAPI` with your Google Vision API key (replace placeholder).  
   - Connect `Code` output to this node.

6. **Create an HTTP Request node to call Google Vision API**  
   - Name: `HTTP Request`  
   - Method: POST  
   - URL: `https://vision.googleapis.com/v1/images:annotate?key={{ $json.visionAPI }}`  
   - Request Body Type: JSON  
   - JSON Body:
     ```json
     {
       "requests": [
         {
           "image": {
             "content": "{{ $json.base64 }}"
           },
           "features": [
             {
               "type": "TEXT_DETECTION"
             }
           ]
         }
       ]
     }
     ```
   - Connect `Set Vision API` output to this node.

7. **Create a Langchain LLM Chain node for parsing**  
   - Name: `Basic LLM Chain`  
   - Set input text expression: `{{$json.responses[0].fullTextAnnotation.text}}`  
   - Add a message prompt instructing the LLM to parse date, voucher number, transaction detail, vendor, and value, with fallback defaults.  
   - Enable output parser.

8. **Configure Langchain OpenRouter Chat Model node**  
   - Name: `OpenRouter Chat Model`  
   - Model: `google/gemini-2.0-flash-exp:free`  
   - Connect as AI language model node to `Basic LLM Chain`.

9. **Configure Langchain Structured Output Parser node**  
   - Name: `Structured Output Parser`  
   - Provide example JSON schema with keys: DATE, NO VOUCHER, TRANSACTION DETAIL, VENDOR, VALUE.  
   - Connect as AI output parser node to `Basic LLM Chain`.

10. **Create a Code node for data cleaning**  
    - Name: `Code1`  
    - JavaScript code to clean numeric fields (removing commas, periods, converting to integer).  
    - Connect `Basic LLM Chain` output to this node.

11. **Create a Google Sheets node to append or update data**  
    - Name: `Append or update row in sheet`  
    - Operation: Append or update  
    - Document ID: Your Google Sheets document ID  
    - Sheet Name: Use sheet by ID or name (e.g., `gid=0`)  
    - Map columns:  
      - ID: `{{$node["Download file"].json["id"]}}`  
      - IMG: `{{$node["Download file"].json["webViewLink"]}}`  
      - DATE, VALUE, VENDOR, FILE NAME, NO VOUCHER, TRANSACTION DETAIL: from cleaned LLM output (`$json.output`)  
    - Matching columns: `ID` to prevent duplicates  
    - Connect `Code1` output to this node.

12. **Create a Telegram node to send notification**  
    - Name: `Send a text message`  
    - Bot Token: Configure your Telegram Bot OAuth2 credentials  
    - Chat ID: Your Telegram chat or group ID  
    - Message text template:
      ```
      💳 New transaction : 
      - Date : {{ $json.DATE }}
      - Transaction detail : {{ $json['TRANSACTION DETAIL'] }}
      - Vendor : {{ $json.VENDOR }}
      - Total transaction : {{ $json.VALUE }}
      
      ---
      ```
    - Connect `Append or update row in sheet` output to this node.

13. **Connect nodes in the following order:**  
    On form submission → Upload file → Download file → Code (Base64) → Set Vision API → HTTP Request → Basic LLM Chain → Code1 → Append or update row in sheet → Send a text message.

14. **Credential Setup:**  
    - Google Vision API Key (for HTTP Request)  
    - OpenRouter API Key (for Gemini LLM)  
    - Google Drive OAuth2 (for Upload and Download Google Drive nodes)  
    - Google Sheets OAuth2 (for Sheets node)  
    - Telegram Bot Token (for Telegram node)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow template Google Sheets document is available here: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1HMzQtFK9T-GDxGFSD7ErW_QLlq-PvCvoFASiHGG2fGM/edit?gid=0#gid=0)                                         | Spreadsheet template for storing structured invoice data.                                                                                                                                                                          |
| This workflow enables full automation from invoice upload to data extraction, structuring, storage, and notification, saving considerable manual effort and enabling real-time financial monitoring.                                      | General workflow benefits.                                                                                                                                                                                                          |
| Ensure API keys and OAuth2 credentials are correctly set up and tested before running the workflow to avoid authentication errors.                                                                                                       | Credential setup recommendation.                                                                                                                                                                                                   |
| For detailed prompt customization or LLM output schema changes, adjust the "Basic LLM Chain" node message prompt and "Structured Output Parser" node accordingly.                                                                       | Customization tip.                                                                                                                                                                                                                   |
| Telegram chat ID can be obtained by messaging your bot or using Telegram API tools; verify it to ensure notifications reach the intended group or user.                                                                                 | Telegram bot configuration tip.                                                                                                                                                                                                    |

---

This structured documentation provides a clear, concise reference to understand, reproduce, and maintain the "Automated Invoice Collection & Data Extraction Using Vision API and LLM" workflow. It also anticipates common errors related to API keys, file handling, and data parsing to enable proactive troubleshooting.