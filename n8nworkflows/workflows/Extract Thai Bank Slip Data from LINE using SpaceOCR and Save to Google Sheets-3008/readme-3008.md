Extract Thai Bank Slip Data from LINE using SpaceOCR and Save to Google Sheets

https://n8nworkflows.xyz/workflows/extract-thai-bank-slip-data-from-line-using-spaceocr-and-save-to-google-sheets-3008


# Extract Thai Bank Slip Data from LINE using SpaceOCR and Save to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of transaction details from Thai bank slip images received via a LINE BOT and records the structured data into Google Sheets. It is designed for businesses, accountants, and finance teams who want to eliminate manual data entry and improve accuracy in financial record-keeping.

The workflow is logically divided into the following blocks:

- **1.1 LINE Webhook Trigger:** Receives incoming bank slip images sent by users through the LINE BOT.
- **1.2 Image Retrieval and Storage:** Downloads the image content from LINE, converts it to binary, and uploads it to Google Drive for accessible OCR processing.
- **1.3 OCR Processing:** Sends the uploaded image URL to the SpaceOCR API to extract text content from the bank slip.
- **1.4 Data Extraction and Parsing:** Processes the OCR text to parse key transaction details such as sender, receiver, amount, transaction ID, and fees.
- **1.5 Data Recording:** Appends the extracted transaction data into a predefined Google Sheets document for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 LINE Webhook Trigger

- **Overview:**  
  This block listens for incoming POST requests from the LINE BOT webhook, capturing bank slip image messages sent by users.

- **Nodes Involved:**  
  - Line Chat Bot  
  - Sticky Note4 (comment)

- **Node Details:**

  - **Line Chat Bot**  
    - Type: Webhook  
    - Role: Entry point that receives POST requests from LINE BOT with message payloads including image IDs.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `23ba996d-3242-42a1-946c-f04a680b320a` (unique webhook path)  
    - Inputs: External HTTP POST from LINE platform  
    - Outputs: JSON payload containing event data, including message ID for the image  
    - Edge Cases:  
      - Webhook URL must be correctly configured in LINE Developer Console.  
      - Payload format changes from LINE API could break parsing.  
      - Network or authentication errors on webhook reception.

  - **Sticky Note4**  
    - Content: "## LINE Webhook Trigger \n**(Receive Image)**"  
    - Purpose: Documentation for this block.

---

#### 2.2 Image Retrieval and Storage

- **Overview:**  
  This block constructs the image content URL from the LINE message ID, downloads the image as binary data, and uploads it to Google Drive for further processing.

- **Nodes Involved:**  
  - Image slip URL in Line  
  - Get image to Binary  
  - Upload image to Google Drive  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Image slip URL in Line**  
    - Type: Set  
    - Role: Constructs the direct URL to download the image content from LINE API using the message ID.  
    - Configuration:  
      - Sets a variable `file_url` with value:  
        `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`  
      - Uses expression to dynamically insert message ID from webhook payload.  
    - Inputs: JSON from webhook node  
    - Outputs: JSON with `file_url` string  
    - Edge Cases:  
      - If message ID is missing or malformed, URL will be invalid.  
      - LINE API rate limits or access token expiry could cause failures.

  - **Get image to Binary**  
    - Type: HTTP Request  
    - Role: Downloads the image content from the constructed URL as binary data.  
    - Configuration:  
      - URL: Uses expression `={{ $json.file_url }}`  
      - Authentication: HTTP Header Auth with credentials (likely LINE channel access token)  
    - Inputs: JSON with `file_url`  
    - Outputs: Binary data of the image  
    - Edge Cases:  
      - Authentication failure if token expired or invalid.  
      - Network timeouts or HTTP errors.  
      - Large image size causing timeouts.

  - **Upload image to Google Drive**  
    - Type: Google Drive  
    - Role: Uploads the binary image to a specified Google Drive folder for public access by OCR API.  
    - Configuration:  
      - File name: Uses message ID with `.jpg` extension  
      - Drive: "My Drive"  
      - Folder: Specific folder URL (configured via folderId)  
      - Credentials: Google Drive OAuth2 account  
    - Inputs: Binary image data  
    - Outputs: JSON with uploaded file metadata including file ID  
    - Edge Cases:  
      - Google Drive API quota limits or permission errors.  
      - Folder ID must be accessible by the OAuth2 account.  
      - Large files may cause upload failures.

  - **Sticky Note**  
    - Content:  
      ```
      ## Prepare data
      **- Get content image from Line** 
      https://api-data.line.me/v2/bot/message/xxx/content

      **- Get image URL to Binary**
      ```
    - Purpose: Documentation for this block.

  - **Sticky Note1**  
    - Content: "## Upload image to Google Drive"  
    - Purpose: Documentation for this block.

---

#### 2.3 OCR Processing

- **Overview:**  
  This block sends the Google Drive image URL to the SpaceOCR API to extract text from the bank slip image.

- **Nodes Involved:**  
  - Send Image URL to OCR Space for Text Extraction  
  - Sticky Note2

- **Node Details:**

  - **Send Image URL to OCR Space for Text Extraction**  
    - Type: HTTP Request  
    - Role: Calls SpaceOCR API with the image URL to perform OCR and return parsed text.  
    - Configuration:  
      - URL:  
        ```
        https://api.ocr.space/parse/imageurl?apikey=K82173083188957&language=tha&isOverlayRequired=false&OCREngine=2&filetype=JPG&url={{ "https://drive.google.com/uc?id=" + $json["id"] }}
        ```  
      - Method: GET (default)  
      - No authentication besides API key in URL  
    - Inputs: JSON containing Google Drive file ID from previous node  
    - Outputs: JSON with OCR parsed results  
    - Edge Cases:  
      - API key invalid or quota exceeded.  
      - OCR API downtime or slow response.  
      - Image URL must be publicly accessible; permission issues on Google Drive may cause failures.  
      - OCR errors due to poor image quality.

  - **Sticky Note2**  
    - Content:  
      ```
      ## OCR and get value
      **- OCR API by SpaceOCR**
      https://api.ocr.space/parse/imageurl?apikey=YOURAPI&language=tha&isOverlayRequired=false&OCREngine=2&filetype=JPG&url=xxx

      **- Parse Transaction Details**
      ```
    - Purpose: Documentation for this block.

---

#### 2.4 Data Extraction and Parsing

- **Overview:**  
  This block processes the OCR text output to extract structured transaction details such as transaction type, date/time, sender and receiver info, transaction ID, amount, and fees. It handles both standard bank slips and PromptPay transactions.

- **Nodes Involved:**  
  - Extract Transaction Details  
  - Sticky Note3

- **Node Details:**

  - **Extract Transaction Details**  
    - Type: Code (JavaScript)  
    - Role: Parses OCR text lines to extract key transaction fields using keyword searches and line offsets.  
    - Configuration:  
      - JavaScript code that:  
        - Splits OCR text by line breaks  
        - Searches for keywords like "นาย", "Prompt", "เลขที่รายการ:", "จำนวน", "ค่าธรรมเนียม"  
        - Differentiates between PromptPay and standard transactions  
        - Extracts sender name, bank, account; receiver name, bank, account; transaction ID; amount; fee  
        - Returns JSON object with extracted fields or error if essential data missing  
    - Inputs: OCR JSON response from SpaceOCR node  
    - Outputs: JSON with structured transaction data or error message  
    - Edge Cases:  
      - OCR inaccuracies causing missing or malformed data  
      - Unexpected slip formats or missing keywords  
      - Partial extraction leading to error output  
      - Logging included for debugging extracted lines  
    - Version-specific: Uses n8n Code node v2 syntax

  - **Sticky Note3**  
    - Content: "## Store Data in Google Sheets"  
    - Purpose: Documentation for next block but placed here for proximity.

---

#### 2.5 Data Recording

- **Overview:**  
  This block appends the extracted transaction data into a Google Sheets document, mapping each field to the corresponding column.

- **Nodes Involved:**  
  - Record in Google Sheets

- **Node Details:**

  - **Record in Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends a new row with transaction details into a specified Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Document: Google Sheets URL (configured)  
      - Sheet: gid=0 (first sheet)  
      - Columns mapped explicitly:  
        - Transaction Type  
        - Date & Time  
        - Sender Name  
        - Sender Account  
        - Receiver Name  
        - Receiver Bank  
        - Receiver Account  
        - Transaction ID  
        - Amount  
        - Fee  
      - Mapping mode: Define below (manual mapping)  
      - Credentials: Google Sheets OAuth2 account  
    - Inputs: JSON with extracted transaction data  
    - Outputs: JSON with append operation result  
    - Edge Cases:  
      - Google Sheets API quota or permission errors  
      - Data type mismatches (all mapped as strings)  
      - Missing fields (workflow logic tries to avoid this)  

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                          | Input Node(s)            | Output Node(s)                        | Sticky Note                                                                                     |
|-------------------------------------|---------------------|----------------------------------------|--------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| Line Chat Bot                       | Webhook             | Receives bank slip image from LINE BOT | External HTTP POST       | Image slip URL in Line              | ## LINE Webhook Trigger \n**(Receive Image)**                                                  |
| Image slip URL in Line              | Set                 | Constructs image download URL           | Line Chat Bot            | Get image to Binary                 | ## Prepare data\n**- Get content image from Line** \nhttps://api-data.line.me/v2/bot/message/xxx/content\n\n**- Get image URL to Binary** |
| Get image to Binary                 | HTTP Request        | Downloads image binary from LINE URL    | Image slip URL in Line   | Upload image to Google Drive        |                                                                                                |
| Upload image to Google Drive        | Google Drive        | Uploads image to Google Drive folder    | Get image to Binary      | Send Image URL to OCR Space for Text Extraction | ## Upload image to Google Drive                                                                |
| Send Image URL to OCR Space for Text Extraction | HTTP Request        | Sends image URL to SpaceOCR API         | Upload image to Google Drive | Extract Transaction Details         | ## OCR and get value\n**- OCR API by SpaceOCR**\nhttps://api.ocr.space/parse/imageurl?apikey=YOURAPI&language=tha&isOverlayRequired=false&OCREngine=2&filetype=JPG&url=xxx\n\n**- Parse Transaction Details** |
| Extract Transaction Details         | Code (JavaScript)   | Parses OCR text to extract transaction data | Send Image URL to OCR Space for Text Extraction | Record in Google Sheets             | ## Store Data in Google Sheets                                                                 |
| Record in Google Sheets             | Google Sheets       | Appends extracted data to Google Sheets | Extract Transaction Details | None                              |                                                                                                |
| Sticky Note                        | Sticky Note         | Documentation                          | None                     | None                              | ## Prepare data\n**- Get content image from Line** \nhttps://api-data.line.me/v2/bot/message/xxx/content\n\n**- Get image URL to Binary** |
| Sticky Note1                       | Sticky Note         | Documentation                          | None                     | None                              | ## Upload image to Google Drive                                                                |
| Sticky Note2                       | Sticky Note         | Documentation                          | None                     | None                              | ## OCR and get value\n**- OCR API by SpaceOCR**\nhttps://api.ocr.space/parse/imageurl?apikey=YOURAPI&language=tha&isOverlayRequired=false&OCREngine=2&filetype=JPG&url=xxx\n\n**- Parse Transaction Details** |
| Sticky Note3                       | Sticky Note         | Documentation                          | None                     | None                              | ## Store Data in Google Sheets                                                                 |
| Sticky Note4                       | Sticky Note         | Documentation                          | None                     | None                              | ## LINE Webhook Trigger \n**(Receive Image)**                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Line Chat Bot`  
   - HTTP Method: POST  
   - Path: Use a unique path (e.g., `23ba996d-3242-42a1-946c-f04a680b320a`)  
   - Purpose: Receive incoming LINE BOT messages containing bank slip image IDs.

2. **Create Set Node**  
   - Type: Set  
   - Name: `Image slip URL in Line`  
   - Add a string field `file_url` with value:  
     ```
     https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content
     ```  
   - Connect input from `Line Chat Bot`.

3. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: `Get image to Binary`  
   - URL: Use expression `={{ $json.file_url }}`  
   - Authentication: HTTP Header Auth (configure with LINE channel access token)  
   - Purpose: Download the image content as binary data.  
   - Connect input from `Image slip URL in Line`.

4. **Create Google Drive Node**  
   - Type: Google Drive  
   - Name: `Upload image to Google Drive`  
   - Operation: Upload file  
   - File Name: Use expression `={{ $('Line Chat Bot').item.json.body.events[0].message.id }}.jpg`  
   - Drive: Select "My Drive"  
   - Folder: Specify folder by URL or ID where images will be stored  
   - Credentials: Google Drive OAuth2 account with write access  
   - Connect input from `Get image to Binary`.

5. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: `Send Image URL to OCR Space for Text Extraction`  
   - URL: Use expression:  
     ```
     https://api.ocr.space/parse/imageurl?apikey=YOUR_API_KEY&language=tha&isOverlayRequired=false&OCREngine=2&filetype=JPG&url={{ "https://drive.google.com/uc?id=" + $json["id"] }}
     ```  
   - Replace `YOUR_API_KEY` with your SpaceOCR API key.  
   - Purpose: Send the Google Drive image URL to SpaceOCR API for OCR processing.  
   - Connect input from `Upload image to Google Drive`.

6. **Create Code Node**  
   - Type: Code (JavaScript)  
   - Name: `Extract Transaction Details`  
   - Paste the provided JavaScript code that:  
     - Parses OCR text lines  
     - Extracts transaction type, date/time, sender/receiver info, transaction ID, amount, fee  
     - Returns structured JSON or error if extraction fails  
   - Connect input from `Send Image URL to OCR Space for Text Extraction`.

7. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: `Record in Google Sheets`  
   - Operation: Append  
   - Document: Link to your Google Sheets document (must have columns for transaction data)  
   - Sheet: Select appropriate sheet (e.g., gid=0)  
   - Columns: Map fields from code node JSON output to columns:  
     - Transaction Type  
     - Date & Time  
     - Sender Name  
     - Sender Account  
     - Receiver Name  
     - Receiver Bank  
     - Receiver Account  
     - Transaction ID  
     - Amount  
     - Fee  
   - Credentials: Google Sheets OAuth2 account with write access  
   - Connect input from `Extract Transaction Details`.

8. **Connect all nodes sequentially:**  
   `Line Chat Bot` → `Image slip URL in Line` → `Get image to Binary` → `Upload image to Google Drive` → `Send Image URL to OCR Space for Text Extraction` → `Extract Transaction Details` → `Record in Google Sheets`.

9. **Credential Setup:**  
   - Configure HTTP Header Auth credentials for LINE API access token.  
   - Configure Google Drive OAuth2 credentials with permission to upload files.  
   - Configure Google Sheets OAuth2 credentials with permission to append rows.  
   - Obtain and configure SpaceOCR API key in the HTTP Request node URL.

10. **Deploy and Test:**  
    - Activate the workflow.  
    - Set the webhook URL in LINE Developer Console to the n8n webhook URL.  
    - Send a test bank slip image via LINE BOT.  
    - Verify image upload in Google Drive, OCR text extraction, and data appended in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Google Sheets template with required columns: Date, Time, Sender, Receiver, Bank Name, Amount, Transaction ID                   | https://docs.google.com/spreadsheets/d/1IpvzcnWmb-aLpSleTIF0xoF8xzbOOJQhuT6ITAeEQks/edit?usp=sharing         |
| SpaceOCR API documentation and signup for API key                                                                             | https://spaceocr.com/                                                                                        |
| LINE Messaging API documentation for setting up webhook and message content retrieval                                           | https://developers.line.biz/en/docs/messaging-api/                                                          |
| Important: Google Drive folder must have sharing permissions allowing public or API access for OCR API to fetch image           | Google Drive folder sharing settings                                                                         |
| Ensure n8n instance has internet access and proper credentials configured for external API calls (LINE, Google, SpaceOCR)      | n8n credentials management                                                                                   |
| Workflow designed to handle both Standard Bank Slips and PromptPay transaction slips with specific parsing logic               | See code node comments for details                                                                           |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and maintaining the "Extract Thai Bank Slip Data from LINE using SpaceOCR and Save to Google Sheets" workflow.