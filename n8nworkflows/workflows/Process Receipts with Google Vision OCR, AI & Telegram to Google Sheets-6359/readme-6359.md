Process Receipts with Google Vision OCR, AI & Telegram to Google Sheets

https://n8nworkflows.xyz/workflows/process-receipts-with-google-vision-ocr--ai---telegram-to-google-sheets-6359


# Process Receipts with Google Vision OCR, AI & Telegram to Google Sheets

### 1. Workflow Overview

This workflow automates financial transaction recording by processing receipt images sent via Telegram. It leverages Google Vision OCR to extract text from images, uses AI (OpenRouter LLM) to parse and structure the financial data, and stores the results in Google Sheets. Additionally, it enables users to query stored financial data via Telegram, answered by an AI assistant referencing the Google Sheets data.

Logical blocks include:

- **1.1 Input Reception:** Receiving images or text messages from users via Telegram.
- **1.2 Image Download & OCR Processing:** Downloading receipt images from Telegram, converting them to base64, and extracting text using Google Vision API.
- **1.3 AI Parsing & Structuring:** Using OpenRouter LLM to parse OCR text into structured financial data.
- **1.4 Data Cleaning & Sheet Updating:** Cleaning numeric fields, splitting item arrays, and appending/updating Google Sheets rows.
- **1.5 User Feedback:** Sending processing status and summaries to users on Telegram.
- **1.6 AI Query Agent:** Responding to text queries from users by searching Google Sheets data and answering via AI.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures incoming Telegram messages (photo or text). Routes photo messages for OCR processing and text messages for AI query handling.

**Nodes Involved:**  
- Telegram Trigger  
- If  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for any Telegram updates specifically message updates.  
  - Config: Listens to "message" updates only, uses Telegram bot API credential.  
  - Input: Incoming Telegram webhook calls.  
  - Output: Emits Telegram message JSON.  
  - Edge Cases: Invalid token, webhook misconfiguration, Telegram API downtime.  
- **If**  
  - Type: If  
  - Role: Conditional routing based on presence of photo in message.  
  - Config: Checks if `message.photo` exists (boolean exists operator).  
  - Input: Output of Telegram Trigger.  
  - Output:  
    - True branch: message contains photo → go to OCR processing.  
    - False branch: message is text → go to AI query agent.  
  - Edge Cases: Message without photo or text; malformed message structure.

---

#### 2.2 Image Download & OCR Processing

**Overview:**  
Fetches the file path of the image from Telegram, downloads it, converts to base64, and sends it to Google Vision API for OCR text extraction.

**Nodes Involved:**  
- Set Telegram Token  
- Get Path  
- Get URL  
- Code  
- Set Vision API  
- HTTP Request  
- Send a text message2  

**Node Details:**  

- **Set Telegram Token**  
  - Type: Set  
  - Role: Stores Telegram bot API token string in variable `telegramToken` for reuse.  
  - Config: Hardcoded string with placeholder `YOUR_TELEGRAM_BOT_TOKEN`.  
  - Input: True branch of If node.  
  - Output: Passes data with `telegramToken`.  
  - Edge Cases: Token missing or invalid token.

- **Get Path**  
  - Type: HTTP Request  
  - Role: Calls Telegram's `getFile` API to retrieve file path for the last photo in the message.  
  - Config: URL dynamically constructed with `telegramToken` and Telegram photo file_id from message.  
  - Input: Output of Set Telegram Token.  
  - Output: JSON containing `file_path`.  
  - Edge Cases: Invalid file_id, expired file link, Telegram API errors.

- **Get URL**  
  - Type: HTTP Request  
  - Role: Downloads the actual image file using the file path from previous node.  
  - Config: Uses Telegram file URL with `telegramToken` and file_path from Get Path.  
  - Input: Output of Get Path.  
  - Output: Binary image data.  
  - Edge Cases: Download failure, file not found, network errors.

- **Code**  
  - Type: Code  
  - Role: Converts binary image data to base64 string for OCR API.  
  - Config: Extracts `item.binary.data.data` and returns it as `json.base64`.  
  - Input: Output of Get URL.  
  - Output: JSON with base64 image string.  
  - Edge Cases: Binary data missing, conversion errors.

- **Set Vision API**  
  - Type: Set  
  - Role: Stores Google Vision API key in variable `visionAPI`.  
  - Config: Hardcoded placeholder `YOUR_GOOGLE_VISION_API_KEY`.  
  - Input: Output from Code node.  
  - Output: Adds `visionAPI` key to JSON.  
  - Edge Cases: Missing or invalid API key.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to Google Vision OCR endpoint with base64 image content.  
  - Config: URL uses `visionAPI` key; JSON body includes base64 image and requests TEXT_DETECTION feature.  
  - Input: Output of Set Vision API.  
  - Output: JSON response with OCR result including `fullTextAnnotation`.  
  - Edge Cases: API limit exceeded, invalid base64, timeout, malformed response.

- **Send a text message2**  
  - Type: Telegram  
  - Role: Sends immediate feedback to user that data is being processed.  
  - Config: Fixed text “Data is being processed . . .” sent to user chat ID.  
  - Input: Parallel output from Set Vision API.  
  - Output: Telegram message sent confirmation.  
  - Edge Cases: Telegram API errors, invalid chat ID.

---

#### 2.3 AI Parsing & Structuring

**Overview:**  
Processes raw OCR text with LLM to structure financial receipt data including date, vendor, items, and totals.

**Nodes Involved:**  
- Basic LLM Chain  
- OpenRouter Chat Model  
- Structured Output Parser  

**Node Details:**  

- **Basic LLM Chain**  
  - Type: Langchain LLM Chain  
  - Role: Sends OCR text to LLM with prompt instructing to extract financial fields.  
  - Config: Uses `responses[0].fullTextAnnotation` from Google Vision output as text input; prompt defines extraction instructions including missing data handling.  
  - Input: Output from HTTP Request (Google Vision OCR).  
  - Output: LLM raw output with structured data as string.  
  - Edge Cases: LLM API failures, incomplete OCR text, prompt misinterpretation.

- **OpenRouter Chat Model**  
  - Type: Langchain LM Chat (OpenRouter)  
  - Role: Provides underlying OpenRouter LLM model for Basic LLM Chain.  
  - Config: Model set to `google/gemini-2.0-flash-exp:free`; requires OpenRouter API key credential.  
  - Input: Connected as AI language model to Basic LLM Chain.  
  - Output: LLM responses.  
  - Edge Cases: API key invalid, rate limits, network issues.

- **Structured Output Parser**  
  - Type: Langchain Output Parser (Structured)  
  - Role: Parses LLM output into defined JSON schema with fields like Date, Vendor, Items[].  
  - Config: JSON schema example includes date, invoice number, items array with product, quantity, unit price, total.  
  - Input: Output from Basic LLM Chain.  
  - Output: Parsed JSON structured data.  
  - Edge Cases: Parsing failures due to malformed LLM output.

---

#### 2.4 Data Cleaning & Sheet Updating

**Overview:**  
Cleans numeric fields, splits item arrays, and appends or updates rows in Google Sheets accordingly.

**Nodes Involved:**  
- Code1  
- Split Out  
- Append or update row in sheet  

**Node Details:**  

- **Code1**  
  - Type: Code  
  - Role: Cleans numeric fields (removes commas/periods) and converts to integers for Quantity, Unit Price, Total.  
  - Config: Defines `cleanNumber` function; maps over Items array to clean and convert fields.  
  - Input: Output from Structured Output Parser (via Basic LLM Chain).  
  - Output: JSON output with cleaned Items array.  
  - Edge Cases: Non-numeric strings, missing fields, conversion errors.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits Items array into individual items to process each row separately.  
  - Config: Field to split out is `output.Items`.  
  - Input: Output of Code1.  
  - Output: Multiple items, one per product line.  
  - Edge Cases: Empty Items array, malformed Items.

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Role: Inserts or updates a row per item in the specified Google Sheet.  
  - Config:  
    - Document ID: Google Sheet template or user sheet.  
    - Sheet Name: gid=0 (first sheet).  
    - Matching column: 'Invoice Number' to update existing rows.  
    - Columns mapped: Date, Total, Vendor, Product, Category, Quantity, Unit Price, Invoice Number, Vendor Address from parsed data.  
  - Input: Each split item from Split Out.  
  - Output: Confirmation of sheet update.  
  - Edge Cases: Google Sheets API errors, permission denied, invalid sheet ID, duplicate invoice number conflicts.

---

#### 2.5 User Feedback

**Overview:**  
Sends a neatly formatted summary text message back to the Telegram user showing the recorded transaction details.

**Nodes Involved:**  
- Basic LLM Chain1  
- OpenRouter Chat Model1  
- Send a text message  

**Node Details:**  

- **Basic LLM Chain1**  
  - Type: Langchain LLM Chain  
  - Role: Formats the structured transaction data into human-readable text with emojis.  
  - Config: Takes parsed `output` JSON, prompt requests plain text summary with all items and total amount.  
  - Input: Output from Basic LLM Chain (structured data).  
  - Output: Summary text string.  
  - Edge Cases: Formatting errors, empty data.

- **OpenRouter Chat Model1**  
  - Type: Langchain LM Chat (OpenRouter)  
  - Role: Provides LLM model for Basic LLM Chain1.  
  - Config: Same model and credential as previous OpenRouter nodes.  
  - Input: AI language model connection from Basic LLM Chain1.  
  - Output: Formatted text.  
  - Edge Cases: API failures.

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends the summary message to the user.  
  - Config: Uses `text` field from Basic LLM Chain1 output; chatId from Telegram Trigger message sender.  
  - Input: Output from Basic LLM Chain1.  
  - Output: Telegram message confirmation.  
  - Edge Cases: Telegram API errors.

---

#### 2.6 AI Query Agent

**Overview:**  
Handles text messages (non-photo) by querying Google Sheets data and responding with AI-generated answers based on stored financial records.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model2  
- Simple Memory  
- Get row(s) in sheet in Google Sheets  
- Send a text message1  

**Node Details:**  

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Acts as conversational financial assistant; answers user queries referencing Google Sheets data.  
  - Config:  
    - Uses user message text as prompt.  
    - System message sets persona as professional accountant, conversational style with emojis, plain text.  
    - Connected to memory (Simple Memory), tool (Google Sheets), and LLM (OpenRouter Chat Model2).  
    - Output parser enabled.  
  - Input: False branch of If node (text messages).  
  - Output: AI-generated answer text.  
  - Edge Cases: API limits, memory overflow, Google Sheets query failures.

- **OpenRouter Chat Model2**  
  - Type: Langchain LM Chat (OpenRouter)  
  - Role: Provides LLM to AI Agent.  
  - Config: Same model and credential.  
  - Input: AI language model for AI Agent.  
  - Output: AI responses.  
  - Edge Cases: API key issues.

- **Simple Memory**  
  - Type: Memory Buffer Window  
  - Role: Maintains conversational context per user session keyed by Telegram user ID.  
  - Config: Context window length 10 messages.  
  - Input: From Telegram Trigger user ID.  
  - Output: Memory context to AI Agent.  
  - Edge Cases: Memory size limits, session mismatch.

- **Get row(s) in sheet in Google Sheets**  
  - Type: Google Sheets  
  - Role: Tool for AI Agent to query transaction data from Google Sheets.  
  - Config: Reads from same document and sheet as used for data appending.  
  - Input: AI Agent tool calls.  
  - Output: Sheet row data for AI Agent.  
  - Edge Cases: API errors, permission issues.

- **Send a text message1**  
  - Type: Telegram  
  - Role: Sends AI Agent's response back to user.  
  - Config: Uses AI Agent output text and Telegram chat ID.  
  - Input: AI Agent output.  
  - Output: Telegram message confirmation.  
  - Edge Cases: Telegram failures.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                    | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                                     |
|-----------------------------|----------------------------------|--------------------------------------------------|--------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                  | Entry point, receives Telegram messages          | -                              | If                           |                                                                                                                |
| If                         | If                               | Routes photo messages to OCR, text messages to AI| Telegram Trigger               | Set Telegram Token / AI Agent |                                                                                                                |
| Set Telegram Token          | Set                              | Stores Telegram API token                         | If (photo branch)              | Get Path                     | Store the Telegram bot API token as a variable (telegram Token) so that it can be used by subsequent nodes without repeated hardcoding. |
| Get Path                   | HTTP Request                     | Retrieves Telegram file path for photo           | Set Telegram Token             | Get URL                      | Retrieves the file path (image) from Telegram by accessing the getFile endpoint. This path is needed so we can download the image file from the Telegram server. After obtaining the file_path, this node accesses the direct URL to download the image file from the Telegram server. Next, it retrieves the binary image data from the previous node and converts it to a base64 string, which is required by the Google Vision API for the OCR process. |
| Get URL                    | HTTP Request                     | Downloads image file from Telegram server        | Get Path                      | Code                         |                                                                                                                |
| Code                       | Code                             | Converts binary image to base64 string           | Get URL                       | Set Vision API               |                                                                                                                |
| Set Vision API             | Set                              | Stores Google Vision API key                      | Code                          | HTTP Request, Send a text message2 | Save the Google Vision API key into a variable named visionAPI. This variable will later be used to dynamically call the OCR endpoint. Send a POST request to the Google Vision API endpoint. The sent data contains the image in base64 format from the previous node (Code) and uses TEXT_DETECTION as the OCR feature. Send a message to the Telegram user as a notification that the data is being processed. This provides immediate feedback so the user knows the workflow is running. |
| HTTP Request              | HTTP Request                     | Calls Google Vision OCR API                       | Set Vision API                | Basic LLM Chain              |                                                                                                                |
| Basic LLM Chain            | Langchain LLM Chain              | Extracts and structures text from OCR output    | HTTP Request                  | Basic LLM Chain1             | Using LLM (Language Model) to process OCR text from Google Vision API (fullTextAnnotation) and convert it into structured data. |
| OpenRouter Chat Model      | Langchain LM Chat (OpenRouter)  | Provides LLM model for Basic LLM Chain           | -                            | Basic LLM Chain (ai_languageModel) |                                                                                                                |
| Structured Output Parser   | Langchain Output Parser (Structured) | Parses LLM output into structured JSON            | Basic LLM Chain               | Basic LLM Chain              |                                                                                                                |
| Basic LLM Chain1           | Langchain LLM Chain              | Formats structured data into user-friendly text | Basic LLM Chain              | Send a text message          | Converts parsed transaction data into a human readable text summary. Sends the summary results from LLM to Telegram users. |
| OpenRouter Chat Model1     | Langchain LM Chat (OpenRouter)  | Provides LLM for Basic LLM Chain1                 | -                            | Basic LLM Chain1 (ai_languageModel) |                                                                                                                |
| Send a text message        | Telegram                        | Sends formatted summary to user                   | Basic LLM Chain1             | -                            |                                                                                                                |
| Code1                      | Code                             | Cleans numeric data and converts to integers     | Basic LLM Chain              | Split Out                   | Cleans numeric data from characters such as periods or commas (common in receipt pricing formats), then converts them to whole numbers (integers). Splits the Items[] array (containing products per row) into separate items so they can be recorded individually in rows in Google Sheets. Adds product data one by one to Google Sheets based on a predefined structure. If there are any duplicate Invoice Numbers, the data will be updated. |
| Split Out                  | Split Out                       | Splits Items array for per-item processing       | Code1                         | Append or update row in sheet|                                                                                                                |
| Append or update row in sheet | Google Sheets                  | Inserts or updates transaction rows in sheet     | Split Out                    | -                            |                                                                                                                |
| Send a text message2       | Telegram                        | Sends "Data is being processed..." message       | Set Vision API               | -                            |                                                                                                                |
| AI Agent                   | Langchain Agent                 | Conversational AI assistant for user queries     | If (text branch)             | Send a text message1         | Providing text based interaction features with users on Telegram, this agent is designed as a financial assistant that answers user questions based on transaction data previously recorded in Google Sheets. |
| OpenRouter Chat Model2     | Langchain LM Chat (OpenRouter)  | Provides LLM for AI Agent                          | -                            | AI Agent (ai_languageModel)  |                                                                                                                |
| Simple Memory              | Langchain Memory Buffer Window  | Stores chat context per user session              | -                            | AI Agent (ai_memory)         |                                                                                                                |
| Get row(s) in sheet in Google Sheets | Google Sheets             | Retrieves rows for AI Agent queries               | -                            | AI Agent (ai_tool)           |                                                                                                                |
| Send a text message1       | Telegram                        | Sends AI Agent's reply to user                     | AI Agent                     | -                            |                                                                                                                |
| Sticky Note                | Sticky Note                    | Contains detailed workflow overview and setup    | -                            | -                            | See detailed workflow description, setup instructions, and Google Sheets template link.                        |
| Sticky Note1               | Sticky Note                    | Explains storing Telegram token in variable       | -                            | -                            | Store the Telegram bot API token as a variable (telegram Token) so that it can be used by subsequent nodes without repeated hardcoding. |
| Sticky Note2               | Sticky Note                    | Explains image download and base64 conversion     | -                            | -                            | Retrieves the file path (image) from Telegram by accessing the getFile endpoint. This path is needed so we can download the image file from the Telegram server. After obtaining the file_path, this node accesses the direct URL to download the image file from the Telegram server. Next, it retrieves the binary image data from the previous node and converts it to a base64 string, which is required by the Google Vision API for the OCR process. |
| Sticky Note3               | Sticky Note                    | Explains Google Vision API key setting and OCR call | -                          | -                            | Save the Google Vision API key into a variable named visionAPI. This variable will later be used to dynamically call the OCR endpoint. Send a POST request to the Google Vision API endpoint. The sent data contains the image in base64 format from the previous node (Code) and uses TEXT_DETECTION as the OCR feature. Send a message to the Telegram user as a notification that the data is being processed. This provides immediate feedback so the user knows the workflow is running. |
| Sticky Note4               | Sticky Note                    | Explains LLM processing of OCR text                | -                            | -                            | Using LLM (Language Model) to process OCR text from Google Vision API (fullTextAnnotation) and convert it into structured data. |
| Sticky Note5               | Sticky Note                    | Explains data cleaning, splitting, and sheet updating | -                          | -                            | Cleans numeric data from characters such as periods or commas (common in receipt pricing formats), then converts them to whole numbers (integers). Splits the Items[] array (containing products per row) into separate items so they can be recorded individually in rows in Google Sheets. Adds product data one by one to Google Sheets based on a predefined structure. If there are any duplicate Invoice Numbers, the data will be updated. |
| Sticky Note6               | Sticky Note                    | Explains formatting and sending summary to user   | -                            | -                            | Converts parsed transaction data into a human readable text summary. Sends the summary results from LLM to Telegram users. |
| Sticky Note7               | Sticky Note                    | Explains AI Agent functionality                     | -                            | -                            | Providing text based interaction features with users on Telegram, this agent is designed as a financial assistant that answers user questions based on transaction data previously recorded in Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen to `message` updates.  
   - Credentials: Connect with Telegram Bot API token.

2. **Add an If Node**  
   - Condition: Check if `{{$json.message.photo}}` exists (boolean exists).  
   - True branch for photo messages; False branch for text messages.

3. **True Branch (Photo Processing):**  

   a. **Set Node "Set Telegram Token"**  
      - Set `telegramToken` string variable with your Telegram Bot Token.  

   b. **HTTP Request Node "Get Path"**  
      - Method: GET  
      - URL: `https://api.telegram.org/bot{{ $json.telegramToken }}/getFile?file_id={{ $('Telegram Trigger').item.json.message.photo.slice(-1)[0].file_id }}`  

   c. **HTTP Request Node "Get URL"**  
      - Method: GET  
      - URL: `https://api.telegram.org/file/bot{{ $('Set Telegram Token').item.json.telegramToken }}/{{ $json.result.file_path }}`  
      - Response Format: Binary  

   d. **Code Node "Code"**  
      - JavaScript: Extract base64 string from binary data:  
        ```js
        const base64 = item.binary.data.data;
        return { json: { base64 } };
        ```  

   e. **Set Node "Set Vision API"**  
      - Set `visionAPI` string variable with your Google Vision API Key.  

   f. **HTTP Request Node "HTTP Request"**  
      - Method: POST  
      - URL: `https://vision.googleapis.com/v1/images:annotate?key={{ $json.visionAPI }}`  
      - Body (JSON):  
        ```json
        {
          "requests": [
            {
              "image": { "content": "{{ $('Code').item.json.base64 }}" },
              "features": [{ "type": "TEXT_DETECTION" }]
            }
          ]
        }
        ```  

   g. **Telegram Node "Send a text message2"**  
      - Text: "Data is being processed . . ."  
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.from.id }}`  

4. **AI Parsing & Structuring:**  

   a. **Add OpenRouter Chat Model node**  
      - Model: `google/gemini-2.0-flash-exp:free`  
      - Credentials: Your OpenRouter API key.  

   b. **Basic LLM Chain Node**  
      - Input Text: `={{ $json.responses[0].fullTextAnnotation }}` from Vision API output  
      - Prompt: Instructions to extract date, vendor, invoice, items, etc.  
      - Connect OpenRouter Chat Model as AI language model.  

   c. **Structured Output Parser Node**  
      - Provide JSON schema example for expected structured output fields.  

5. **Data Cleaning & Sheet Update:**  

   a. **Code Node "Code1"**  
      - JS to clean Quantity, Unit Price, Total fields to integer by removing commas/periods.  

   b. **Split Out Node**  
      - Field to split: `output.Items`  

   c. **Google Sheets Node "Append or update row in sheet"**  
      - Operation: Append or update  
      - Document ID: Your Google Sheet document ID  
      - Sheet Name: gid=0 (first sheet)  
      - Matching Column: Invoice Number  
      - Columns mapped: Date, Total, Vendor, Product, Category, Quantity, Unit Price, Invoice Number, Vendor Address  

6. **Send Summary to User:**  

   a. **OpenRouter Chat Model1** (same model, API key)  

   b. **Basic LLM Chain1**  
      - Input: Structured output JSON  
      - Prompt: Format transaction details in easy-to-read text with emojis.  
      - Connect OpenRouter Chat Model1 as language model.  

   c. **Telegram Node "Send a text message"**  
      - Text: Output from Basic LLM Chain1  
      - Chat ID: User Telegram ID  

7. **False Branch (Text Queries):**  

   a. **Simple Memory Node**  
      - Session Key: `{{$json.message.from.id}}`  
      - Window Length: 10 messages  

   b. **OpenRouter Chat Model2**  
      - Same model and API key.  

   c. **Google Sheets Node "Get row(s) in sheet in Google Sheets"**  
      - Document ID and Sheet Name as above.  

   d. **AI Agent Node**  
      - Input: User text message  
      - System prompt: Professional accountant persona  
      - Connect: OpenRouter Chat Model2 (LM), Simple Memory, Google Sheets (tool)  
      - Enable output parser.  

   e. **Telegram Node "Send a text message1"**  
      - Text: AI Agent output  
      - Chat ID: User Telegram ID  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates recording financial transactions from receipt photos sent via Telegram, extracting data via OCR and AI, storing in Google Sheets, and supporting conversational queries.                                             | Workflow overview sticky note.                                                                                                                        |
| Google Sheets Template: [Financial Reporting](https://docs.google.com/spreadsheets/d/11oT95lKoGNFlZtV129ampaBNOxnnKAJnUn1fAlqYZvY/edit?usp=sharing)                                                                                   | Use this template or your own for data storage.                                                                                                      |
| Replace placeholders with your own API keys and tokens: Telegram Bot Token, Google Vision API key, OpenRouter API key, Google Sheets credentials.                                                                                        | Setup instructions sticky note.                                                                                                                      |
| Telegram bot token stored as variable for reusability and easy maintenance.                                                                                                                                                               | Sticky Note1                                                                                                                                        |
| Image download process: getFile API to get file path, download image file, convert to base64 for Google Vision OCR.                                                                                                                      | Sticky Note2                                                                                                                                        |
| Google Vision API key stored as variable; OCR request uses TEXT_DETECTION feature; sends user notification of ongoing processing.                                                                                                        | Sticky Note3                                                                                                                                        |
| LLM processes OCR text to extract structured financial data including date, vendor, invoice, items, and category.                                                                                                                       | Sticky Note4                                                                                                                                        |
| Cleans numeric formats (commas, dots), splits items into individual entries, appends or updates Google Sheets rows based on invoice number.                                                                                              | Sticky Note5                                                                                                                                        |
| Formats structured data into a clear summary with emojis, sends summary back to user via Telegram.                                                                                                                                       | Sticky Note6                                                                                                                                        |
| AI Agent provides conversational interface for users to query financial data stored in Google Sheets, maintaining context per user session.                                                                                             | Sticky Note7                                                                                                                                        |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.