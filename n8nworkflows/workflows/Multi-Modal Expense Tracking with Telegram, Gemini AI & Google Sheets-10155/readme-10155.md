Multi-Modal Expense Tracking with Telegram, Gemini AI & Google Sheets

https://n8nworkflows.xyz/workflows/multi-modal-expense-tracking-with-telegram--gemini-ai---google-sheets-10155


# Multi-Modal Expense Tracking with Telegram, Gemini AI & Google Sheets

---

## 1. Workflow Overview

This workflow implements a **Multi-Modal Expense Tracking System** using Telegram as the input channel, Google's Gemini AI for processing, and Google Sheets for data storage. It targets users who want to log expenses quickly by sending free-form messages to a Telegram bot, including text messages, voice notes, or images of receipts. The workflow automatically extracts structured expense data (`amount`, `item`, `date`) from these inputs and appends them as rows in a Google Sheet, then confirms the operation back to the user on Telegram.

The logic is organized into four main blocks:

- **1.1 Message Intake & Routing:** Listens for incoming Telegram messages and routes them based on content type (text, voice, image). Unsupported message types trigger an error reply.
- **1.2 Input Capture & Normalization:** Downloads and converts voice or image inputs into text using Gemini AI; text messages are passed as-is. Standardizes all inputs into a single text field.
- **1.3 Expense Parsing & Validation:** Uses a large language model (Gemini via LangChain agent) to parse the normalized text into a strict JSON array of expenses; applies auto-fixing and schema validation to ensure clean structured output.
- **1.4 Row Generation, Sheet Writing & Confirmation:** Splits parsed expenses into individual rows, writes them to Google Sheets, and sends a confirmation or empty notification to the user on Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Message Intake & Routing

**Overview:**  
This block triggers on new Telegram messages, detects the message type (text, voice, or image), and routes the flow to the appropriate processing branch. Unsupported message formats receive a friendly error message.

**Nodes Involved:**  
- Start: On Telegram Message  
- Route: Text, Voice or Image?  
- Telegram: Send error message

**Node Details:**

- **Start: On Telegram Message**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point triggered on any new message update from Telegram.  
  - *Config:* Listens to `message` updates; uses Telegram API credentials.  
  - *Input:* Incoming Telegram update payload.  
  - *Output:* Raw message JSON.  
  - *Edge cases:* Missing or malformed message data, webhook failure.  
  - *Notes:* Requires Telegram bot token credential.

- **Route: Text, Voice or Image?**  
  - *Type:* Switch node  
  - *Role:* Detects message type and routes accordingly: text, voice, image, or else (unsupported).  
  - *Config:*  
    - Routes:  
      - Text: if `message.text` exists.  
      - Voice: if `message.voice` exists.  
      - Image: if `message.photo` exists.  
      - Else: fallback for unsupported types.  
  - *Input:* Message JSON from trigger.  
  - *Output:* One of the four routes.  
  - *Edge cases:* Messages with multiple media types, missing expected fields, or unsupported types.  
  - *Failure:* If type detection logic fails, goes to fallback route.

- **Telegram: Send error message**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a textual error response for unsupported input types.  
  - *Config:* Fixed message text: "🤖Sorry, I can only understand text, voice and image for bookkeeping."  
  - *Input:* Routed from "else" output of router.  
  - *Output:* Telegram confirmation message sent.  
  - *Edge cases:* Telegram API failure, invalid chat ID.  
  - *Credential:* Telegram API with bot token.

---

### 2.2 Input Capture & Normalization

**Overview:**  
This block converts all supported input types into plain text for downstream parsing. Text messages are passed through; voice messages are downloaded and transcribed by Gemini AI; images are downloaded and analyzed for text extraction by Gemini AI. All paths produce a unified `raw_text` string.

**Nodes Involved:**  
- Set: Text Input  
- Telegram: Get Voice File  
- Gemini: Transcribe Voice  
- Set: Voice Input (Text)  
- Telegram: Get Image File  
- Gemini: Analyze an image  
- Set: image Input (Text)

**Node Details:**

- **Set: Text Input**  
  - *Type:* Set node  
  - *Role:* Assigns `raw_text` from Telegram text message directly.  
  - *Config:* `raw_text = {{ $json.message.text }}`  
  - *Input:* Text route from router.  
  - *Output:* JSON with `raw_text` string.  
  - *Edge cases:* Empty or malformed text messages.

- **Telegram: Get Voice File**  
  - *Type:* Telegram node (get file)  
  - *Role:* Downloads voice audio binary file using provided file ID.  
  - *Config:* Uses `fileId = {{ $json.message.voice.file_id }}`  
  - *Input:* Voice route from router.  
  - *Output:* Binary audio data.  
  - *Credential:* Telegram API.  
  - *Edge cases:* File not found, Telegram API limits, corrupt files.

- **Gemini: Transcribe Voice**  
  - *Type:* Google Gemini AI node (audio transcription)  
  - *Role:* Transcribes audio binary to text using Gemini AI model `models/gemini-2.5-flash`.  
  - *Config:* Audio resource input, binary property named `data`.  
  - *Input:* Binary audio from previous node.  
  - *Output:* Transcribed text in JSON `content.parts`.  
  - *Credential:* Google Gemini API credential.  
  - *Edge cases:* Transcription errors, API timeouts, unsupported audio formats.

- **Set: Voice Input (Text)**  
  - *Type:* Set node  
  - *Role:* Converts Gemini transcription output into unified `raw_text` string.  
  - *Config:* `raw_text = {{ $json.content.parts.toJsonString() }}`  
  - *Input:* Gemini transcription output.  
  - *Output:* JSON with `raw_text`.  
  - *Edge cases:* Empty transcription, malformed output.

- **Telegram: Get Image File**  
  - *Type:* Telegram node (get file)  
  - *Role:* Downloads the first image file from photo array.  
  - *Config:* `fileId = {{ $json.message.photo[0].file_id }}`  
  - *Input:* Image route from router.  
  - *Output:* Binary image data.  
  - *Credential:* Telegram API.  
  - *Edge cases:* No photo array, file retrieval errors.

- **Gemini: Analyze an image**  
  - *Type:* Google Gemini AI node (image analysis)  
  - *Role:* Extracts financial and transactional text from image binary data.  
  - *Config:*  
    - Text prompt instructing to extract all visible financial text/numbers.  
    - Model: `models/gemini-2.5-flash`.  
    - Input type: binary image.  
  - *Input:* Binary image from previous node.  
  - *Output:* Extracted text in JSON `content.parts`.  
  - *Credential:* Google Gemini API credential.  
  - *Edge cases:* Image too noisy, unsupported image formats, API failures.

- **Set: image Input (Text)**  
  - *Type:* Set node  
  - *Role:* Converts Gemini image analysis output into unified `raw_text` string.  
  - *Config:* `raw_text = {{ $json.content.parts.toJsonString() }}`  
  - *Input:* Gemini image analysis output.  
  - *Output:* JSON with `raw_text`.  
  - *Edge cases:* Empty or invalid extraction.

---

### 2.3 Expense Parsing & Validation

**Overview:**  
This block processes the unified `raw_text` input via an AI agent to parse free-form text into a JSON array of expense objects with fields `{ amount, item, date }`. It enforces strict schema rules, auto-fixes malformed JSON, and validates structured output.

**Nodes Involved:**  
- AI: Extract Expenses  
- Auto-Fix JSON  
- AI Helper: Define Schema  
- LLM: Gemini Flash

**Node Details:**

- **AI: Extract Expenses**  
  - *Type:* LangChain Agent node (AI agent)  
  - *Role:* Extracts expense entries from `raw_text` into structured JSON using Gemini AI.  
  - *Config:*  
    - Input: `text = information:{{ $json.raw_text }}`  
    - System message: Very detailed prompt enforcing strict JSON output with keys `amount` (number|null), `item` (string|null), `date` (ISO string), rules for parsing, multi-entry handling, currency normalization, date resolution including relative dates, and examples.  
    - Output parser enabled.  
  - *Input:* JSON containing `raw_text`.  
  - *Output:* JSON array of expense objects under key `output`.  
  - *Credential:* Gemini API.  
  - *Edge cases:* AI hallucination, malformed JSON responses, incomplete data, API failures.

- **Auto-Fix JSON**  
  - *Type:* Output parser autofixing node  
  - *Role:* Automatically corrects JSON output errors from AI if it deviates from schema or format.  
  - *Config:* Custom prompt to retry completion upon error.  
  - *Input:* AI raw completion output.  
  - *Output:* Corrected, clean JSON array.  
  - *Edge cases:* Persistent parsing errors, malformed AI responses.

- **AI Helper: Define Schema**  
  - *Type:* Output parser structured node  
  - *Role:* Enforces output schema with example JSON of expected structure.  
  - *Config:* Example schema with keys `amount`, `item`, `date`.  
  - *Input:* AI completion output.  
  - *Output:* Validated structured data.  
  - *Edge cases:* Schema mismatch, incomplete data.

- **LLM: Gemini Flash**  
  - *Type:* LangChain language model node  
  - *Role:* The underlying language model used for AI completions in parsing and autofixing.  
  - *Credential:* Gemini API.  
  - *Edge cases:* API limits, latency, model errors.

---

### 2.4 Row Generation, Sheet Write & Confirmation

**Overview:**  
This block splits the parsed JSON array into individual expense entries, maps fields for Google Sheets columns, appends the data into the configured Google Sheet, and sends a Telegram confirmation message. If no valid entries are found, it sends a null notification.

**Nodes Involved:**  
- Split: Entries to Rows  
- If null?  
- Set: Extract Output  
- GSheet: Log Expense  
- Telegram: Send Confirmation  
- Telegram: Send Null

**Node Details:**

- **Split: Entries to Rows**  
  - *Type:* SplitOut node  
  - *Role:* Splits the JSON array of expenses so each object is processed individually downstream.  
  - *Config:* Field to split: `output` (the JSON array key).  
  - *Input:* Output JSON array from AI extraction.  
  - *Output:* One item per expense object.  
  - *Edge cases:* Empty arrays, null entries.

- **If null?**  
  - *Type:* If node  
  - *Role:* Checks if the current split item is null or empty to decide which path to take.  
  - *Config:* Checks if `output` is truthy and does not contain `null`.  
  - *Input:* Single expense item.  
  - *Output:*  
    - True: valid expense object.  
    - False: null or empty entry.  
  - *Edge cases:* Unexpected data types, null fields.

- **Set: Extract Output**  
  - *Type:* Set node  
  - *Role:* Maps expense JSON keys to explicit named fields for Google Sheets: `Amount`, `Item`, `Date`.  
  - *Config:* Assignments:  
    - `Amount = {{ $json.output.amount }}`  
    - `Item = {{ $json.output.item }}`  
    - `Date = {{ $json.output.date }}`  
  - *Input:* Valid expense item.  
  - *Output:* Row data formatted for sheet.  
  - *Edge cases:* Missing or null fields.

- **GSheet: Log Expense**  
  - *Type:* Google Sheets node  
  - *Role:* Appends rows into a specific Google Sheet tab.  
  - *Config:*  
    - Operation: Append rows.  
    - Sheet name and document ID preconfigured to target "Log Expense" sheet.  
    - Columns mapped automatically from input data.  
    - Cell format: RAW.  
  - *Input:* Row data from Set node.  
  - *Output:* Confirmation of append operation.  
  - *Credential:* Google Sheets OAuth2.  
  - *Edge cases:* Authentication failure, API rate limits, sheet permission issues.

- **Telegram: Send Confirmation**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a MarkdownV2 formatted summary listing all saved expenses with `item — amount (date)` per line.  
  - *Config:*  
    - Chat ID must be set to user or group chat.  
    - Text dynamically generated from all items in the batch.  
    - Escapes Telegram MarkdownV2 special characters.  
  - *Input:* Successful completion of sheet append.  
  - *Output:* Confirmation message sent.  
  - *Credential:* Telegram API.  
  - *Edge cases:* Chat ID missing or invalid, message length limits.

- **Telegram: Send Null**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a simple message indicating no items were found or logged.  
  - *Config:* Fixed text: "📒 Bookkeeping Complete\n\nnull"  
  - *Input:* Triggered when no valid expense entries are found.  
  - *Output:* Notification message sent.  
  - *Credential:* Telegram API.  
  - *Edge cases:* Same as above.

---

## 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                                       | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                                                         |
|----------------------------|----------------------------------------|------------------------------------------------------|-----------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Start: On Telegram Message  | Telegram Trigger                       | Entry point, triggers on new Telegram messages       | —                           | Route: Text, Voice or Image?      | ## 1) Message Intake & Routing<br>Purpose: Receive user messages and route by input type.                                           |
| Route: Text, Voice or Image?| Switch                                | Routes messages by type (text, voice, image, else)   | Start: On Telegram Message   | Set: Text Input; Telegram: Get Voice File; Telegram: Get Image File; Telegram: Send error message | ## 1) Message Intake & Routing<br>Purpose: Receive user messages and route by input type.                                           |
| Telegram: Send error message| Telegram (send message)                | Sends error for unsupported message types             | Route: Text, Voice or Image? (else) | —                                | ## 1) Message Intake & Routing<br>Purpose: Receive user messages and route by input type.                                           |
| Set: Text Input             | Set                                   | Assigns raw_text from text message                     | Route: Text, Voice or Image? (Text) | AI: Extract Expenses             | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| Telegram: Get Voice File    | Telegram (get file)                    | Downloads voice audio file                             | Route: Text, Voice or Image? (Voice) | Gemini: Transcribe Voice          | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| Gemini: Transcribe Voice    | Google Gemini AI (audio transcription)| Transcribes voice audio to text                        | Telegram: Get Voice File      | Set: Voice Input (Text)           | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| Set: Voice Input (Text)     | Set                                   | Assigns raw_text from transcription output             | Gemini: Transcribe Voice      | AI: Extract Expenses              | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| Telegram: Get Image File    | Telegram (get file)                    | Downloads image file                                  | Route: Text, Voice or Image? (Image) | Gemini: Analyze an image          | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| Gemini: Analyze an image    | Google Gemini AI (image analysis)     | Extracts financial text from image                     | Telegram: Get Image File      | Set: image Input (Text)           | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| Set: image Input (Text)     | Set                                   | Assigns raw_text from image analysis output            | Gemini: Analyze an image      | AI: Extract Expenses              | ## 2) Input Capture & Normalization<br>Turn everything into plain text for parsing.                                                  |
| AI: Extract Expenses        | LangChain Agent (AI agent)             | Parses raw_text into JSON array of expenses            | Set: Text Input; Set: Voice Input (Text); Set: image Input (Text) | Split: Entries to Rows            | ## 3) Expense Parsing & Validation<br>Convert free-form text into structured expenses.                                              |
| Auto-Fix JSON               | Output Parser Autofixing               | Auto-corrects malformed JSON output                    | LLM: Gemini Flash            | AI: Extract Expenses (ai_outputParser) | ## 3) Expense Parsing & Validation<br>Convert free-form text into structured expenses.                                              |
| AI Helper: Define Schema    | Output Parser Structured               | Enforces JSON schema for AI output                     | LLM: Gemini Flash            | Auto-Fix JSON                    | ## 3) Expense Parsing & Validation<br>Convert free-form text into structured expenses.                                              |
| LLM: Gemini Flash           | LangChain LLM (Google Gemini)          | Language model used for parsing and auto-fixing       | —                           | Auto-Fix JSON; AI: Extract Expenses | ## 3) Expense Parsing & Validation<br>Convert free-form text into structured expenses.                                              |
| Split: Entries to Rows      | SplitOut                             | Splits JSON array into individual expense entries     | AI: Extract Expenses         | If null?                        | ## 4) Row Generation, Sheet Write & Confirmation<br>Turn parsed entries into rows, save them, and notify the user.                 |
| If null?                   | If                                    | Checks if current expense entry is null/empty         | Split: Entries to Rows        | Set: Extract Output (true); Telegram: Send Null (false) | ## 4) Row Generation, Sheet Write & Confirmation<br>Turn parsed entries into rows, save them, and notify the user.                 |
| Set: Extract Output         | Set                                   | Maps expense fields to explicit column names          | If null? (true)              | GSheet: Log Expense              | ## 4) Row Generation, Sheet Write & Confirmation<br>Turn parsed entries into rows, save them, and notify the user.                 |
| GSheet: Log Expense         | Google Sheets                         | Appends expense rows to Google Sheet                   | Set: Extract Output          | Telegram: Send Confirmation      | ## 4) Row Generation, Sheet Write & Confirmation<br>Turn parsed entries into rows, save them, and notify the user.                 |
| Telegram: Send Confirmation | Telegram (send message)                | Sends summary of logged expenses                        | GSheet: Log Expense          | —                              | ## 4) Row Generation, Sheet Write & Confirmation<br>Turn parsed entries into rows, save them, and notify the user.                 |
| Telegram: Send Null         | Telegram (send message)                | Sends message when no valid expense entries found      | If null? (false)             | —                              | ## 4) Row Generation, Sheet Write & Confirmation<br>Turn parsed entries into rows, save them, and notify the user.                 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Credentials:**
   - Use BotFather on Telegram to create a bot and obtain its Bot Token.
   - Add Telegram API credential in n8n using this token.

2. **Create Google Sheets OAuth2 Credentials:**
   - Set up OAuth2 credentials in n8n to access Google Sheets.
   - Share the target Google Sheet with the OAuth2 user (or use your own account).

3. **Create Google Gemini API Credential:**
   - Add Google AI (Gemini) API credentials in n8n for voice transcription and image analysis.

4. **Create Google Sheet:**
   - Prepare a Google Sheet named e.g. "Log Expense" with an appropriate sheet tab.
   - Note its Document ID and Sheet Name (tab ID).

5. **Add Node: Start: On Telegram Message**
   - Type: Telegram Trigger.
   - Configure to listen for `message` updates.
   - Select Telegram credential.

6. **Add Node: Route: Text, Voice or Image?**
   - Type: Switch node.
   - Add conditions:
     - Text route: Check if `message.text` exists.
     - Voice route: Check if `message.voice` exists.
     - Image route: Check if `message.photo` exists.
     - Else: fallback for unsupported types.

7. **Add Node: Telegram: Send error message**
   - Type: Telegram node.
   - Text: "🤖Sorry, I can only understand text, voice and image for bookkeeping."
   - Connect fallback output of router here.

8. **Text Path:**
   - Add Set node "Set: Text Input".
   - Assign `raw_text = {{ $json.message.text }}`.
   - Connect from router Text output.

9. **Voice Path:**
   - Add Telegram node "Telegram: Get Voice File".
   - File ID: `{{ $json.message.voice.file_id }}`
   - Connect from router Voice output.
   - Add Gemini node "Gemini: Transcribe Voice".
     - Model: `models/gemini-2.5-flash`
     - Input: Audio binary from previous node, property `data`.
   - Add Set node "Set: Voice Input (Text)".
     - Assign `raw_text = {{ $json.content.parts.toJsonString() }}`
   - Connect nodes sequentially.

10. **Image Path:**
    - Add Telegram node "Telegram: Get Image File".
      - File ID: `{{ $json.message.photo[0].file_id }}`
    - Add Gemini node "Gemini: Analyze an image".
      - Model: `models/gemini-2.5-flash`
      - Text prompt instructing to extract all financial text and numbers visible.
      - Input: Binary image.
    - Add Set node "Set: image Input (Text)".
      - Assign `raw_text = {{ $json.content.parts.toJsonString() }}`
    - Connect nodes sequentially.

11. **Add AI Agent Node "AI: Extract Expenses"**
    - Type: LangChain Agent.
    - Input text: `information:{{ $json.raw_text }}`
    - System message: Include detailed prompt specifying output as JSON array with keys `amount`, `item`, `date`, strict rules on parsing and normalization.
    - Enable output parser.
    - Connect outputs of all three Set nodes ("Set: Text Input", "Set: Voice Input (Text)", "Set: image Input (Text)") to this node.

12. **Add LLM Node "LLM: Gemini Flash"**
    - Set as the AI language model for the agent and autofix nodes.
    - Use the Gemini API credential.

13. **Add Output Parser Nodes:**
    - "Auto-Fix JSON" node connected to LLM node output to fix malformed JSON.
    - "AI Helper: Define Schema" node defining the expected JSON schema.
    - Connect "Auto-Fix JSON" output back to "AI: Extract Expenses" parser input for validation.

14. **Add Split Node "Split: Entries to Rows"**
    - Field to split: `output` (JSON array of expenses).
    - Connect from "AI: Extract Expenses" output.

15. **Add If Node "If null?"**
    - Condition: Check if current item is null or empty in `output`.
    - Connect from split node.

16. **Add Set Node "Set: Extract Output"**
    - Assign fields for Google Sheets:
      - `Amount = {{ $json.output.amount }}`
      - `Item = {{ $json.output.item }}`
      - `Date = {{ $json.output.date }}`
    - Connect true output of If node here.

17. **Add Google Sheets Node "GSheet: Log Expense"**
    - Operation: Append rows.
    - Document ID and Sheet Name: Set to your prepared Google Sheet.
    - Map columns automatically from Set node.
    - Connect from Set: Extract Output.

18. **Add Telegram Node "Telegram: Send Confirmation"**
    - Sends a summary message listing all saved expenses.
    - Use expression to build MarkdownV2 escaped text listing items with `item — amount (date)`.
    - Set correct chat ID.
    - Connect from Google Sheets node.

19. **Add Telegram Node "Telegram: Send Null"**
    - Sends message "📒 Bookkeeping Complete\n\nnull"
    - Connect from false output of If node ("If null?").

20. **Verify all credentials and parameters are correctly set.**

21. **Activate the workflow and test by sending text, voice, and image messages to your Telegram bot.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| The workflow uses Telegram's MarkdownV2 format for message confirmation, requiring special characters to be escaped carefully to avoid formatting errors.                                                                                                                                                                                                                                                                                                                                                                       | See Telegram MarkdownV2 documentation.                                                                             |
| The Gemini AI models used (`models/gemini-2.5-flash`) provide advanced text extraction from audio and images, but API rate limits and quota management should be considered in production.                                                                                                                                                                                                                                                                                                                                       | Gemini API documentation.                                                                                           |
| The expense parser prompt enforces strict JSON output with no hallucinations, ensuring downstream processes receive clean, deterministic data. Adjust the prompt carefully when modifying the expense extraction logic.                                                                                                                                                                                                                                                                                                            | Prompt is embedded in the "AI: Extract Expenses" node.                                                             |
| Google Sheets append operation uses RAW cell format to prevent unwanted formatting or type conversion. Ensure the target sheet's permissions allow appending rows.                                                                                                                                                                                                                                                                                                                                                                 | Google Sheets API specification.                                                                                    |
| Telegram bot chat ID must be set explicitly in confirmation and error messaging nodes. This can be set dynamically if needed by extending the workflow.                                                                                                                                                                                                                                                                                                                                                                         | Telegram Bot API.                                                                                                   |
| The workflow is structured to fail fast with friendly messages on unsupported input types or empty expense extractions, improving UX.                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note #1 provides a quick start overview and setup instructions.                                            |
| For multi-entry messages, the AI parser separates entries based on punctuation and conjunctions, allowing efficient batch processing of multiple expenses in one message.                                                                                                                                                                                                                                                                                                                                                        | See AI prompt examples in the "AI: Extract Expenses" node.                                                         |
| Workflow designed for n8n version supporting LangChain nodes and Google Gemini integration; ensure environment compatibility.                                                                                                                                                                                                                                                                                                                                                                                                   | n8n version >= 0.210 recommended, with LangChain and Google Gemini nodes installed.                                |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow built with n8n, adhering strictly to content policies and containing no illegal or protected elements. All processed data is legal and public.