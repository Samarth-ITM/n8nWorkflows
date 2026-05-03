Track Expenses from Telegram to Google Sheets with GPT-4.1 Mini

https://n8nworkflows.xyz/workflows/track-expenses-from-telegram-to-google-sheets-with-gpt-4-1-mini-6970


# Track Expenses from Telegram to Google Sheets with GPT-4.1 Mini

---

### 1. Workflow Overview

This workflow automates tracking personal expenses by receiving user messages via Telegram, analyzing them with GPT-4.1 Mini, and logging validated expense records into a Google Sheet. It is designed for individuals or small teams who want a simple, conversational interface to capture daily expenses without manual data entry.

**Target Use Cases:**  
- Individuals tracking daily spending via chat  
- Freelancers logging receipts on the go  
- Small businesses or teams needing lightweight expense capture  

**Logical Blocks:**  
- **1.1 Input Reception:** Capture incoming Telegram messages via a Telegram bot.  
- **1.2 Message Type Validation:** Determine if the incoming message is text; reject unsupported types (images/files).  
- **1.3 AI Processing:** Send text messages to GPT-4.1 Mini to extract structured expense data in JSON.  
- **1.4 Output Parsing & Validation:** Parse AI response and verify if it contains a valid, relevant expense record.  
- **1.5 Expense Acknowledgement & Logging:** If relevant, confirm to user and transform the data for logging to Google Sheets.  
- **1.6 Unsupported Scenario Handling:** If the message or AI response is not an expense, send a polite fallback message.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for incoming Telegram messages using a Telegram bot webhook. Downloads any attached files but primarily awaits text messages.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Configuration: Listens for "message" updates; downloads content if available.  
    - Credentials: Connected to Telegram bot API token.  
    - Input: External Telegram webhook message.  
    - Output: Message JSON including chat ID, message ID, text, attachments, etc.  
    - Edge Cases: Telegram API errors or invalid webhook setup; messages without text content.

---

#### 1.2 Message Type Validation

- **Overview:**  
  Checks if the incoming Telegram message contains text. Routes non-text messages to an unsupported message handler.

- **Nodes Involved:**  
  - Is text message? (If node)  
  - Un-supported message type (Telegram node)

- **Node Details:**  
  - **Is text message?**  
    - Type: If node  
    - Configuration: Checks if message JSON contains a "text" field (string contains "text").  
    - Input: Output from Telegram Trigger.  
    - Output: True branch to AI processing; False branch to unsupported message reply.  
    - Edge Cases: Messages with malformed JSON or missing text field may cause false negatives.  
  - **Un-supported message type**  
    - Type: Telegram node (Send message)  
    - Configuration: Sends a polite fallback message: "Sorry, I can’t read files or images right now. Just send me a message describing what you spent, and I’ll help you track it! 💬💸"  
    - Credentials: Same Telegram bot API as trigger.  
    - Input: Chat ID from Telegram Trigger node.  
    - Edge Cases: Telegram API errors; chat ID missing or invalid.

---

#### 1.3 AI Processing

- **Overview:**  
  Sends the user's text message to GPT-4.1 Mini model with a detailed system prompt instructing it to extract a structured expense record in JSON format.

- **Nodes Involved:**  
  - Budget Buddy Telegram Agent Text (Langchain Agent node)  
  - OpenAI Chat Model2 (Langchain OpenAI Chat Model)  
  - Structured Output Parser (Langchain Output Parser)

- **Node Details:**  
  - **Budget Buddy Telegram Agent Text**  
    - Type: Langchain Agent node  
    - Configuration: Receives input text from the Telegram message and sends it to GPT-4.1 Mini with a system prompt that:  
      - Defines the bot as a friendly personal finance assistant  
      - Extracts a JSON with keys: `relevant` (boolean), `expense_record` (object with date, amount, currency, category, description), and `message` (confirmation/fallback text)  
      - Enforces JSON-only output with no explanations  
      - Provides a fixed list of expense categories for classification  
    - Input: Text message from "Is text message?" node  
    - Output: Raw AI JSON response  
    - Edge Cases: API call failures, timeout, model response not conforming to JSON format.  
    - Credentials: OpenAI API key for GPT-4.1 Mini access.  
  - **OpenAI Chat Model2**  
    - Type: Langchain OpenAI Chat Model  
    - Configuration: Model set to "gpt-4.1-mini"  
    - Input/Output: Connected as AI language model for Budget Buddy Agent node.  
  - **Structured Output Parser**  
    - Type: Langchain Output Parser Structured  
    - Configuration: Uses a JSON schema example matching the expected AI output to parse and validate the AI response.  
    - Input: AI response from Budget Buddy Agent node  
    - Output: Parsed JSON object with structured fields  
    - Edge Cases: Parsing failures if AI returns invalid JSON or unexpected structure.

---

#### 1.4 Output Parsing & Validation

- **Overview:**  
  Checks the parsed AI output to determine if the message contains a valid expense (`relevant`=true). Routes accordingly to either proceed with logging or send fallback message.

- **Nodes Involved:**  
  - Supported scenario? (If node)  
  - Acknowledge the expense (Telegram node)  
  - Send un-supported scenario message (Telegram node)

- **Node Details:**  
  - **Supported scenario?**  
    - Type: If node  
    - Configuration: Checks parsed JSON field `output.relevant` for true (boolean).  
    - Input: Parsed AI output from Structured Output Parser.  
    - Output: True branch to acknowledge and log expense; false branch to send fallback message.  
    - Edge Cases: Missing or malformed `relevant` field.  
  - **Acknowledge the expense**  
    - Type: Telegram node (Send message)  
    - Configuration: Sends AI-generated friendly confirmation message back to user. Uses `output.message` from AI.  
    - Input: Chat ID from Telegram Trigger node.  
    - Credentials: Telegram API token.  
    - Edge Cases: Telegram API errors; missing chat ID.  
  - **Send un-supported scenario message**  
    - Type: Telegram node (Send message)  
    - Configuration: Sends AI-generated fallback message when the content is not a valid expense (e.g., "sorry" message from AI).  
    - Input: Chat ID from Telegram Trigger node.  
    - Credentials: Telegram API token.

---

#### 1.5 Expense Acknowledgement & Logging

- **Overview:**  
  Transforms the validated AI expense output into a flatter object and appends it as a new row to a Google Sheets document for expense tracking.

- **Nodes Involved:**  
  - Transform the output to expense record (Code node)  
  - Log expense record to google sheet (Google Sheets node)

- **Node Details:**  
  - **Transform the output to expense record**  
    - Type: Code (JavaScript) node  
    - Configuration: Extracts fields from AI parsed output: date, amount, currency, category, description; also adds Telegram message ID and chat ID for traceability.  
    - Input: Parsed AI output and Telegram Trigger data.  
    - Output: Flattened JSON object matching Google Sheet columns.  
    - Edge Cases: Missing fields in AI output; type mismatches.  
  - **Log expense record to google sheet**  
    - Type: Google Sheets node  
    - Configuration: Appends a new row automatically mapped from input data to the sheet columns: Date, Amount, Currency, Category, Description, Source, File ID.  
    - Sheet: Fixed Google Sheet document ID and Sheet name ("gid=0")  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Edge Cases: Google API errors; permission issues; invalid sheet ID or column mismatch.

---

#### 1.6 Unsupported Scenario Handling

- **Overview:**  
  Handles user messages that are not recognized as valid expenses by sending a polite fallback message encouraging correct input.

- **Nodes Involved:**  
  - Send un-supported scenario message (Telegram node) *[also listed above]*  
  - Un-supported message type (Telegram node) *[also listed above]*

- **Node Details:**  
  - Covered above in sections 1.2 and 1.4.

---

### 3. Summary Table

| Node Name                        | Node Type                              | Functional Role                                 | Input Node(s)              | Output Node(s)                     | Sticky Note                                                                                               |
|---------------------------------|--------------------------------------|------------------------------------------------|----------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------|
| Telegram Trigger                | Telegram Trigger                     | Entry point; receives Telegram messages         | External webhook            | Is text message?                  | ### 1. 📩 Telegram Trigger  \n**Description**: Listens for incoming messages from the user via the connected Telegram bot. This is the entry point of the workflow. |
| Is text message?               | If node                             | Checks if message is text                        | Telegram Trigger            | Budget Buddy Telegram Agent Text, Un-supported message type | ### 2. Is Text Message?  \n**Description**: Checks whether the incoming Telegram message is a text message. If not, the workflow routes to an "unsupported message type" handler. |
| Budget Buddy Telegram Agent Text | Langchain Agent (OpenAI GPT-4.1 Mini) | Sends text to GPT-4.1 Mini AI to extract expense JSON | Is text message?            | Supported scenario?               | ### 3. 💬 Budget Buddy Telegram Agent (GPT-4.1 Mini)  \n**Description**: Sends the user’s text message to the GPT-4.1 Mini model along with a system prompt. The model returns a JSON object with `relevant`, `expense_record`, and `message`. |
| OpenAI Chat Model2             | Langchain OpenAI Chat Model         | OpenAI GPT-4.1 Mini model invoked by Agent node | Budget Buddy Telegram Agent Text | Budget Buddy Telegram Agent Text (AI response) |                                                                                                           |
| Structured Output Parser       | Langchain Output Parser Structured  | Parses AI JSON output safely                     | Budget Buddy Telegram Agent Text | Supported scenario?               |                                                                                                           |
| Supported scenario?            | If node                            | Checks if AI response marks message as relevant | Structured Output Parser    | Acknowledge the expense, Send un-supported scenario message |                                                                                                           |
| Acknowledge the expense       | Telegram                            | Sends confirmation message to user               | Supported scenario? (true)  | Transform the output to expense record | ### 4.1. ✅ Acknowledge the Expense  \n**Description**: Sends a friendly confirmation message back to the user on Telegram, using the `message` field returned by the LLM. |
| Transform the output to expense record | Code (JavaScript)                   | Flattens AI data to Google Sheets format         | Acknowledge the expense     | Log expense record to google sheet | ### 4.2 Transform & log expense\n- Transform the Output to Expense Record  \n- Log Expense Record to Google Sheet  \n\n |
| Log expense record to google sheet | Google Sheets                      | Appends expense record row to Google Sheet       | Transform the output to expense record | -                            |                                                                                                           |
| Send un-supported scenario message | Telegram                          | Sends fallback message if AI response not relevant | Supported scenario? (false) | -                                | ### 5 Let use know that the scenario is not supported\n- Only support budget expense\n- Let use know that their question/message is not relevant |
| Un-supported message type     | Telegram                            | Sends fallback for non-text message types         | Is text message? (false)    | -                                |                                                                                                           |
| Sticky Note                   | Sticky Note                        | Documentation and visual aid                      | -                          | -                                | Contains detailed workflow overview, setup instructions, and customization tips                           |
| Sticky Note1                  | Sticky Note                        | Screenshot illustration                           | -                          | -                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-04+at+9.32.37%E2%80%AFPM.png) |
| Sticky Note2                  | Sticky Note                        | Screenshot illustration                           | -                          | -                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-04+at+9.32.13%E2%80%AFPM.png) |
| Sticky Note3                  | Sticky Note                        | Screenshot illustration                           | -                          | -                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-04+at+9.28.40%E2%80%AFPM.png) |
| Sticky Note4                  | Sticky Note                        | Screenshot illustration                           | -                          | -                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-04+at+9.37.48%E2%80%AFPM.png) |
| Sticky Note5                  | Sticky Note                        | Describes Telegram Trigger node                   | -                          | -                                | ### 1. 📩 Telegram Trigger  \n**Description**: Listens for incoming messages from the user via the connected Telegram bot. This is the entry point of the workflow. |
| Sticky Note6                  | Sticky Note                        | Describes Is Text Message node                     | -                          | -                                | ### 2. Is Text Message?  \n**Description**: Checks whether the incoming Telegram message is a text message. If not, the workflow routes to an "unsupported message type" handler. |
| Sticky Note7                  | Sticky Note                        | Describes Budget Buddy Telegram Agent node        | -                          | -                                | ### 3. 💬 Budget Buddy Telegram Agent (GPT-4.1 Mini)  \n**Description**: Sends the user’s text message to the GPT-4.1 Mini model along with a system prompt. The model returns a JSON object with `relevant`, `expense_record`, and `message`. |
| Sticky Note8                  | Sticky Note                        | Describes Acknowledge the Expense node            | -                          | -                                | ### 4.1. ✅ Acknowledge the Expense  \n**Description**: Sends a friendly confirmation message back to the user on Telegram, using the `message` field returned by the LLM. |
| Sticky Note9                  | Sticky Note                        | Describes Transform & Log Expense nodes           | -                          | -                                | ### 4.2 Transform & log expense\n- Transform the Output to Expense Record  \n- Log Expense Record to Google Sheet  \n\n |
| Sticky Note10                 | Sticky Note                        | Describes unsupported scenario message handling   | -                          | -                                | ### 5 Let use know that the scenario is not supported\n- Only support budget expense\n- Let use know that their question/message is not relevant |
| Sticky Note11                 | Sticky Note                        | Workflow overview screenshot                       | -                          | -                                | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-04+at+9.49.43%E2%80%AFPM.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Bot and Get Token**  
   - Use [@BotFather](https://t.me/botfather) on Telegram to create a bot.  
   - Copy the bot token for n8n credentials.

2. **Add Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configuration: Listen for "message" updates, enable "download" for files.  
   - Credentials: Connect using Telegram Bot API token.  
   - Position: Entry point.

3. **Add IF Node "Is text message?"**  
   - Node Type: If  
   - Condition: Check if incoming message JSON contains "text" field (string contains "text").  
   - Input: Connect from Telegram Trigger.  
   - True branch: Proceed to AI processing.  
   - False branch: Connect to unsupported message handler.

4. **Add Telegram Node "Un-supported message type"**  
   - Node Type: Telegram (Send message)  
   - Text: "Sorry, I can’t read files or images right now. Just send me a message describing what you spent, and I’ll help you track it! 💬💸"  
   - Chat ID: `={{ $json.message.chat.id }}` from Telegram Trigger.  
   - Credentials: Same Telegram bot token.  
   - Connect False branch of "Is text message?" node here.

5. **Add Langchain OpenAI Chat Model Node "OpenAI Chat Model2"**  
   - Node Type: Langchain OpenAI Chat Model  
   - Model: "gpt-4.1-mini"  
   - Credentials: Use OpenAI API key with GPT-4.1 Mini access.

6. **Add Langchain Agent Node "Budget Buddy Telegram Agent Text"**  
   - Node Type: Langchain Agent  
   - Text Input: `={{ $json.message.text }}` from Telegram message.  
   - Options: Use system prompt (see below).  
   - Prompt Type: define (fixed system prompt).  
   - Link AI Model: Connect "OpenAI Chat Model2" as language model.  
   - Add Output Parser: Use Structured Output Parser node (next step).  
   - Connect True branch of "Is text message?" node here.

   **System Prompt Content (summary):**  
   - Role: Friendly budget assistant extracting expense data from plain text.  
   - Output: JSON with `relevant` (boolean), `expense_record` (fields: date, amount, currency, category, description), and `message` (friendly confirmation/fallback).  
   - Categories: Fixed list (Food, Transportation, Utilities, etc.).  
   - Instruction: Only output valid JSON, no extra text.

7. **Add Langchain Structured Output Parser Node**  
   - Node Type: Structured Output Parser  
   - JSON Schema Example:  
     ```json
     {
       "relevant": true,
       "expense_record": {
         "date": "2025-08-04",
         "amount": 120000,
         "currency": "VND",
         "category": "Food",
         "description": "Lunch at Phở Hòa with Linh"
       },
       "message": ""
     }
     ```  
   - Connect AI output from "Budget Buddy Telegram Agent Text" to this node.

8. **Add IF Node "Supported scenario?"**  
   - Node Type: If  
   - Condition: Check if `output.relevant` is true (boolean).  
   - Input: From Structured Output Parser output.  
   - True branch: Proceed to acknowledge and log expense.  
   - False branch: Proceed to unsupported scenario message.

9. **Add Telegram Node "Acknowledge the expense"**  
   - Node Type: Telegram (Send message)  
   - Text: `={{ $json.output.message }}` from AI parsed output.  
   - Chat ID: `={{ $('Telegram Trigger').first().json.message.chat.id }}`  
   - Credentials: Telegram bot token.  
   - Connect True branch of "Supported scenario?" node here.

10. **Add Code Node "Transform the output to expense record"**  
    - Node Type: Code (JavaScript)  
    - Code:  
      ```javascript
      const record = $input.first().json.output.expense_record;
      return {
        Date: record.date.toString(),
        Amount: record.amount,
        Currency: record.currency,
        Category: record.category,
        Description: record.description,
        MessageID: $('Telegram Trigger').first().json.message.message_id,
        ChatID: $('Telegram Trigger').first().json.message.chat.id
      };
      ```  
    - Input: From "Acknowledge the expense" node.  
    - Output: Flattened expense object.

11. **Add Google Sheets Node "Log expense record to google sheet"**  
    - Node Type: Google Sheets (Append row)  
    - Document ID: Your Google Sheet ID containing expense tracking data.  
    - Sheet Name: e.g., "gid=0" (usually the first sheet).  
    - Columns: Map automatically using input keys: Date, Amount, Currency, Category, Description, MessageID, ChatID.  
    - Credentials: Google Sheets OAuth2 account with edit permissions on the sheet.  
    - Input: Connect from Code node "Transform the output to expense record".

12. **Add Telegram Node "Send un-supported scenario message"**  
    - Node Type: Telegram (Send message)  
    - Text: `={{ $json.output.message }}` from AI parsed output (fallback message).  
    - Chat ID: `={{ $('Telegram Trigger').first().json.message.chat.id }}`  
    - Credentials: Same Telegram bot token.  
    - Connect False branch of "Supported scenario?" node here.

13. **Connect all branches properly**  
    - Telegram Trigger → Is text message?  
    - Is text message? True → Budget Buddy Telegram Agent Text  
    - Is text message? False → Un-supported message type  
    - Budget Buddy Telegram Agent Text → Structured Output Parser  
    - Structured Output Parser → Supported scenario?  
    - Supported scenario? True → Acknowledge the expense → Transform the output → Log expense  
    - Supported scenario? False → Send un-supported scenario message

14. **Test the workflow end-to-end**  
    - Send sample text expense messages in Telegram bot chat.  
    - Confirm friendly acknowledgment and check Google Sheet for new entries.  
    - Send non-text or irrelevant messages and confirm fallback responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is built using Telegram + GPT-4.1 Mini + Google Sheets + n8n, offering a lightweight personal finance tracking solution via chat.                                                                                                                                | Workflow description sticky note content.                                                                                                                   |
| Telegram Bot Setup Guide: Create a bot with [BotFather](https://t.me/botfather) and obtain the token.                                                                                                                                                                           | Telegram official bot creation documentation.                                                                                                               |
| Google Sheets Setup: Create a sheet with columns - Date, Amount, Currency, Category, Description, SourceMessage. Share it with your n8n service account email to allow writing.                                                                                                 | Google Sheets API and sharing documentation.                                                                                                                |
| OpenAI GPT-4.1 Mini Model: Use appropriate API key with access to GPT-4.1 Mini. The system prompt enforces strict JSON output for reliable parsing.                                                                                                                             | OpenAI API documentation.                                                                                                                                    |
| Categories list used for classification: Food, Transportation, Utilities, Shopping, Entertainment, Health, Education, Housing, Travel, Groceries, Subscriptions, Gifts, Insurance, Investment, Phone & Internet, Bank Fees, Charity, Childcare, Pets, Other.                       | Defined in system prompt.                                                                                                                                    |
| Workflow customization tips: Add multi-currency support, expand categories, track multiple users by adding username/chat ID columns, add alerts or summaries, connect to dashboards.                                                                                            | See Sticky Note content for customization table.                                                                                                            |
| Screenshots of workflow node configurations and output examples are included as sticky notes for visual reference.                                                                                                                                                              | See Sticky Note1 to Sticky Note4 for images.                                                                                                               |

---

_Disclaimer: The text provided is solely from an automated workflow created with n8n, respecting all content policies and containing no illegal or protected material. All data handled is legal and public._