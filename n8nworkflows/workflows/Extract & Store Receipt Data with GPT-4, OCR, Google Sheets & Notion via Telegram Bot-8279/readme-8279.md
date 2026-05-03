Extract & Store Receipt Data with GPT-4, OCR, Google Sheets & Notion via Telegram Bot

https://n8nworkflows.xyz/workflows/extract---store-receipt-data-with-gpt-4--ocr--google-sheets---notion-via-telegram-bot-8279


# Extract & Store Receipt Data with GPT-4, OCR, Google Sheets & Notion via Telegram Bot

### 1. Workflow Overview

This workflow automates the extraction, validation, and multi-platform storage of receipt data received via a Telegram bot. It supports both image-based receipts (processed through OCR and AI extraction) and manual text entries. The extracted transaction data is validated and then stored across multiple systems including Google Sheets, Google Drive (for receipt images), Notion database, and a custom website API. Users receive confirmation or error messages directly via Telegram.

Logical blocks:

- **1.1 Input Reception & Routing:** Receives user input from Telegram (images or text) and routes processing accordingly.
- **1.2 OCR Processing (Image Input):** Downloads receipt images from Telegram and performs OCR to extract raw text.
- **1.3 Manual Text Handling:** Parses manual text inputs for receipt data.
- **1.4 AI Data Extraction:** Uses GPT-4 to extract structured transaction data from raw or OCR-processed text.
- **1.5 Data Validation:** Validates and cleans AI-extracted data, generates transaction IDs, and prepares data for storage.
- **1.6 Error Handling:** Detects validation errors and communicates them back to the user.
- **1.7 Data Storage & Confirmation:** Stores validated data and receipt images in Google Sheets, Google Drive, Notion, and a website API. Sends success confirmation to users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Routing

- **Overview:**  
Receives Telegram messages, distinguishes between photo and text inputs, and routes to appropriate processing nodes.

- **Nodes Involved:**  
Telegram Bot Trigger, Message Type Router

- **Node Details:**

1. **Telegram Bot Trigger**  
   - Type: Trigger node for Telegram messages  
   - Configuration: Listens for "message" updates on the configured Telegram bot webhook.  
   - Inputs: Telegram user messages  
   - Outputs: Raw Telegram message JSON  
   - Credentials: Telegram Bot API credentials  
   - Edge cases: Telegram API downtime, webhook misconfiguration, unsupported message types

2. **Message Type Router**  
   - Type: Switch node  
   - Configuration: Routes based on message content presence of `photo` or `text`.  
     - Route 1: If message contains a photo → Download Telegram Image  
     - Route 2: If message contains text → Handle Text Message  
   - Inputs: Output from Telegram Bot Trigger  
   - Outputs: Two branches to image or text processing  
   - Edge cases: Messages without photo or text; possible empty payloads

#### 1.2 OCR Processing (Image Input)

- **Overview:**  
Downloads the receipt image from Telegram, sends it to OCR.space API to extract text, then passes the OCR text to the AI extractor.

- **Nodes Involved:**  
Download Telegram Image, OCR Receipt Processing

- **Node Details:**

1. **Download Telegram Image**  
   - Type: Telegram node (file download)  
   - Configuration: Downloads the highest resolution image from the Telegram message photo array using its file_id.  
   - Credentials: Telegram Bot API  
   - Inputs: Photo file_id from Message Type Router  
   - Outputs: JSON with file path to downloaded Telegram image  
   - Edge cases: Invalid or expired file_id, Telegram API limits

2. **OCR Receipt Processing**  
   - Type: HTTP Request node  
   - Configuration: POST request to `https://api.ocr.space/parse/image` with parameters:  
     - URL of the downloaded image file from Telegram  
     - Language: English ("eng")  
     - Detect Orientation: true  
     - Is Table: true (to detect tables in receipts)  
   - Headers include API key from OCR.space credentials  
   - Inputs: Image file path from Download Telegram Image  
   - Outputs: OCR API JSON response with parsed text  
   - Credentials: OCR.space API key  
   - Edge cases: OCR API rate limits, malformed images, network timeouts

#### 1.3 Manual Text Handling

- **Overview:**  
Handles messages that are plain text, parsing a simple format into a pseudo-structured text to feed into AI extraction.

- **Nodes Involved:**  
Handle Text Message

- **Node Details:**

1. **Handle Text Message**  
   - Type: Function node  
   - Configuration: Parses text messages expecting format like `"Vendor $amount description"`  
   - Extracts vendor name (first word), amount (regex for $number), sets current date as default, and a fixed type "purchase".  
   - Throws error if format is invalid.  
   - Inputs: Text message from Message Type Router  
   - Outputs: JSON with a `ParsedResults` array containing a `ParsedText` string formatted to mimic OCR output  
   - Edge cases: Unexpected text formats, missing or invalid amounts, error thrown to be caught downstream

#### 1.4 AI Data Extraction

- **Overview:**  
Uses GPT-4 to extract structured transaction fields from raw OCR or parsed text.

- **Nodes Involved:**  
AI Purchase Data Extractor

- **Node Details:**

1. **AI Purchase Data Extractor**  
   - Type: Langchain OpenAI node  
   - Configuration:  
     - Model: GPT-4  
     - Temperature: 0.1 (low randomness for consistency)  
     - Max tokens: 500  
     - Prompt: Asks GPT-4 to parse input text and return ONLY a JSON object with fields: vendor, amount (numeric), currency (USD), date (YYYY-MM-DD), transaction_type, category.  
     - Input text is from either OCR ParsedText or manual text.  
   - Credentials: OpenAI API key  
   - Inputs: Parsed OCR text or manual text  
   - Outputs: JSON object with extracted transaction data (as string or JSON)  
   - Edge cases: GPT API errors, malformed or unexpected AI responses, rate limits, JSON parsing failures

#### 1.5 Data Validation

- **Overview:**  
Validates AI-extracted data, ensures required fields exist and are valid, standardizes data, generates unique transaction IDs, and prepares data for downstream storage.

- **Nodes Involved:**  
Transaction Data Validator

- **Node Details:**

1. **Transaction Data Validator**  
   - Type: Function node  
   - Configuration:  
     - Attempts to parse AI response JSON (handles string or object).  
     - Checks required fields: vendor, amount, date, transaction_type. Throws error if missing.  
     - Generates unique transaction ID (`txn_` + random + timestamp).  
     - Cleans and standardizes fields, e.g., trims vendor, parses amount as float, defaults currency to USD, sets processed_date to now.  
     - Validates amount is positive number.  
     - Validates date format YYYY-MM-DD, otherwise sets to current date.  
     - Adds user info from Telegram message for traceability.  
   - Inputs: AI Purchase Data Extractor output  
   - Outputs: Validated transaction JSON or error JSON with original data and user info  
   - Edge cases: JSON parse errors, missing or invalid fields, invalid amounts, invalid dates

#### 1.6 Error Handling

- **Overview:**  
Detects validation errors and sends user-friendly error messages back to the user via Telegram.

- **Nodes Involved:**  
Check for Errors, Send Error Message

- **Node Details:**

1. **Check for Errors**  
   - Type: Switch node  
   - Configuration: Checks if `error` boolean field is true in the input JSON. Routes error messages for sending.  
   - Inputs: Output from Transaction Data Validator  
   - Outputs: Error branch or success branch  
   - Edge cases: Unexpected payload structure

2. **Send Error Message**  
   - Type: Telegram node  
   - Configuration: Sends error message text to user Telegram chat, including instructions for retry and manual input format.  
   - Inputs: Error data from Check for Errors  
   - Outputs: Continues to downstream nodes for logging and storage  
   - Credentials: Telegram Bot API  
   - Edge cases: Telegram API failures, invalid chat ID

#### 1.7 Data Storage & Confirmation

- **Overview:**  
Stores validated transaction data and receipt images to multiple platforms (Google Sheets, Google Drive, Notion, Website API), then sends confirmation message to user.

- **Nodes Involved:**  
Record to Database (Google Sheets), Store Receipt Image (Google Drive), Save to Notion Database, Send to Website API, Send Confirmation

- **Node Details:**

1. **Record to Database**  
   - Type: Google Sheets node  
   - Configuration: Appends a new row to the "Transactions" sheet of the specified Google Sheet document ID.  
   - Credentials: Google Sheets OAuth2  
   - Inputs: Validated transaction data  
   - Outputs: Confirmation path  
   - Edge cases: Google Sheets API quota, document access issues

2. **Store Receipt Image**  
   - Type: Google Drive node  
   - Configuration: Uploads receipt image with filename pattern `receipt_{id}_{date}.jpg` to Google Drive root or specified folder.  
   - Credentials: Google Drive OAuth2  
   - Inputs: Transaction data with image file reference  
   - Outputs: Confirmation path  
   - Edge cases: Google Drive quota, upload failures

3. **Save to Notion Database**  
   - Type: Notion node  
   - Configuration: Creates a page in a configured Notion database with transaction info.  
   - Credentials: Notion API  
   - Inputs: Validated transaction data  
   - Outputs: Confirmation path  
   - Edge cases: Notion API rate limits, permission issues

4. **Send to Website API**  
   - Type: HTTP Request node  
   - Configuration: POSTs transaction JSON to a custom website API endpoint with Authorization header.  
   - Inputs: Validated transaction data  
   - Outputs: Confirmation path  
   - Edge cases: API downtime, authentication failure, invalid payload

5. **Send Confirmation**  
   - Type: Telegram node  
   - Configuration: Sends a detailed success message to the user with transaction summary and confirmation of multi-platform storage.  
   - Credentials: Telegram Bot API  
   - Inputs: Success data from any storage node  
   - Outputs: None (end of flow)  
   - Edge cases: Telegram API failures, invalid chat ID

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                             | Input Node(s)               | Output Node(s)                                   | Sticky Note                                                                                                                                                                                                                 |
|-------------------------|---------------------------------|--------------------------------------------|-----------------------------|-------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Bot Trigger     | Telegram Trigger                 | Receive Telegram user messages              | -                           | Message Type Router                              |                                                                                                                                                                                                                             |
| Message Type Router      | Switch                          | Route by message content type (photo/text) | Telegram Bot Trigger         | Download Telegram Image, Handle Text Message      |                                                                                                                                                                                                                             |
| Download Telegram Image  | Telegram file download           | Download receipt image from Telegram        | Message Type Router          | OCR Receipt Processing                           |                                                                                                                                                                                                                             |
| OCR Receipt Processing   | HTTP Request                    | OCR API call to extract text from image     | Download Telegram Image      | AI Purchase Data Extractor                        |                                                                                                                                                                                                                             |
| Handle Text Message      | Function                        | Parse manual text input to structured text  | Message Type Router          | AI Purchase Data Extractor                        |                                                                                                                                                                                                                             |
| AI Purchase Data Extractor | Langchain OpenAI (GPT-4)        | Extract structured transaction data from text | OCR Receipt Processing, Handle Text Message | Transaction Data Validator                        |                                                                                                                                                                                                                             |
| Transaction Data Validator | Function                      | Validate and clean AI extracted data         | AI Purchase Data Extractor   | Check for Errors                                 |                                                                                                                                                                                                                             |
| Check for Errors         | Switch                         | Detect validation errors                      | Transaction Data Validator   | Send Error Message (on error), Record to Database (on success) |                                                                                                                                                                                                                             |
| Send Error Message       | Telegram                        | Send error message to user                    | Check for Errors             | Save to Notion Database, Send to Website API, Record to Database, Store Receipt Image |                                                                                                                                                                                                                             |
| Record to Database       | Google Sheets                  | Append transaction data to Google Sheet      | Check for Errors, Send Error Message | Send Confirmation                                |                                                                                                                                                                                                                             |
| Store Receipt Image      | Google Drive                   | Upload receipt image                          | Send Error Message           | Send Confirmation                                |                                                                                                                                                                                                                             |
| Save to Notion Database  | Notion                         | Save transaction as Notion page               | Send Error Message           | Send Confirmation                                |                                                                                                                                                                                                                             |
| Send to Website API      | HTTP Request                   | Post transaction data to custom API           | Send Error Message           | Send Confirmation                                |                                                                                                                                                                                                                             |
| Send Confirmation       | Telegram                        | Send success confirmation message             | Record to Database, Store Receipt Image, Save to Notion Database, Send to Website API | -                                               |                                                                                                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Trigger node:**  
   - Type: Telegram Trigger  
   - Configure webhook for Telegram bot updates (message type)  
   - Set credentials with your Telegram Bot API token

2. **Create Message Type Router node:**  
   - Type: Switch  
   - Condition 1: `$json.message.photo` exists → route to image flow  
   - Condition 2: `$json.message.text` exists → route to text flow  
   - Connect Telegram Bot Trigger output to this node input

3. **Image Processing Branch:**  
   a. Create **Download Telegram Image** node:  
      - Type: Telegram  
      - Set to download file with fileId from last photo in `$json.message.photo` array  
      - Use Telegram credentials  
      - Connect Message Type Router (photo branch) to this node  
  
   b. Create **OCR Receipt Processing** node:  
      - Type: HTTP Request  
      - Method: POST  
      - URL: `https://api.ocr.space/parse/image`  
      - Body parameters:  
        - `url`: `https://api.telegram.org/file/bot{{ $credentials.telegramApi.accessToken }}/{{ $('Download Telegram Image').item.json.file_path }}`  
        - `language`: "eng"  
        - `detectOrientation`: "true"  
        - `isTable`: "true"  
      - Headers: `apikey` from OCR.space API credentials  
      - Connect Download Telegram Image to this node

4. **Text Processing Branch:**  
   a. Create **Handle Text Message** node:  
      - Type: Function  
      - Code to parse text message expecting `"Vendor $amount description"` format, outputting formatted ParsedText  
      - Connect Message Type Router (text branch) to this node

5. **AI Extraction:**  
   - Create **AI Purchase Data Extractor** node:  
     - Type: Langchain OpenAI (GPT-4)  
     - Model: GPT-4  
     - Temperature: 0.1  
     - Max tokens: 500  
     - Prompt: Extract transaction data as JSON with fields: vendor, amount (numeric), currency (USD), date (YYYY-MM-DD), transaction_type, category  
     - Input text: Use `ParsedResults[0].ParsedText` from OCR or output from Handle Text Message  
     - Credentials: OpenAI API key  
   - Connect both OCR Receipt Processing and Handle Text Message outputs to this node

6. **Data Validation:**  
   - Create **Transaction Data Validator** node (Function):  
     - Parse AI JSON response, validate required fields: vendor, amount, date, transaction_type  
     - Generate unique transaction ID  
     - Clean and standardize fields  
     - Validate amount (>0) and date format (YYYY-MM-DD)  
     - Attach Telegram user ID and username  
     - Output error JSON if validation fails  
   - Connect AI Purchase Data Extractor to this node

7. **Error Checking:**  
   - Create **Check for Errors** node (Switch):  
     - Condition: if `$json.error` is true, route to error path  
     - Else route to success path  
   - Connect Transaction Data Validator to this node

8. **Error Path:**  
   a. Create **Send Error Message** node (Telegram):  
      - Send error message to user with instructions  
      - Use `chatId` from `$json.user_id`  
      - Connect Check for Errors error output to this node

9. **Success Path (and also after error message for logging):**  
   a. Create **Record to Database** node (Google Sheets):  
      - Operation: Append to sheet "Transactions"  
      - Document ID: your Google Sheet ID  
      - Credentials: Google Sheets OAuth2  
      - Connect Check for Errors success output and Send Error Message output to this node  
  
   b. Create **Store Receipt Image** node (Google Drive):  
      - File name: `receipt_{{ $json.id }}_{{ $json.date }}.jpg`  
      - Folder: root or specified folder  
      - Credentials: Google Drive OAuth2  
      - Connect Send Error Message output to this node  
  
   c. Create **Save to Notion Database** node:  
      - Resource: databasePage  
      - Database ID: your Notion database ID  
      - Credentials: Notion API  
      - Connect Send Error Message output to this node  
  
   d. Create **Send to Website API** node:  
      - HTTP Method: POST  
      - URL: your website API endpoint  
      - Headers: Content-Type application/json, Authorization Bearer token  
      - Body: transaction JSON  
      - Connect Send Error Message output to this node

10. **Send Confirmation:**  
    - Create **Send Confirmation** node (Telegram):  
      - Message: Confirmation with transaction details (vendor, amount, date, etc.)  
      - Chat ID: from `$json.user_id`  
      - Credentials: Telegram Bot API  
      - Connect outputs of Record to Database, Store Receipt Image, Save to Notion Database, Send to Website API to this node

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses OCR.space API for OCR processing of receipt images. Requires obtaining an API key from https://ocr.space/        | OCR Receipt Processing node                                                                             |
| GPT-4 model in OpenAI node requires API key and appropriate access to GPT-4 via OpenAI platform                                   | AI Purchase Data Extractor node                                                                         |
| Google Sheets, Google Drive, and Notion nodes require OAuth2 credentials configured with appropriate scopes and access permissions | Storage nodes                                                                                           |
| Telegram Bot credentials require bot token and webhook setup in Telegram BotFather and n8n                                         | Telegram Trigger and Telegram nodes                                                                     |
| Custom API endpoint requires a valid URL and API key/token for authentication                                                    | Send to Website API node                                                                                |
| Text input format for manual entries is `"Vendor $amount description"` (e.g., `McDonald's $12.50 lunch food`)                      | Handle Text Message node                                                                                 |
| Error messages instruct users on proper text input format and to send clearer images if OCR fails                                 | Send Error Message node                                                                                  |

---

**Disclaimer:**  
The provided content originates exclusively from an automated n8n workflow. It fully complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.