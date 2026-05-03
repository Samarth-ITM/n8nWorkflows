Track Expenses from Receipt Photos with Telegram & Google Sheets using OCR.space

https://n8nworkflows.xyz/workflows/track-expenses-from-receipt-photos-with-telegram---google-sheets-using-ocr-space-6686


# Track Expenses from Receipt Photos with Telegram & Google Sheets using OCR.space

---
### 1. Workflow Overview

This n8n workflow automates tracking of income and expenses sent via Telegram messages and receipt photos, storing the data into Google Sheets. It targets users who want quick and easy expense management either by typing simple commands or by sending receipt photos for OCR extraction.

The workflow logic is organized into the following main blocks:

- **1.1 Input Reception:** Receives Telegram messages (text or photo) via a bot webhook.
- **1.2 Text Message Processing:** Parses the message text to detect income or expense entries and appends them to Google Sheets.
- **1.3 Photo Processing and OCR:** Handles receipt photos, downloads them from Telegram, sends to OCR.space for text extraction, parses the OCR results, and adds expense data to Google Sheets.
- **1.4 Confirmation and Error Handling:** Sends success confirmations back to the Telegram user or error messages if input format is invalid.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives updates from the Telegram Bot and routes messages based on their content type (text or photo).

**Nodes Involved:**  
- Telegram Bot (Webhook)

**Node Details:**  
- **Telegram Bot (Webhook)**
  - Type: Telegram Bot node (Webhook)
  - Purpose: Polls Telegram API for new messages every 5 seconds using the bot token stored in environment variables.
  - Configuration:
    - Poll interval: 5 seconds
    - Allowed updates: Only “message”
    - Only new updates processed to avoid duplicates
  - Inputs: None (start node)
  - Outputs: Routes to three checks — 'Income', 'Expense' (Text), and 'Photo'
  - Edge Cases:  
    - Network or Telegram API downtime could delay or block message receipt.
    - Incorrect or expired bot token in environment variable causes auth errors.
  - Version: Compatible with n8n v1.x Telegram node API

---

#### 2.2 Text Message Processing

**Overview:**  
Processes incoming text messages to detect either income or expense entries based on keywords and message format, then appends the extracted data to Google Sheets.

**Nodes Involved:**  
- Check 'Income' (If node)  
- Check 'Expense' (Text) (If node)  
- Add Income to Google Sheet  
- Add Expense to Google Sheet (Text)  
- Send Income Confirmation  
- Send Expense Confirmation (Text)  
- Send Error Message (Text)

**Node Details:**  

- **Check 'Income'**  
  - Type: If node  
  - Checks if the text message contains the word "income" (case-sensitive substring match).  
  - Input: Telegram Bot (Webhook) output  
  - Outputs:  
    - True: Add Income to Google Sheet  
    - False: Send Error Message (Text) if no other conditions matched  
  - Edge Cases:  
    - Partial matches in message text may cause false positives.  
    - Case sensitivity might miss some valid inputs if user types 'Income' or 'INCOME'.

- **Check 'Expense' (Text)**  
  - Type: If node  
  - Checks if text message contains "expense".  
  - Input: Telegram Bot (Webhook) output  
  - Outputs:  
    - True: Add Expense to Google Sheet (Text)  
    - False: Send Error Message (Text) if no other conditions matched  
  - Edge Cases: Same as above for case sensitivity and substring matching.

- **Add Income to Google Sheet**  
  - Type: Google Sheets Append Row  
  - Adds a row in the "Income" sheet with columns: Date, Type (fixed "Income"), Amount, Source (fixed "Manual"), Description.  
  - Extracts amount and description by splitting the message text by spaces and using the 2nd and 3rd tokens respectively.  
  - Date defaults to current day ($today).  
  - Uses OAuth2 credentials for Google Sheets.  
  - Inputs: From Check 'Income' node (true output)  
  - Outputs: Send Income Confirmation  
  - Edge Cases:  
    - Message format errors can cause index out of bounds or incorrect data.  
    - Google Sheets API quota or auth errors possible.

- **Add Expense to Google Sheet (Text)**  
  - Type: Google Sheets Append Row  
  - Adds a row in "Expenses" sheet with columns: Date, Type ("Expense"), Amount, Source ("Manual"), Category, Description.  
  - Parses description, amount, category from message text split tokens (2nd, 3rd, and 4th tokens).  
  - Inputs: Check 'Expense' (Text) node (true output)  
  - Outputs: Send Expense Confirmation (Text)  
  - Edge Cases: Similar to Add Income node, with additional field for category. Message format must strictly follow: expense [description] [amount] [category].

- **Send Income Confirmation**  
  - Type: Telegram Bot (Send Message)  
  - Sends a confirmation message to the user confirming recorded income with formatted amount and description.  
  - Uses chat ID from incoming message.  
  - Inputs: Add Income to Google Sheet node output  
  - Outputs: None  
  - Edge Cases: Network or Telegram API errors.

- **Send Expense Confirmation (Text)**  
  - Type: Telegram Bot (Send Message)  
  - Sends confirmation for manual expense entry with description, amount, and category.  
  - Inputs: Add Expense to Google Sheet (Text) output  
  - Outputs: None  

- **Send Error Message (Text)**  
  - Type: Telegram Bot (Send Message)  
  - Sends error instructions when message format is unrecognized.  
  - Includes example correct formats and suggests sending photos for automatic tracking.  
  - Inputs: False outputs of Check 'Income', Check 'Expense' (Text), and alternative path from Check 'Photo' node.  
  - Outputs: None  

---

#### 2.3 Photo Processing and OCR

**Overview:**  
Handles receipt photos sent by user, downloads the photo file, submits it to OCR.space API for text extraction, parses the OCR results for expense data, and appends the expense to Google Sheets.

**Nodes Involved:**  
- Check 'Photo' (If node)  
- Get Telegram Photo  
- OCR.space Request (HTTP Request)  
- Parse OCR Data (Function)  
- Add Expense to Google Sheet (OCR)  
- Send Expense Confirmation (OCR)

**Node Details:**  

- **Check 'Photo'**  
  - Type: If node  
  - Checks if the incoming Telegram message contains a photo array (non-empty).  
  - Inputs: Telegram Bot (Webhook) output  
  - Outputs:  
    - True: Get Telegram Photo  
    - False: Send Error Message (Text) (via second output)  
  - Edge Cases:  
    - Messages with no photos or stickers might trigger false negatives.

- **Get Telegram Photo**  
  - Type: Telegram Bot (Get File)  
  - Downloads the largest photo from the array by selecting the last element (highest resolution).  
  - Uses file_id from photo metadata, chat ID to retrieve file.  
  - Inputs: Check 'Photo' (true output)  
  - Outputs: OCR.space Request  
  - Edge Cases:  
    - Telegram API errors if photo file expired or permissions invalid.

- **OCR.space Request**  
  - Type: HTTP Request (POST with multipart/form-data)  
  - Sends photo binary to OCR.space API with parameters: apikey (env var), language English, overlay required.  
  - Inputs: Binary data from Get Telegram Photo  
  - Outputs: Parse OCR Data  
  - Edge Cases:  
    - OCR API quota exceeded, invalid API key, network errors.

- **Parse OCR Data**  
  - Type: Function (JavaScript)  
  - Parses OCR response JSON to extract: description, amount, date, category.  
  - Uses regex to find amounts and dates in OCR text.  
  - Defaults to today’s date and “OCR Expense” description if no matches found.  
  - Category inferred by keyword matching (food, transport, shopping, utilities).  
  - Inputs: OCR.space Request JSON output  
  - Outputs: Data object with extracted fields for Google Sheets  
  - Edge Cases:  
    - OCR errors or low-quality images may lead to inaccurate extraction.  
    - Date parsing ambiguity (MM/DD/YYYY vs DD/MM/YYYY) simplified to MM/DD/YYYY, which may cause errors in some locales.

- **Add Expense to Google Sheet (OCR)**  
  - Type: Google Sheets Append Row  
  - Adds extracted expense data to "Expenses" sheet with source marked as "OCR".  
  - Inputs: Parse OCR Data output  
  - Outputs: Send Expense Confirmation (OCR)  
  - Edge Cases: Similar to manual entry nodes.

- **Send Expense Confirmation (OCR)**  
  - Type: Telegram Bot (Send Message)  
  - Sends confirmation with description, amount, and category extracted from receipt.  
  - Inputs: Add Expense to Google Sheet (OCR) output  
  - Outputs: None  

---

#### 2.4 Confirmation and Error Handling (Cross-cutting)

This block is distributed within previous blocks to send appropriate feedback messages.

**Nodes Involved:**  
- Send Income Confirmation  
- Send Expense Confirmation (Text)  
- Send Expense Confirmation (OCR)  
- Send Error Message (Text)

**Key Points:**  
- Confirms successful data additions to Google Sheets.  
- Provides user guidance for incorrect message formats or unsupported inputs.  
- Uses HTML parse mode in Telegram messages for better formatting.  
- Handles chatId dynamically from incoming messages to respond correctly.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                          | Input Node(s)                   | Output Node(s)                           | Sticky Note                                                                                                          |
|-------------------------------|-------------------------|----------------------------------------|--------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Telegram Bot (Webhook)         | Telegram Bot (Webhook)   | Receives Telegram messages via webhook | None                           | Check 'Income', Check 'Expense' (Text), Check 'Photo' |                                                                                                                      |
| Check 'Income'                 | If                      | Checks if message contains "income"   | Telegram Bot (Webhook)          | Add Income to Google Sheet, Send Error Message (Text) |                                                                                                                      |
| Check 'Expense' (Text)         | If                      | Checks if message contains "expense"  | Telegram Bot (Webhook)          | Add Expense to Google Sheet (Text), Send Error Message (Text) |                                                                                                                      |
| Add Income to Google Sheet     | Google Sheets           | Appends income data to "Income" sheet | Check 'Income' (true)           | Send Income Confirmation                |                                                                                                                      |
| Add Expense to Google Sheet (Text) | Google Sheets           | Appends expense data to "Expenses" sheet (manual text) | Check 'Expense' (Text) (true)   | Send Expense Confirmation (Text)        |                                                                                                                      |
| Send Income Confirmation       | Telegram Bot (Send Msg) | Sends confirmation for income entry   | Add Income to Google Sheet      | None                                    |                                                                                                                      |
| Send Expense Confirmation (Text) | Telegram Bot (Send Msg) | Sends confirmation for manual expense | Add Expense to Google Sheet (Text) | None                                    |                                                                                                                      |
| Send Error Message (Text)      | Telegram Bot (Send Msg) | Sends error message for unrecognized input | Check 'Income' (false), Check 'Expense' (Text) (false), Check 'Photo' (false) | None                                    |                                                                                                                      |
| Check 'Photo'                  | If                      | Checks if message contains photo       | Telegram Bot (Webhook)          | Get Telegram Photo, Send Error Message (Text) |                                                                                                                      |
| Get Telegram Photo             | Telegram Bot (Get File) | Downloads photo file from Telegram     | Check 'Photo' (true)            | OCR.space Request                       |                                                                                                                      |
| OCR.space Request              | HTTP Request            | Sends photo to OCR.space API for text extraction | Get Telegram Photo              | Parse OCR Data                         |                                                                                                                      |
| Parse OCR Data                | Function                | Parses OCR response to extract expense data | OCR.space Request              | Add Expense to Google Sheet (OCR)       |                                                                                                                      |
| Add Expense to Google Sheet (OCR) | Google Sheets           | Appends OCR-extracted expense to sheet | Parse OCR Data                  | Send Expense Confirmation (OCR)         |                                                                                                                      |
| Send Expense Confirmation (OCR) | Telegram Bot (Send Msg) | Sends confirmation for OCR expense     | Add Expense to Google Sheet (OCR) | None                                    |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot (Webhook) Node**  
   - Type: Telegram Bot  
   - Operation: getUpdates (Webhook)  
   - Set `webhookUrl` to: `https://api.telegram.org/bot{{$env.YOUR_TELEGRAM_BOT_TOKEN}}/getUpdates`  
   - Poll interval: 5 seconds  
   - Allowed updates: message  
   - Enable only new updates  
   - No input connections

2. **Create If Node "Check 'Income'"**  
   - Condition: Text contains "income" (case sensitive)  
   - Input: connect from Telegram Bot (Webhook) output (index 0)  

3. **Create If Node "Check 'Expense' (Text)"**  
   - Condition: Text contains "expense"  
   - Input: connect from Telegram Bot (Webhook) output (index 0)  

4. **Create If Node "Check 'Photo'"**  
   - Condition: message.photo is not empty (check non-empty array)  
   - Input: connect from Telegram Bot (Webhook) output (index 0)  

5. **Create Google Sheets Node "Add Income to Google Sheet"**  
   - Operation: Append Row  
   - Sheet Name: Income  
   - Columns mapping:  
     - Date: `{{$today}}`  
     - Type: "Income" (static)  
     - Amount: `{{$json.message.text.split(' ')[2]}}`  
     - Source: "Manual" (static)  
     - Description: `{{$json.message.text.split(' ')[1]}}`  
   - Spreadsheet ID: `{{$env.YOUR_GOOGLE_SHEET_ID}}`  
   - Authentication: OAuth2  
   - Connect input from Check 'Income' (true output)  

6. **Create Google Sheets Node "Add Expense to Google Sheet (Text)"**  
   - Operation: Append Row  
   - Sheet Name: Expenses  
   - Columns mapping:  
     - Date: `{{$today}}`  
     - Type: "Expense"  
     - Amount: `{{$json.message.text.split(' ')[2]}}`  
     - Source: "Manual"  
     - Category: `{{$json.message.text.split(' ')[3]}}`  
     - Description: `{{$json.message.text.split(' ')[1]}}`  
   - Spreadsheet ID: `{{$env.YOUR_GOOGLE_SHEET_ID}}`  
   - Authentication: OAuth2  
   - Connect input from Check 'Expense' (Text) (true output)  

7. **Create Google Sheets Node "Add Expense to Google Sheet (OCR)"**  
   - Operation: Append Row  
   - Sheet Name: Expenses  
   - Columns mapping:  
     - Date: `{{$json.date}}`  
     - Type: "Expense"  
     - Amount: `{{$json.amount}}`  
     - Source: "OCR"  
     - Category: `{{$json.category}}`  
     - Description: `{{$json.description}}`  
   - Spreadsheet ID: `{{$env.YOUR_GOOGLE_SHEET_ID}}`  
   - Authentication: OAuth2  
   - Connect input from Parse OCR Data node output  

8. **Create Telegram Bot Node "Send Income Confirmation"**  
   - Operation: Send Message  
   - Text: `Income "{{$json.message.text.split(' ')[1]}}" of ${{parseFloat($json.message.text.split(' ')[2]).toLocaleString('en-US')}} successfully recorded!`  
   - Chat ID: `{{$json.message.chat.id}}`  
   - Parse mode: HTML  
   - Connect input from Add Income to Google Sheet output  

9. **Create Telegram Bot Node "Send Expense Confirmation (Text)"**  
   - Operation: Send Message  
   - Text: `Expense "{{$json.message.text.split(' ')[1]}}" of ${{parseFloat($json.message.text.split(' ')[2]).toLocaleString('en-US')}} for category "{{$json.message.text.split(' ')[3]}}" successfully recorded!`  
   - Chat ID: `{{$json.message.chat.id}}`  
   - Parse mode: HTML  
   - Connect input from Add Expense to Google Sheet (Text) output  

10. **Create Telegram Bot Node "Send Expense Confirmation (OCR)"**  
    - Operation: Send Message  
    - Text:  
      ```
      Expense from receipt successfully recorded:
      Description: *{{$json.description}}*
      Amount: *${{parseFloat($json.amount).toLocaleString('en-US')}}*
      Category: *{{$json.category}}*
      ```  
    - Chat ID: `{{$json.message.chat.id}}`  
    - Parse mode: HTML  
    - Connect input from Add Expense to Google Sheet (OCR) output  

11. **Create Telegram Bot Node "Send Error Message (Text)"**  
    - Operation: Send Message  
    - Text:  
      ```
      Sorry, message format not recognized. Use format:
      *income [description] [amount]*
      or
      *expense [description] [amount] [category]*
      Alternatively, send a photo of your receipt for automatic tracking!
      ```  
    - Chat ID: `{{$json.message.chat.id}}`  
    - Parse mode: HTML  
    - Connect inputs from:  
      - Check 'Income' (false output)  
      - Check 'Expense' (Text) (false output)  
      - Check 'Photo' (false output)  

12. **Create Telegram Bot Node "Get Telegram Photo"**  
    - Operation: Get File  
    - Chat ID: `{{$json.message.chat.id}}`  
    - File ID: `{{$json.message.photo[{$json.message.photo.length - 1}].file_id}}` (last/highest quality photo)  
    - Connect input from Check 'Photo' (true output)  

13. **Create HTTP Request Node "OCR.space Request"**  
    - Method: POST  
    - URL: `https://api.ocr.space/parse/image`  
    - Content Type: multipart/form-data  
    - Query parameters:  
      - apikey: `{{$env.YOUR_OCR_SPACE_API_KEY}}`  
      - language: "eng"  
      - isOverlayRequired: "true"  
    - Body parameter:  
      - file: `{{$binary.data}}` (binary photo from previous node)  
    - Connect input from Get Telegram Photo output  

14. **Create Function Node "Parse OCR Data"**  
    - Paste provided JavaScript function (parses OCR text, extracts date, amount, category, description)  
    - Connect input from OCR.space Request output  

15. **Connect Outputs Appropriately:**  
    - From Telegram Bot (Webhook) to Check 'Income', Check 'Expense' (Text), Check 'Photo'  
    - From Check 'Income' (true) to Add Income to Google Sheet  
    - From Check 'Income' (false) to Send Error Message (Text)  
    - From Check 'Expense' (Text) (true) to Add Expense to Google Sheet (Text)  
    - From Check 'Expense' (Text) (false) to Send Error Message (Text)  
    - From Check 'Photo' (true) to Get Telegram Photo  
    - From Check 'Photo' (false) to Send Error Message (Text)  
    - From Add Income to Google Sheet to Send Income Confirmation  
    - From Add Expense to Google Sheet (Text) to Send Expense Confirmation (Text)  
    - From Get Telegram Photo to OCR.space Request  
    - From OCR.space Request to Parse OCR Data  
    - From Parse OCR Data to Add Expense to Google Sheet (OCR)  
    - From Add Expense to Google Sheet (OCR) to Send Expense Confirmation (OCR)

16. **Set Environment Variables:**  
    - YOUR_TELEGRAM_BOT_TOKEN: Your Telegram bot token  
    - YOUR_GOOGLE_SHEET_ID: Your Google Sheets spreadsheet ID  
    - YOUR_OCR_SPACE_API_KEY: Your OCR.space API key  

17. **Set up OAuth2 Credentials in n8n for Google Sheets:**  
    - Use OAuth2 authentication for Google Sheets nodes  
    - Ensure scopes include editing spreadsheets  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Use the format `income [description] [amount]` or `expense [description] [amount] [category]` in Telegram text messages for manual entries. | Workflow usage instructions                                                                        |
| Photo receipts sent via Telegram are automatically processed through OCR.space to extract expense details.    | Key feature enabling automatic data entry                                                         |
| Ensure environment variables for tokens and API keys are securely stored and correctly set in n8n.            | Security best practice                                                                              |
| For more info on n8n Telegram Bot node: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.telegrambot/ | Official n8n documentation                                                                          |
| OCR.space API documentation: https://ocr.space/ocrapi | OCR service used for receipt photo text extraction                                               |
| Google Sheets API and OAuth2 setup guidance: https://developers.google.com/sheets/api/quickstart/js          | Required for Google Sheets integration                                                             |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.