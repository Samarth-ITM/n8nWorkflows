Automated Financial Tracker: Telegram Invoices to Notion with Gemini AI Reports

https://n8nworkflows.xyz/workflows/automated-financial-tracker--telegram-invoices-to-notion-with-gemini-ai-reports-3960


# Automated Financial Tracker: Telegram Invoices to Notion with Gemini AI Reports

### 1. Workflow Overview

This workflow automates financial expense tracking by capturing invoice images sent via Telegram, extracting detailed transaction data using AI (Google Gemini), recording the data into a Notion database, and providing immediate and scheduled summaries back to Telegram. It is designed for individuals or small teams who want to streamline expense recording and gain visual insights without manual data entry.

The workflow is logically divided into two main blocks:

- **1.1 Real-time Invoice Processing & Logging:** From receiving invoice photos in Telegram, extracting and parsing invoice data with AI, splitting multiple items, storing them in Notion, and sending back a summary message to Telegram.

- **1.2 Scheduled Financial Reporting:** Periodically fetching recent expense data from Notion, summarizing spending by category, generating a visual chart, and sending the report as an image to a Telegram chat or group.

---

### 2. Block-by-Block Analysis

#### 2.1 Real-time Invoice Processing & Logging

**Overview:**  
This block listens for invoice photos sent to a Telegram bot, downloads the image, sends it to Google Gemini AI to extract structured transaction data, parses the AI output, splits multi-item invoices, records each transaction into a Notion database, and sends a summary message back to the Telegram chat.

**Nodes Involved:**  
- Telegram Trigger | When recive photo  
- Telegram (Get file)  
- Get Image Info  
- Google Gemini Chat Model  
- Basic LLM Chain  
- Parse To your object | Table  
- Split Out | data transaction  
- Record To Notion Database  
- Sendback to chat and give summarize text  

**Node Details:**

- **Telegram Trigger | When recive photo**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point; triggers workflow on receiving any message with a photo in Telegram chat.  
  - *Config:* Monitors "message" updates; requires Telegram API credentials with bot token.  
  - *Connections:* Outputs to Telegram node (to fetch photo file).  
  - *Edge cases:* Missing photo in message, API token invalid, Telegram rate limits.

- **Telegram (Get file)**  
  - *Type:* Telegram node  
  - *Role:* Downloads the highest resolution photo from the received message.  
  - *Config:* Uses the largest photo's `file_id` (index 3) from incoming message data; Telegram API credentials required.  
  - *Connections:* Outputs to Get Image Info node.  
  - *Edge cases:* File missing or deleted from Telegram server, network errors.

- **Get Image Info**  
  - *Type:* Edit Image node  
  - *Role:* Retrieves metadata about the downloaded image; prepares it for AI processing.  
  - *Config:* Operation set to "information" to extract image details.  
  - *Connections:* Outputs to Google Gemini Chat Model.  
  - *Edge cases:* Corrupted image file, unsupported formats.

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini Chat Model node  
  - *Role:* Sends image data to Google Gemini API for OCR and AI-driven data extraction.  
  - *Config:* Uses model "models/gemini-2.5-flash-preview-04-17"; Google PaLM API credentials required.  
  - *Connections:* Sends AI output to Basic LLM Chain (as language model input).  
  - *Edge cases:* API authentication errors, request timeouts, malformed image data.

- **Basic LLM Chain**  
  - *Type:* Langchain LLM Chain node  
  - *Role:* Applies a custom prompt instructing the AI to extract structured invoice data and produce a JSON summary message.  
  - *Config:*  
    - Prompt expects invoice details: date, id, name, qty, price, total, category, tax.  
    - Categories fixed to Food & Beverage, Transportation, Utilities, Shopping, Healthcare, Entertainment, Housing, Education.  
    - Output is a JSON array with a summary message.  
  - *Connections:* Outputs to Split Out and Sendback to chat. Also receives parsed AI output.  
  - *Edge cases:* Output parsing errors if AI response deviates, prompt misconfiguration.

- **Parse To your object | Table**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Converts AI’s raw JSON text output into structured JSON object for downstream node consumption.  
  - *Config:* Manual JSON schema defines fields: message (string), summary (array of objects with date, id, name, qty, price, tax, total, category).  
  - *Connections:* Output feeds into Basic LLM Chain node as parsed data.  
  - *Edge cases:* AI output deviates from schema, parsing failures.

- **Split Out | data transaction**  
  - *Type:* SplitOut node  
  - *Role:* Splits multiple transaction items from the AI summary array into individual workflow items for separate processing.  
  - *Config:* Splits on field `output.summary`.  
  - *Connections:* Outputs individual transactions to Record To Notion Database.  
  - *Edge cases:* Empty or malformed array, splitting errors.

- **Record To Notion Database**  
  - *Type:* Notion node  
  - *Role:* Creates a new page in the configured Notion database for each transaction item.  
  - *Config:*  
    - Database ID specified.  
    - Maps extracted fields to Notion properties (Name, Quantity, Price, Total, Category, Date, Tax).  
    - Notion API credentials required.  
  - *Connections:* Terminal node for each transaction.  
  - *Edge cases:* API rate limits, invalid database ID, permission errors.

- **Sendback to chat and give summarize text**  
  - *Type:* Telegram node  
  - *Role:* Sends back a summary message generated by AI to the original Telegram chat for user confirmation.  
  - *Config:* Uses chat ID from the Telegram Trigger node; disables attribution.  
  - *Connections:* Terminal node for summary message.  
  - *Edge cases:* Invalid chat ID, Telegram send message errors.

---

#### 2.2 Scheduled Financial Reporting

**Overview:**  
This block periodically triggers, fetches recent expense data from Notion, summarizes spending by category, formats the data for charting, generates a bar chart image, and sends it to a Telegram chat or group.

**Nodes Involved:**  
- Schedule Trigger | for send chart report  
- Get Recent Data from Notions  
- Summarize Transaction Data  
- Convert Data to JSON chart payload  
- Generate Chart  
- Send Chart Image to Group or Private Chat  

**Node Details:**

- **Schedule Trigger | for send chart report**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the reporting process on a fixed interval (default: every week).  
  - *Config:* Interval set to run periodically; configurable per user need.  
  - *Connections:* Outputs to Get Recent Data from Notions.  
  - *Edge cases:* Misconfiguration may cause no triggers or too frequent firing.

- **Get Recent Data from Notions**  
  - *Type:* Notion node  
  - *Role:* Retrieves all pages from the Notion database created in the past week (or configured filter period).  
  - *Config:* Filter by Created time with condition "past_week"; returns all matching pages.  
  - *Connections:* Outputs to Summarize Transaction Data.  
  - *Edge cases:* No data found (empty result), API issues, database permission errors.

- **Summarize Transaction Data**  
  - *Type:* Summarize node  
  - *Role:* Aggregates total expenses grouped by the Notion property `category`.  
  - *Config:* Splits by `property_category`, sums `property_total`.  
  - *Connections:* Outputs to Convert Data to JSON chart payload.  
  - *Edge cases:* Missing or improperly typed data fields.

- **Convert Data to JSON chart payload**  
  - *Type:* Code node  
  - *Role:* Formats the summarized data into a JSON payload compatible with QuickChart for bar chart generation.  
  - *Config:* JavaScript code extracts categories and sums, builds `labels` and `datasets` arrays with styling options.  
  - *Connections:* Outputs formatted JSON to Generate Chart node.  
  - *Edge cases:* Empty input data, JS code errors.

- **Generate Chart**  
  - *Type:* QuickChart node  
  - *Role:* Creates chart image (bar chart) from JSON data payload.  
  - *Config:* Uses chart data and labels from previous node; can be customized for chart type and options.  
  - *Connections:* Outputs image to Telegram send node.  
  - *Edge cases:* QuickChart API limits, malformed chart data.

- **Send Chart Image to Group or Private Chat**  
  - *Type:* Telegram node  
  - *Role:* Sends the generated chart image to a specified Telegram chat or group.  
  - *Config:*  
    - Chat ID set to target Telegram chat/group.  
    - Sends as photo with filename "chart".  
    - Telegram API credentials required.  
  - *Connections:* Terminal node for report delivery.  
  - *Edge cases:* Invalid chat ID, Telegram API errors.

---

### 3. Summary Table

| Node Name                          | Node Type                                   | Functional Role                         | Input Node(s)                         | Output Node(s)                             | Sticky Note                                                                                          |
|-----------------------------------|---------------------------------------------|---------------------------------------|-------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger | When recive photo | Telegram Trigger                            | Entry trigger for invoice photo       | None                                | Telegram                                  | 📸 INVOICE INPUT 📸 Bot listens here for photos of your receipts/invoices. Ensure your Telegram Bot API token is set in credentials. |
| Telegram                          | Telegram node                               | Downloads photo file from Telegram    | Telegram Trigger                    | Get Image Info                            |                                                                                                    |
| Get Image Info                   | Edit Image                                  | Extracts image metadata                | Telegram                           | Google Gemini Chat Model                   |                                                                                                    |
| Google Gemini Chat Model          | Langchain Google Gemini Chat Model          | AI OCR & data extraction               | Get Image Info                     | Basic LLM Chain                           | 🤖 AI MAGIC HAPPENS HERE 🧠 - Image is sent to Google Gemini for data extraction. - Check 'Basic LLM Chain' to customize the AI prompt (e.g., categories, output format). - Requires Google Gemini API credentials. |
| Basic LLM Chain                  | Langchain LLM Chain                          | Applies AI prompt & parses response   | Google Gemini Chat Model, Parse To your object | Split Out | data transaction, Sendback to chat and give summarize text |                                                                                                    |
| Parse To your object | Table       | Langchain Output Parser Structured           | Parses AI JSON output to structured format | None (feeds Basic LLM Chain input)  | Basic LLM Chain                           | ✨ STRUCTURING AI DATA ✨ Converts the AI's text output into a usable JSON object. Check the schema if you modify the AI prompt significantly. |
| Split Out | data transaction       | SplitOut                                    | Splits multiple transactions          | Basic LLM Chain                    | Record To Notion Database                  |                                                                                                    |
| Record To Notion Database         | Notion                                      | Saves transaction to Notion DB        | Split Out                        | None                                       | 📝 SAVING TO NOTION 📝 - Extracted transaction data is saved here. - Configure with your Notion API key & Database ID. - Map fields correctly to your database columns! |
| Sendback to chat and give summarize text | Telegram                                    | Sends summary message back to Telegram chat | Basic LLM Chain                    | None                                       | 💬 TRANSACTION SUMMARY 💬 Sends a confirmation message back to the user in Telegram with a summary of the recorded expense. |
| Schedule Trigger | for send chart report | Schedule Trigger                            | Starts scheduled reporting            | Get Recent Data from Notions      | None                                       | 🗓️ REPORTING SCHEDULE 🗓️ Set how often you want to receive your spending report (e.g., weekly, monthly). |
| Get Recent Data from Notions      | Notion                                      | Retrieves recent transactions         | Schedule Trigger                  | Summarize Transaction Data                 | 📊 FETCHING DATA FOR REPORT 📊 - Retrieves transactions from Notion for the report period. - Default: "Past Week". Adjust filter as needed. - Requires Notion API credentials & Database ID. |
| Summarize Transaction Data        | Summarize                                   | Aggregates total spending by category | Get Recent Data from Notions      | Convert Data to JSON chart payload          | ➕ SUMMARIZING SPENDING ➕ Aggregates your expenses, usually by category, to prepare for the chart. |
| Convert Data to JSON chart payload| Code                                        | Formats data for QuickChart            | Summarize Transaction Data       | Generate Chart                             | 🎨 PREPARING CHART DATA 🎨 This Code node formats the summarized data into the JSON structure needed by QuickChart. |
| Generate Chart                   | QuickChart                                  | Generates chart image                  | Convert Data to JSON chart payload| Send Chart Image to Group or Private Chat  | 📈 GENERATING VISUAL REPORT 📈 Creates the actual chart image based on your spending data. You can customize chart type (bar, pie, etc.) here. |
| Send Chart Image to Group or Private Chat | Telegram                                    | Sends chart report image to Telegram  | Generate Chart                   | None                                       | 📤 SENDING REPORT TO TELEGRAM 📤 - Delivers the generated chart to your chosen Telegram chat/group. - Set the correct Chat ID and Bot API token. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure: Use your Telegram Bot API credentials. Set to trigger on "message" updates.  
   - Purpose: Trigger when a photo message is received.

2. **Create Telegram node (Get file)**  
   - Type: Telegram  
   - Configure: Use same Telegram credentials. Set operation to "get file" with fileId from the largest photo in the incoming message (`{{$json.message.photo[3].file_id}}`).  
   - Connect from Telegram Trigger.

3. **Create Get Image Info node**  
   - Type: Edit Image  
   - Configure: Operation "information" to retrieve metadata about the downloaded photo.  
   - Connect from Telegram node.

4. **Create Google Gemini Chat Model node**  
   - Type: Langchain Google Gemini Chat Model  
   - Configure: Select model `models/gemini-2.5-flash-preview-04-17`. Provide Google Gemini (PaLM) API credentials.  
   - Connect from Get Image Info node.

5. **Create Basic LLM Chain node**  
   - Type: Langchain LLM Chain  
   - Configure:  
     - Set prompt to instruct AI to extract invoice data as JSON with fields: date, id, name, qty, price, total, category, tax, and provide a summary message.  
     - Categories fixed: Food & Beverage, Transportation, Utilities, Shopping, Healthcare, Entertainment, Housing, Education.  
     - Use an Output Parser for structured JSON.  
   - Connect `ai_languageModel` input from Google Gemini Chat Model.  
   - Connect output to Split Out node and Telegram send summary.

6. **Create Parse To your object | Table node**  
   - Type: Langchain Output Parser Structured  
   - Configure: Define JSON schema matching AI output (message string and summary array with required fields).  
   - Connect output parser input to Basic LLM Chain node.

7. **Create Split Out | data transaction node**  
   - Type: SplitOut  
   - Configure: Field to split out: `output.summary` (array of transactions).  
   - Connect from Basic LLM Chain node.

8. **Create Record To Notion Database node**  
   - Type: Notion  
   - Configure:  
     - Use Notion API credentials.  
     - Select your Notion database ID.  
     - Map fields from split transaction items to Notion properties:  
       - Name → title  
       - qty → Quantity (number)  
       - price → Price (number)  
       - total → Total (number)  
       - category → Category (select)  
       - date → Date (rich_text)  
       - tax → Tax (number)  
   - Connect from Split Out node.

9. **Create Sendback to chat and give summarize text node**  
   - Type: Telegram  
   - Configure:  
     - Use Telegram credentials.  
     - Send message text from AI output: `={{ $json.output.message }}`  
     - Chat ID from Telegram Trigger message chat id: `={{ $('Telegram Trigger | When recive photo').item.json.message.chat.id }}`  
     - Disable attribution.  
   - Connect from Basic LLM Chain node (parallel to Split Out).

10. **Create Schedule Trigger | for send chart report node**  
    - Type: Schedule Trigger  
    - Configure: Set interval (e.g., weekly on Monday 9 AM).  
    - Purpose: Periodic reporting.

11. **Create Get Recent Data from Notions node**  
    - Type: Notion  
    - Configure:  
      - Use same Notion credentials and database as above.  
      - Filter by Created time with condition "past_week".  
      - Return all entries.  
    - Connect from Schedule Trigger.

12. **Create Summarize Transaction Data node**  
    - Type: Summarize  
    - Configure:  
      - Split by `property_category`.  
      - Sum `property_total`.  
    - Connect from Get Recent Data node.

13. **Create Convert Data to JSON chart payload node**  
    - Type: Code  
    - Configure: JavaScript code to build chart JSON from summarized data:  
      - Extract categories and sums into labels and data arrays.  
      - Build JSON for QuickChart bar chart with labels, datasets, and styling.  
    - Connect from Summarize node.

14. **Create Generate Chart node**  
    - Type: QuickChart  
    - Configure:  
      - Use labels and data from previous node.  
      - Chart type: bar (default).  
    - Connect from Code node.

15. **Create Send Chart Image to Group or Private Chat node**  
    - Type: Telegram  
    - Configure:  
      - Use Telegram credentials.  
      - Set `chatId` to target Telegram chat/group for reports.  
      - Operation: sendPhoto with binary data.  
      - Filename: "chart".  
    - Connect from Generate Chart node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| 🔑 Credentials needed: Telegram Bot API token, Google Gemini (PaLM) API key, Notion API key with database access.                                                                | Setup instructions section                          |
| 💡 Customize AI prompts in 'Basic LLM Chain' node for better extraction accuracy, modify Notion database structure or categories, adjust report frequency and content.            | Workflow flexibility                                |
| 📸 Invoice input: Telegram bot listens for photos of receipts; ensure bot token and chat ID are correctly set to capture invoices.                                              | Sticky Note on Telegram Trigger node                |
| 🤖 AI magic: Google Gemini processes image and extracts structured data; review and adjust prompt for your invoice formats and category needs.                                   | Sticky Notes near Gemini and Basic LLM Chain nodes  |
| 📝 Saving to Notion: Map extracted data fields correctly to Notion database properties and ensure integration permission granted to the bot.                                     | Sticky Note on Notion node                           |
| 💬 Transaction summaries: Confirmation messages sent back to Telegram user for transparency and verification.                                                                     | Sticky Note on summary send node                     |
| 🗓️ Reporting schedule: Configure schedule trigger for periodic reports, default weekly but customizable.                                                                         | Sticky Note on Schedule Trigger                      |
| 📊 Data fetching and summarizing: Notion data queried and aggregated by category for visual representation.                                                                       | Sticky Notes on Notion and Summarize nodes          |
| 🎨 Chart generation: Chart data formatted and QuickChart used to generate bar charts; customize chart style in code node or QuickChart node.                                      | Sticky Notes on Code and QuickChart nodes            |
| 📤 Report delivery: Generated charts sent to Telegram group or private chat; ensure chat ID is correct and bot has access.                                                        | Sticky Note on report send node                      |
| Workflow tested for n8n version 1.x+ with Langchain nodes and standard nodes; ensure all credentials and access rights are active and valid to avoid runtime errors.              | Version note                                        |

---

This detailed documentation provides a complete understanding of the workflow’s structure, logic, nodes, configurations, and reproduction steps. It also anticipates common errors and provides customization points to adapt to your financial tracking needs.