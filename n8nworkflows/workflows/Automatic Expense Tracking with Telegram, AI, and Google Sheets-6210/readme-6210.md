Automatic Expense Tracking with Telegram, AI, and Google Sheets

https://n8nworkflows.xyz/workflows/automatic-expense-tracking-with-telegram--ai--and-google-sheets-6210


# Automatic Expense Tracking with Telegram, AI, and Google Sheets

### 1. Workflow Overview

This workflow automates personal expense tracking by integrating Telegram messaging, AI-based natural language processing, currency exchange rate retrieval, and Google Sheets logging. It targets users who record expenses via Telegram messages in free text, which the system parses, categorizes, converts to a base currency (EGP), and stores in a structured spreadsheet.

The logic is divided into these main blocks:

- **1.1 Input Reception:** Captures user expense messages from a specific Telegram chat.
- **1.2 AI Processing:** Uses a LangChain AI Agent with OpenAI and Anthropic models to parse natural language expense messages into structured JSON, including currency, category, and payment method extraction.
- **1.3 Exchange Rate Handling:** Conditionally fetches currency exchange rates for USD, SAR, and AED to convert amounts into EGP.
- **1.4 Output Parsing and Data Formatting:** Validates and formats AI output according to a strict JSON schema and adjusts date formatting for Google Sheets compatibility.
- **1.5 Data Storage:** Appends the cleaned and structured expense record as a new row in a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives incoming messages from a predefined Telegram chat, triggering the workflow on each new expense message.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**  
- **Telegram Trigger**  
  - Type: Trigger node listening to Telegram message updates  
  - Configuration: Listens for "message" updates only; restricts to a specific chat ID via environment variable `chat_id`; downloads message content if any media attached  
  - Inputs: None (trigger)  
  - Outputs: JSON containing message text, user ID, timestamp, etc.  
  - Potential Failures: Telegram API connectivity issues, invalid credentials, missing or incorrect chat ID environment variable  
  - Credential: Uses Telegram API credentials (OAuth token linked to Telegram bot)

---

#### 2.2 AI Processing

**Overview:**  
Processes the raw Telegram message text with an AI agent that parses natural language into a structured expense record. The AI extracts amount, currency, category, payment method, and applies business logic for currency conversion.

**Nodes Involved:**  
- AI Agent  
- o3 (OpenAI Model)  
- 2.5F (Google Gemini Model)  
- H3.5 (Anthropic Claude Model)  
- 2.5F1 (Google Gemini Model - secondary instance)  
- Get Rates (HTTP request for currency exchange rates)  
- Parser (Structured output parser node)

**Node Details:**  
- **AI Agent**  
  - Type: LangChain Agent node  
  - Configuration:  
    - Text prompt instructs the agent to parse the expense message extracting all required fields strictly as JSON with no explanations.  
    - Includes complex logic: default currency to EGP, only calls `Get Rates` tool if currency is USD, SAR, or AED, otherwise sets exchange rate to 1.  
    - Category and payment method mappings are provided as hints inside the prompt.  
    - Uses a fallback option for robustness.  
  - Inputs: JSON from Telegram Trigger with message text, user ID, and timestamp  
  - Outputs: JSON expense record (amount, category, description, date, user_id, payment_method, currency, exchange_rate, amount_converted)  
  - Connections:  
    - Calls multiple AI language models (o3, 2.5F, H3.5) as language model options  
    - Uses `Get Rates` as an AI tool for exchange rate retrieval  
    - Output parsed by `Parser` node  
  - Edge Cases:  
    - AI parsing errors or invalid JSON output  
    - Missing or ambiguous currency or payment method in message  
    - API rate limits or failures for OpenAI, Anthropic, or Google Gemini  
    - HTTP request failure in `Get Rates`  
  - Credentials: OpenAI, Anthropic, Google Gemini API keys configured

- **o3 / 2.5F / H3.5 / 2.5F1**  
  - Type: AI language model nodes (OpenAI GPT, Google Gemini, Anthropic Claude)  
  - Role: Provide alternative LLM backends for the AI Agent node to call, allowing fallback or model choice flexibility  
  - Inputs: Controlled by AI Agent node, no direct triggering  
  - Outputs: AI-generated text used by AI Agent  
  - Edge Cases: API connectivity, rate limits, model availability

- **Get Rates**  
  - Type: HTTP Request Tool node  
  - Role: Retrieves latest exchange rates for EGP from an open API (`https://open.er-api.com/v6/latest/EGP`)  
  - Triggered only if the detected currency is USD, SAR, or AED  
  - Inputs: Called as AI Agent tool  
  - Outputs: JSON exchange rates to be used for conversion  
  - Failure modes: HTTP errors, API downtime, malformed responses

- **Parser**  
  - Type: LangChain Output Parser node  
  - Role: Validates and structurally parses the AI Agent’s JSON output according to a detailed JSON schema defining types, enums, required properties  
  - AutoFix enabled to correct minor format issues automatically  
  - Inputs: AI Agent output text  
  - Outputs: Structured JSON object ready for further processing  
  - Failures: Parsing failures if AI output deviates significantly from schema

---

#### 2.3 Output Formatting and Data Preparation

**Overview:**  
Transforms the validated JSON output from the AI Agent to align with Google Sheets date format and ensures all fields have appropriate types and fallback defaults.

**Nodes Involved:**  
- Code

**Node Details:**  
- **Code**  
  - Type: JavaScript code execution node  
  - Function:  
    - Receives AI Agent parsed output JSON  
    - Converts ISO 8601 date string to a Google Sheets friendly datetime string format `YYYY-MM-DD HH:MM:SS`  
    - Ensures numeric fields are numbers, string fields have defaults if missing  
  - Inputs: Structured JSON from Parser node  
  - Outputs: JSON with normalized and type-safe expense data  
  - Edge Cases: Invalid or missing date strings, non-numeric amount fields, missing payment method or category  
  - Failure: JavaScript exceptions if unexpected data types appear

---

#### 2.4 Data Storage

**Overview:**  
Appends the cleaned, structured expense data as a new row into a designated Google Sheets spreadsheet.

**Nodes Involved:**  
- Append row in sheet

**Node Details:**  
- **Append row in sheet**  
  - Type: Google Sheets node  
  - Operation: Append new row to sheet with specific columns mapped from expense data fields (date, amount, user_id, category, currency, description, exchange_rate, payment_method, amount_converted)  
  - Sheet details:  
    - Document ID linked to a specific Google Sheet (expense tracker)  
    - Sheet name is the first sheet (`gid=0`)  
  - Inputs: JSON from Code node with formatted data  
  - Outputs: Confirmation of row insertion  
  - Credential: Google Sheets OAuth2 account with write access  
  - Failure modes: Authentication errors, quota limits, spreadsheet not found, schema mismatches in column mapping

---

### 3. Summary Table

| Node Name          | Node Type                                | Functional Role                        | Input Node(s)         | Output Node(s)          | Sticky Note                                  |
|--------------------|----------------------------------------|-------------------------------------|-----------------------|-------------------------|----------------------------------------------|
| Telegram Trigger    | Telegram Trigger                       | Receives Telegram messages           | None                  | AI Agent                |                                              |
| AI Agent           | LangChain AI Agent                    | Parses natural language expense data | Telegram Trigger, o3, 2.5F, H3.5, Get Rates, Parser | Code                    |                                              |
| o3                 | LangChain LM Chat OpenAI              | OpenAI model backend for AI Agent   | AI Agent (ai_languageModel) | AI Agent                |                                              |
| 2.5F               | LangChain LM Chat Google Gemini       | Google Gemini model backend          | AI Agent (ai_languageModel) | AI Agent                |                                              |
| H3.5               | LangChain LM Chat Anthropic           | Anthropic Claude model backend       | AI Agent (ai_languageModel) | None                    |                                              |
| 2.5F1              | LangChain LM Chat Google Gemini       | Secondary Gemini model for parsing   | None                  | Parser                  |                                              |
| Get Rates          | HTTP Request Tool                     | Retrieves currency exchange rates    | AI Agent (ai_tool)     | AI Agent                |                                              |
| Parser             | LangChain Output Parser Structured    | Validates and parses AI JSON output  | 2.5F1                  | AI Agent                |                                              |
| Code               | Code                                  | Formats data for Google Sheets       | AI Agent               | Append row in sheet     |                                              |
| Append row in sheet| Google Sheets                         | Appends expense record to sheet      | Code                   | None                    |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates only  
   - Additional Fields: Set `chatIds` to environment variable `{{$env.chat_id}}` to restrict to specific chat  
   - Credentials: Connect Telegram API credentials (Telegram Bot token)  
   - Position: Leftmost start of workflow

2. **Create AI Agent Node:**  
   - Type: LangChain Agent node  
   - Parameters:  
     - Text prompt: Copy the detailed multi-step instructions to parse the expense message into JSON, including category and payment method hints, currency and exchange rate logic  
     - Enable fallback for robustness  
     - Set Output Parser to use the "Parser" node (to be created)  
   - Configure language models as options: Add nodes o3, 2.5F, H3.5 as AI language models  
   - Add `Get Rates` node as AI tool for exchange rate retrieval  
   - Connect Telegram Trigger to AI Agent main input

3. **Create AI Language Model Nodes:**
   - **o3:**  
     - Type: LangChain LM Chat OpenAI  
     - Parameters: Model set to "o3"  
     - Credentials: Link OpenAI API credentials  
   - **2.5F:**  
     - Type: LangChain LM Chat Google Gemini  
     - Credentials: Link Google Gemini API Key  
   - **H3.5:**  
     - Type: LangChain LM Chat Anthropic  
     - Credentials: Link Anthropic API Key

4. **Create Get Rates Node:**  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://open.er-api.com/v6/latest/EGP`  
     - Method: GET  
     - No authentication or additional headers needed  
   - Position: Connected as AI tool in AI Agent node

5. **Create Parser Node:**  
   - Type: LangChain Output Parser Structured  
   - Parameters:  
     - Enable AutoFix  
     - Input schema: Copy the detailed JSON schema for expense fields with types, enums, and required properties  
   - Connect output of AI Agent or 2.5F1 node to this Parser

6. **Create 2.5F1 Node:**  
   - Type: LangChain LM Chat Google Gemini  
   - Purpose: Secondary Gemini model instance for parsing  
   - Credentials: Same as 2.5F  
   - Connect output to Parser node

7. **Connect AI Agent to Code Node:**  
   - Create Code node (JavaScript)  
   - Paste the provided code to format ISO date to Google Sheets datetime, ensure numeric and string fields with defaults  
   - Connect AI Agent main output to Code node input

8. **Create Append row in sheet Node:**  
   - Type: Google Sheets node  
   - Operation: Append  
   - Document ID: Set to your Google Sheets document ID (e.g., `1v5ffTb0q-kS4yE6ItklO-L1CX3qKVgIT1VlGK2pEU6s`)  
   - Sheet Name: Set to the first sheet or specific sheet name (e.g., `gid=0`)  
   - Columns: Define columns mapping from Code output JSON fields (date, amount, user_id, category, currency, description, exchange_rate, payment_method, amount_converted)  
   - Credentials: Connect Google Sheets OAuth2 credentials with write access  
   - Connect Code node to this node

9. **Final Connections:**  
   - Telegram Trigger → AI Agent → Code → Append row in sheet  
   - AI Agent uses o3, 2.5F, H3.5 as language models and Get Rates as AI tool  
   - 2.5F1 → Parser → AI Agent output parser

10. **Set Environment Variable:**  
    - Define environment variable `chat_id` with the Telegram chat ID to restrict message triggers

11. **Activate Workflow and Test:**  
    - Deploy and activate workflow  
    - Send test expense messages in the specified Telegram chat  
    - Verify parsed data appears correctly in Google Sheets

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses advanced LangChain AI integration allowing multiple LLM backends for fallback | LangChain documentation: https://js.langchain.com/docs/                                       |
| Exchange rates are fetched from a free public API: https://open.er-api.com/                      | Useful for currency conversion, but consider API rate limits                                  |
| Google Sheets date format conversion is critical for correct spreadsheet display                  | Google Sheets date/time format guide: https://support.google.com/docs/answer/6055612           |
| Telegram Bot API restricts triggers to specific chats via chat ID environment variable            | Telegram Bot API docs: https://core.telegram.org/bots/api#available-methods                    |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.