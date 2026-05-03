Expense Logging with Telegram to Google Sheets using AI Voice & Text Parsing

https://n8nworkflows.xyz/workflows/expense-logging-with-telegram-to-google-sheets-using-ai-voice---text-parsing-10993


# Expense Logging with Telegram to Google Sheets using AI Voice & Text Parsing

### 1. Workflow Overview

This workflow automates logging of multiple expenses sent via Telegram messages—either text or voice notes—directly into a Google Sheet. It uses AI to parse the input messages into structured expense records, handles audio transcription if needed, appends each expense to the sheet, and sends a confirmation message back to the user.

The workflow is logically divided into these blocks:

- **1.1 Telegram Input & Audio Transcription:** Receives incoming Telegram messages, detects if input is audio or text, and transcribes audio messages.
- **1.2 AI Parsing & Expense Extraction:** Uses OpenAI's GPT model to extract structured expense data in a strict JSON schema.
- **1.3 Expense Processing & Google Sheets Integration:** Splits parsed expenses into individual items, appends each to a Google Sheet, and prepares a summary.
- **1.4 Confirmation Messaging:** Sends a confirmation message to Telegram summarizing the logged expenses.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input & Audio Transcription

**Overview:**  
This block handles incoming Telegram messages, checks if the message contains an audio voice note, and if so, downloads and transcribes it into text. Text messages are passed as-is.

**Nodes Involved:**  
- Telegram Message Trigger  
- Check if Audio file (If)  
- Get File  
- Transcribe audio  
- Set field (for text messages)  

**Node Details:**

- **Telegram Message Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point for all Telegram messages (text or voice).  
  - Configuration: Listens to "message" updates on Telegram.  
  - Inputs: Incoming Telegram messages.  
  - Outputs: Message data including text or voice object.  
  - Notes: Requires Telegram API credentials.  
  
- **Check if Audio file**  
  - Type: If node  
  - Role: Determines if the incoming message contains a voice note.  
  - Configuration: Checks if the `message.voice` field is not empty.  
  - Inputs: Telegram message JSON.  
  - Outputs: Two branches: True (audio present), False (text message).  
  - Edge cases: Empty or unsupported message types might bypass transcription.  
  
- **Get File**  
  - Type: Telegram node (get file)  
  - Role: Downloads the voice note file from Telegram for transcription.  
  - Configuration: Uses the `file_id` from the voice message.  
  - Inputs: Audio file id from Telegram message.  
  - Outputs: Audio file data.  
  - Edge cases: File download failure, network issues.  
  
- **Transcribe audio**  
  - Type: OpenAI Audio Transcription node  
  - Role: Sends audio file to OpenAI Whisper model for transcription to text.  
  - Configuration: Uses OpenAI credentials, Whisper model selected for transcription.  
  - Inputs: Audio file from Get File node.  
  - Outputs: Transcribed text.  
  - Edge cases: API errors, unsupported audio format, transcription inaccuracies.  
  
- **Set field**  
  - Type: Set node  
  - Role: For text messages, extracts text into a common field named `text` for downstream processing.  
  - Configuration: Sets field `text` to the message text string.  
  - Inputs: Telegram message JSON for text messages.  
  - Outputs: JSON with unified `text` field.  

---

#### 1.2 AI Parsing & Expense Extraction

**Overview:**  
This block sends the unified text (from transcription or direct text) to an OpenAI GPT-4o-mini model configured to parse expense data strictly into a JSON array of expense objects matching a predefined schema. The structured output parser validates this JSON.

**Nodes Involved:**  
- OpenAI Chat Model  
- Parse Expenses with AI (Langchain Chain LLM)  
- Structured Output Parser  
- Split Out  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Runs GPT-4o-mini to generate the expense parsing output.  
  - Configuration: Model set to `gpt-4o-mini`; uses OpenAI API credentials.  
  - Inputs: Text from the previous block (via Langchain chain node).  
  - Outputs: Raw GPT output with expense JSON array.  
  - Edge cases: Rate limiting, invalid prompts, API errors.  
  
- **Parse Expenses with AI**  
  - Type: Langchain Chain LLM node  
  - Role: Defines prompt instructing the AI to output ONLY a JSON array with fields Date, Category, Merchant, Amount, Note, following strict rules.  
  - Configuration: Prompt includes fallback rules (e.g., date defaults to today, category inference). Uses expressions to inject Telegram or audio text dynamically.  
  - Inputs: Text field named `text`.  
  - Outputs: Parsed JSON array of expenses.  
  - Edge cases: AI output not conforming to schema, empty or invalid input text.  
  
- **Structured Output Parser**  
  - Type: Langchain Output Parser (Structured)  
  - Role: Validates that AI output matches the explicit JSON schema for expenses.  
  - Configuration: JSON schema requires Date (YYYY-MM-DD), Category (from predefined set), optional Merchant and Note, Amount as number.  
  - Inputs: Raw AI output string.  
  - Outputs: Parsed and validated array of expense objects.  
  - Edge cases: Parsing failures if AI output deviates from schema.  
  
- **Split Out**  
  - Type: Split Out node  
  - Role: Separates the array of expense objects into individual items for further processing.  
  - Configuration: Splits on the `output` field.  
  - Inputs: Array of expenses.  
  - Outputs: Individual expense JSON objects (one per item).  

---

#### 1.3 Expense Processing & Google Sheets Integration

**Overview:**  
Processes each expense item individually, appends it as a new row in the Google Sheet, waits briefly between inserts to avoid race conditions, then aggregates the expenses for confirmation.

**Nodes Involved:**  
- Loop Over Items (Split in Batches)  
- Append to Google Sheet  
- Wait 0.5 seconds  
- Build Expense Summary Text  
- Merge  

**Node Details:**

- **Loop Over Items**  
  - Type: Split in Batches node  
  - Role: Iterates over each expense item one at a time for sequential processing.  
  - Configuration: Default batch size (1).  
  - Inputs: Individual expense items from Split Out.  
  - Outputs: Passes one item per execution to next nodes.  
  - Edge cases: Large batch sizes could cause API rate limits.  
  
- **Append to Google Sheet**  
  - Type: Google Sheets node  
  - Role: Appends each expense as a new row in the specified Google Sheet and Sheet tab.  
  - Configuration: Maps expense fields Date, Category, Merchant, Amount, Note to columns; appends rows; uses Google OAuth2 credentials.  
  - Inputs: Single expense JSON item.  
  - Outputs: Confirmation of append operation.  
  - Edge cases: Invalid Sheet ID, wrong column mapping, permission errors, API quota.  
  
- **Wait 0.5 seconds**  
  - Type: Wait node  
  - Role: Adds a 500ms delay after each row append to prevent race conditions or API throttling.  
  - Configuration: Wait time set to 0.5 seconds.  
  - Inputs: Append operation completion.  
  - Outputs: Delayed trigger for next batch item processing.  
  
- **Build Expense Summary Text**  
  - Type: Code (JavaScript) node  
  - Role: Aggregates all individual expense items into a human-readable summary text for confirmation.  
  - Configuration: Builds lines like `Date — Category $Amount (Merchant)` and counts total expenses.  
  - Inputs: All split out items before batch looping.  
  - Outputs: JSON object with a `text` field summarizing expenses.  
  
- **Merge**  
  - Type: Merge node  
  - Role: Combines two branches: one from batch append finish, one from summary text, to prepare confirmation message.  
  - Configuration: Mode chooseBranch, outputs summary branch for confirmation sending.  

---

#### 1.4 Confirmation Messaging

**Overview:**  
Sends a confirmation message back to the original Telegram chat, showing the user a summary of all expenses logged.

**Nodes Involved:**  
- Build Confirmation Text  
- Send Telegram Confirmation  

**Node Details:**

- **Build Confirmation Text**  
  - Type: Set node  
  - Role: Prepares the text message payload for Telegram using the summary text built earlier.  
  - Configuration: Sets `text` field to the summary message string.  
  - Inputs: Merged JSON summary.  
  - Outputs: Formatted message for Telegram.  
  
- **Send Telegram Confirmation**  
  - Type: Telegram node  
  - Role: Sends the confirmation message back to the Telegram chat using the original chat ID.  
  - Configuration: Uses Telegram API credentials, disables attribution append, dynamically sets chat ID from trigger node.  
  - Inputs: Confirmation message JSON.  
  - Outputs: Sends message to user.  
  - Edge cases: Telegram API rate limits, invalid chat ID, network errors.  

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role                         | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                               |
|---------------------------|-----------------------------------|---------------------------------------|----------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------|
| Telegram Message Trigger   | n8n-nodes-base.telegramTrigger    | Entry point, receives Telegram input  | -                                | Check if Audio file                 | Setup instructions, purpose, and input notes grouped in sticky notes                                     |
| Check if Audio file        | n8n-nodes-base.if                 | Detects if message contains audio     | Telegram Message Trigger          | Get File (true), Set field (false) |                                                                                                           |
| Get File                  | n8n-nodes-base.telegram           | Downloads voice audio file             | Check if Audio file (true)        | Transcribe audio                   |                                                                                                           |
| Transcribe audio           | @n8n/n8n-nodes-langchain.openAi   | Transcribes audio to text              | Get File                        | Parse Expenses with AI             |                                                                                                           |
| Set field                  | n8n-nodes-base.set                | Extracts text from text messages       | Check if Audio file (false)       | Parse Expenses with AI             |                                                                                                           |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi | Runs GPT-4o-mini model for parsing    | Parse Expenses with AI            | Parse Expenses with AI             |                                                                                                           |
| Parse Expenses with AI     | @n8n/n8n-nodes-langchain.chainLlm | AI parsing with strict JSON output     | OpenAI Chat Model / Transcribe audio / Set field | Structured Output Parser           |                                                                                                           |
| Structured Output Parser   | @n8n/n8n-nodes-langchain.outputParserStructured | Validates AI output JSON schema        | Parse Expenses with AI            | Split Out                        |                                                                                                           |
| Split Out                 | n8n-nodes-base.splitOut           | Splits array of expenses into items   | Structured Output Parser          | Loop Over Items, Build Expense Summary Text |                                                                                                           |
| Loop Over Items            | n8n-nodes-base.splitInBatches     | Processes each expense item sequentially | Split Out                    | Append to Google Sheet, Merge      |                                                                                                           |
| Append to Google Sheet     | n8n-nodes-base.googleSheets        | Appends each expense row to Google Sheets | Loop Over Items               | Wait 0.5 seconds                   |                                                                                                           |
| Wait 0.5 seconds           | n8n-nodes-base.wait                | Delay to avoid race conditions         | Append to Google Sheet            | Loop Over Items                   | Troubleshooting note suggests increasing wait time for missing rows                                       |
| Build Expense Summary Text | n8n-nodes-base.code               | Creates summary text of expenses       | Split Out                       | Merge                           |                                                                                                           |
| Merge                     | n8n-nodes-base.merge               | Combines expense appends with summary  | Loop Over Items, Build Expense Summary Text | Build Confirmation Text           |                                                                                                           |
| Build Confirmation Text    | n8n-nodes-base.set                | Prepares confirmation message text    | Merge                           | Send Telegram Confirmation         |                                                                                                           |
| Send Telegram Confirmation | n8n-nodes-base.telegram           | Sends confirmation back to Telegram   | Build Confirmation Text           | -                                |                                                                                                           |
| Sticky Note1               | n8n-nodes-base.stickyNote         | Troubleshooting tips                   | -                                | -                                | See note content in section 5                                                                              |
| Sticky Note2               | n8n-nodes-base.stickyNote         | Setup instructions                    | -                                | -                                |                                                                                                           |
| Sticky Note3               | n8n-nodes-base.stickyNote         | Telegram Input & Transcription overview | -                              | -                                |                                                                                                           |
| Sticky Note4               | n8n-nodes-base.stickyNote         | AI Parsing & Expense Extraction overview | -                              | -                                |                                                                                                           |
| Sticky Note5               | n8n-nodes-base.stickyNote         | Sheets writing & Confirmation overview | -                              | -                                |                                                                                                           |
| Sticky Note                | n8n-nodes-base.stickyNote         | Workflow purpose description           | -                                | -                                |                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates  
   - Credentials: Connect Telegram API credentials  
   - Position: Start of workflow  

2. **Add If Node "Check if Audio file"**  
   - Type: If node  
   - Condition: Check if `message.voice` field is not empty (strict, not case sensitive)  
   - Inputs: Connect Telegram Trigger output  

3. **Create Telegram "Get File" Node**  
   - Type: Telegram node (resource: file)  
   - Parameters: fileId = `{{ $json.message.voice.file_id }}`  
   - Credentials: Use Telegram credentials  
   - Connect True output of If node to this node  

4. **Add OpenAI Audio Transcription Node "Transcribe audio"**  
   - Type: OpenAI node, resource: audio, operation: transcribe  
   - Credentials: OpenAI API credentials (with Whisper enabled)  
   - Connect output of Get File node here  

5. **Create Set Node "Set field"**  
   - Type: Set node  
   - Parameters: Assign `text` field = `{{ $json.message.text }}`  
   - Connect False output of If node here  

6. **Create OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Parameters: Model = `gpt-4o-mini`  
   - Credentials: OpenAI API credentials  
   - Connect outputs of Transcribe audio and Set field nodes to this node  

7. **Add Langchain Chain LLM Node "Parse Expenses with AI"**  
   - Type: Langchain Chain LLM  
   - Parameters:  
     - Prompt: Define prompt instructing AI to parse expenses strictly into JSON array with fields Date, Category, Merchant, Amount, Note, using rules described (date today default, etc.)  
     - Input: Use expression referencing `Telegram Message Trigger` text or transcribed text as input  
     - Enable output parser  
   - Connect OpenAI Chat Model output here  

8. **Add Langchain Structured Output Parser Node**  
   - Type: Output Parser Structured  
   - Parameters: JSON schema specifying array of objects with required fields and data types, including regex for date format and enum for Category  
   - Connect output of Parse Expenses with AI here  

9. **Add Split Out Node**  
   - Type: Split Out  
   - Parameters: Split by field `output` (the array of expenses)  
   - Connect Structured Output Parser output here  

10. **Add Split In Batches Node "Loop Over Items"**  
    - Type: Split In Batches  
    - Parameters: Default batch size 1 (process one expense at a time)  
    - Connect Split Out output here  

11. **Add Google Sheets Node "Append to Google Sheet"**  
    - Type: Google Sheets Append  
    - Parameters:  
      - Spreadsheet ID: Your Google Sheet ID  
      - Sheet Name: Your sheet tab name or ID  
      - Columns mapping: Map fields Date, Category, Merchant, Amount, Note accordingly  
      - Operation: Append  
    - Credentials: Google OAuth2 credentials  
    - Connect Loop Over Items output here  

12. **Add Wait Node "Wait 0.5 seconds"**  
    - Type: Wait  
    - Parameters: Amount 0.5 seconds  
    - Connect Append to Google Sheet output here  

13. **Connect Wait node output back to Loop Over Items input**  
    - This creates a loop to process all expenses sequentially with delay  

14. **Add Code Node "Build Expense Summary Text"**  
    - Type: Code (JavaScript)  
    - Parameters: Code to build a summary string from all split out expense items, formatting date, category, amount, and optional merchant, counting total items  
    - Connect Split Out output also to this node (parallel branch)  

15. **Add Merge Node**  
    - Type: Merge (mode: Choose Branch)  
    - Connect Loop Over Items second output (index 1) and Build Expense Summary Text outputs here  

16. **Add Set Node "Build Confirmation Text"**  
    - Type: Set  
    - Parameters: Assign `text` field to `{{ $json.text }}` (summary text from Merge node)  
    - Connect Merge output here  

17. **Add Telegram Node "Send Telegram Confirmation"**  
    - Type: Telegram  
    - Parameters:  
      - Text: `{{ $json.text }}`  
      - Chat ID: `{{ $('Telegram Message Trigger').first().json.message.chat.id }}`  
      - Additional Fields: Disable append attribution  
    - Credentials: Telegram API credentials  
    - Connect Build Confirmation Text output here  

18. **Add Sticky Notes**  
    - Add notes describing purpose, setup instructions, troubleshooting, and block explanations as per sticky note content for ease of maintenance  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🛠️ Troubleshooting: Increase wait time between appends if rows are missing; ensure messages include numbers and merchants; verify Google Sheet ID and column mapping; confirm Whisper is selected for audio transcription.                                                                                                                                                 | Sticky Note1 content                                                                                                                                |
| 🔧 Setup Instructions: Connect Telegram, Google, OpenAI credentials; prepare Google Sheet with headers Date, Category, Merchant, Amount, Note; map columns correctly; use GPT-4o-mini for chat; Whisper for audio; keep wait node at 500–1000 ms; test with example message `Gas 34.67, Groceries 82.45, Coffee 6.25, Lunch 14.90`.                                                                                                     | Sticky Note2 content                                                                                                                                |
| 🎤 Telegram Input & Transcription: Handles both text and voice notes to unify into text for AI processing.                                                                                                                                                                                                                                                                 | Sticky Note3 content                                                                                                                                |
| 🤖 AI Parsing & Expense Extraction: Uses AI to reliably extract multiple expenses from a single message into structured JSON.                                                                                                                                                                                                                                           | Sticky Note4 content                                                                                                                                |
| 📊 Write to Sheets & Send Confirmation: Writes individual expenses to Google Sheet and sends a summary back to Telegram for user confirmation.                                                                                                                                                                                                                         | Sticky Note5 content                                                                                                                                |
| 📌 Purpose: Enables fast, zero-manual-entry expense logging via Telegram text or voice messages, ideal for busy users who prefer mobile, hands-free expense tracking without full accounting software.                                                                                                                                                                  | Sticky Note (large purpose description)                                                                                                            |
| Uses OpenAI GPT-4o-mini model and Whisper for transcription, Google Sheets for data storage, and Telegram for user interaction.                                                                                                                                                                                                                                         | Workflow design notes                                                                                                                               |

---

**Disclaimer:** The text provided is exclusively from an n8n automated workflow. It strictly complies with content policies and contains no illegal or offensive material. All manipulated data is legal and public.