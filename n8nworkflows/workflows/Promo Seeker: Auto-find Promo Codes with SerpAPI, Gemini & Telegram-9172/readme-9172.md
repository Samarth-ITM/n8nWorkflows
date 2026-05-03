Promo Seeker: Auto-find Promo Codes with SerpAPI, Gemini & Telegram

https://n8nworkflows.xyz/workflows/promo-seeker--auto-find-promo-codes-with-serpapi--gemini---telegram-9172


# Promo Seeker: Auto-find Promo Codes with SerpAPI, Gemini & Telegram

### 1. Workflow Overview

This workflow, titled **"Promo Seeker: Auto-find Promo Codes with SerpAPI, Gemini & Telegram"**, is designed to automatically find, verify, and distribute valid promotional codes and discount vouchers from the internet. It targets users who want to easily discover and receive up-to-date promo codes for various platforms via Telegram or email.

The workflow integrates multiple functional blocks to achieve this:

- **1.1 Input Reception**: Accepts requests via Telegram messages, webhook POST requests, or form submissions to specify the platform or product category for which promos are sought.
- **1.2 Platform Data Handling**: Parses input to extract platform queries and receiver identifiers (chat IDs or emails).
- **1.3 Promo Code Lookup**: Checks an internal data table for existing promo codes matching the platform query.
- **1.4 Promo Code Existence Decision**: Decides if promo codes exist for the platform and acts accordingly.
- **1.5 AI-Powered Promo Search**: Uses AI agents powered by LangChain with SerpAPI and Google Gemini 2.5 Pro to search the web for new promo codes, applying filtering and formatting logic.
- **1.6 Promo Code Storage**: Updates or inserts new promo codes into an internal data table for future reference.
- **1.7 User Notification**: Sends promo codes to users via Telegram messages and Gmail emails in rich, formatted content.
- **1.8 Auxiliary and Maintenance Nodes**: Includes scheduling, no-operation placeholders, and sticky notes for documentation and credentials instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles receiving user requests for promo codes through Telegram, webhook POST requests, and form submissions.

**Nodes Involved:**  
- Telegram Trigger  
- Webhook  
- On form submission

**Node Details:**  
- **Telegram Trigger**  
  - Type: Trigger node for Telegram updates  
  - Config: Monitors incoming messages (`updates: message`)  
  - Credentials: Telegram Bot Token (named "Khaisa 2 bot")  
  - Input: Telegram chat messages  
  - Output: Message JSON with chat ID and text  
  - Edge Cases: Bot token invalid/expired, Telegram API limits, malformed messages

- **Webhook**  
  - Type: HTTP POST webhook trigger  
  - Config: Path `/v1/promo-seeker`, responds via Respond to Webhook node  
  - Input: JSON POST body with `platform` and/or other parameters  
  - Output: Incoming request data to downstream nodes  
  - Edge Cases: Missing required fields, invalid JSON, network issues

- **On form submission**  
  - Type: Form trigger node  
  - Config: Form titled "Promo Seeker" with fields "Platform" (required) and "email" (required)  
  - Input: User-submitted form data  
  - Output: Form data JSON  
  - Edge Cases: User cancels submission, missing required fields

---

#### 2.2 Platform Data Handling

**Overview:**  
Extracts and sets the platform query term and receiver information from incoming requests for use downstream.

**Nodes Involved:**  
- No Operation, do nothing (used as a placeholder)  
- Platform (Set node)

**Node Details:**  
- **No Operation, do nothing**  
  - Type: NoOp node, passes data unchanged  
  - Role: Used to separate flows or hold place for future logic  
  - Edge Cases: None (no processing)

- **Platform**  
  - Type: Set node  
  - Config: Assigns two variables:  
    - `query`: Extracts platform name from input fields (`Platform`, `message.text`, `body.platform`)  
    - `receiver`: Extracts receiver info (`chatId`, `email`, `body.email`)  
  - Input: Incoming request data (from Telegram, webhook, or form)  
  - Output: JSON with `query` and `receiver` fields  
  - Edge Cases: Missing platform or receiver fields; empty strings handled safely

---

#### 2.3 Promo Code Lookup

**Overview:**  
Checks the internal Data Table "Kode Promo Valid" for existing promo codes matching the requested platform.

**Nodes Involved:**  
- Get row(s)

**Node Details:**  
- **Get row(s)**  
  - Type: Data Table node (get operation)  
  - Config: Filter rows where `platform` equals the `query` value; limits to 3 rows  
  - Input: JSON with platform query  
  - Output: Matching promo code entries from the data table  
  - Edge Cases: No matching rows found, data table unavailability

---

#### 2.4 Promo Code Existence Decision

**Overview:**  
Determines whether promo codes exist for the requested platform and chooses the next action path.

**Nodes Involved:**  
- Code Exist? (If node)

**Node Details:**  
- **Code Exist?**  
  - Type: If node  
  - Config: Condition checks if the `platform` field exists (non-empty) in the promo code results  
  - Input: Rows from Get row(s) node  
  - Output:  
    - True branch: Promo codes exist  
    - False branch: No promo codes found  
  - Edge Cases: Missing or malformed data fields that could cause condition evaluation errors

---

#### 2.5 AI-Powered Promo Search

**Overview:**  
When no existing promo codes are found, this block uses AI agents to search for new valid promo codes online and parse structured results.

**Nodes Involved:**  
- Promo Seeker Agent (LangChain Agent)  
- SerpAPI (LangChain tool)  
- Gemini 2.5Pro (LangChain language model)  
- Structured Output Parser

**Node Details:**  
- **Promo Seeker Agent**  
  - Type: LangChain agent node  
  - Config: Uses a defined system message instructing the agent to find current, valid promo codes posted within the last 30 days, focusing on reputable sources and structured responses (fields like code, value, expiry, source)  
  - Input: Query string `platform: {{ platform }}`  
  - Output: AI-generated promo code data  
  - Integration: Receives AI tool input from SerpAPI and language model input from Gemini 2.5 Pro  
  - Edge Cases: API limits, no response, malformed AI output, outdated data

- **SerpAPI**  
  - Type: LangChain SerpAPI tool node  
  - Config: Uses SerpAPI to perform web searches supporting the Promo Seeker Agent  
  - Credentials: User's SerpAPI API key via GitHub login  
  - Output: Search results passed to Promo Seeker Agent  
  - Edge Cases: API key invalid, quota exceeded, network errors

- **Gemini 2.5Pro**  
  - Type: LangChain language model node (OpenRouter integration)  
  - Config: Uses Google Gemini 2.5 Pro model for natural language understanding and generation  
  - Credentials: OpenRouter API key  
  - Output: Language model completions sent to Promo Seeker Agent  
  - Edge Cases: API errors, rate limits, malformed prompts

- **Structured Output Parser**  
  - Type: LangChain output parser node  
  - Config: Parses AI responses into a structured JSON schema with fields: `platform`, `code1`, `value`, `terms`, `validUntil`  
  - Input: AI output text  
  - Output: Parsed JSON for downstream use  
  - Edge Cases: Parsing failure due to unexpected AI output format

---

#### 2.6 Promo Code Storage

**Overview:**  
Updates or inserts newly found promo codes into the internal Data Table for later retrieval.

**Nodes Involved:**  
- Upsert row(s)

**Node Details:**  
- **Upsert row(s)**  
  - Type: Data Table node (upsert operation)  
  - Config: Maps parsed promo code fields to data table columns (`platform`, `promoCode`, `value`, `termsConditions`, `validUntil`); matches existing rows by `promoCode` to update or inserts new rows  
  - Input: Parsed AI output JSON from Promo Seeker Agent  
  - Output: Confirmation of data table update  
  - Edge Cases: Data table unavailability, duplicate promo code conflicts

---

#### 2.7 User Notification

**Overview:**  
Notifies the user of promo code results via Telegram messages and Gmail emails, using rich formatting to enhance readability.

**Nodes Involved:**  
- notify telegram  
- Send a message (Gmail node)  
- Respond to Webhook (disabled)

**Node Details:**  
- **notify telegram**  
  - Type: Telegram node (send message)  
  - Config: Sends a formatted HTML message with promo details to the Telegram chat ID extracted from incoming messages  
  - Input: Promo code JSON data  
  - Credentials: Telegram Bot Token  
  - Edge Cases: Telegram API errors, invalid chat ID, message formatting issues

- **Send a message (Gmail)**  
  - Type: Gmail node (send email)  
  - Config: Sends an HTML email with promo code details to the email address extracted from webhook or form submission  
  - Input: Promo code JSON data with HTML template including platform, code, value, terms, validity date  
  - Credentials: Gmail OAuth2 credential  
  - Edge Cases: Gmail API authorization errors, invalid email addresses, email sending failures

- **Respond to Webhook** (disabled)  
  - Type: Respond to Webhook node  
  - Config: Designed to respond to webhook requests but currently disabled  
  - Edge Cases: No response sent for webhook POSTs unless enabled

---

#### 2.8 Auxiliary and Documentation Nodes

**Overview:**  
Provides scheduling for possible future automation, placeholders for additional logic, and sticky notes for user guidance.

**Nodes Involved:**  
- Schedule Trigger  
- Sticky Note  
- Sticky Note1  

**Node Details:**  
- **Schedule Trigger**  
  - Type: Schedule trigger  
  - Config: Runs on default interval (every minute/hour by default), currently connected to No Operation node  
  - Edge Cases: None functional currently (used for potential scheduled runs)

- **Sticky Note**  
  - Type: Documentation node  
  - Content: Detailed instructions on obtaining credentials for SerpAPI, OpenRouter API (Gemini), Telegram Bot, Gmail OAuth2, and Data Table schema requirements  
  - Purpose: Helps users set up necessary credentials securely

- **Sticky Note1**  
  - Type: Documentation node  
  - Content: Overview of how the workflow works, explaining its purpose and user interaction  
  - Purpose: Provides context and usage instructions for users and maintainers

---

### 3. Summary Table

| Node Name              | Node Type                              | Functional Role                          | Input Node(s)          | Output Node(s)                   | Sticky Note                                                                                                         |
|------------------------|--------------------------------------|----------------------------------------|-----------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger                     | Receives promo search requests via Telegram | None                  | No Operation, do nothing         | See Sticky Note for credentials setup: https://t.me/botfather for bot creation                                      |
| Webhook                | Webhook                             | Receives promo search requests via HTTP POST | None                  | No Operation, do nothing         | -                                                                                                                   |
| On form submission     | Form Trigger                       | Receives promo search requests via user form | None                  | No Operation, do nothing         | -                                                                                                                   |
| No Operation, do nothing (1) | No Operation                    | Placeholder after input triggers        | Telegram Trigger, Webhook, On form submission | Platform                        | -                                                                                                                   |
| Platform               | Set                                | Extracts platform query and receiver info | No Operation, do nothing (1) | Get row(s)                      | -                                                                                                                   |
| Get row(s)             | Data Table (Get)                   | Looks up existing promo codes for platform | Platform               | Code Exist?                     | -                                                                                                                   |
| Code Exist?            | If                                 | Checks if promo codes exist for platform | Get row(s)             | notify telegram, Send a message, Respond to Webhook (True branch); Promo Seeker Agent (False branch) | -                                                                                                                   |
| notify telegram        | Telegram                           | Sends promo code info via Telegram message | Code Exist? (True)     | None                           | See Sticky Note for Telegram Bot setup: https://t.me/botfather                                                     |
| Send a message         | Gmail                             | Sends promo code info via email         | Code Exist? (True)     | None                           | See Sticky Note for Gmail OAuth2 setup                                                                             |
| Respond to Webhook     | Respond to Webhook (Disabled)     | Intended to respond to webhook requests | Code Exist? (True)     | None                           | Currently disabled                                                                                                  |
| Promo Seeker Agent     | LangChain Agent                   | AI agent to find new promo codes when none exist | Code Exist? (False)    | Upsert row(s)                   | -                                                                                                                   |
| SerpAPI                | LangChain Tool                   | Provides web search results to AI agent | Connected to Promo Seeker Agent (ai_tool) | Promo Seeker Agent (ai_tool)   | See Sticky Note for SerpAPI API key setup                                                                          |
| Gemini 2.5Pro          | LangChain LM (OpenRouter)        | Provides language model responses to AI agent | Connected to Promo Seeker Agent (ai_languageModel) | Promo Seeker Agent (ai_languageModel) | See Sticky Note for OpenRouter API key setup                                                                        |
| Structured Output Parser | LangChain Output Parser           | Parses AI agent output into structured promo code JSON | Connected to Promo Seeker Agent (ai_outputParser) | Promo Seeker Agent             | -                                                                                                                   |
| Upsert row(s)          | Data Table (Upsert)               | Stores new promo codes into data table  | Promo Seeker Agent     | None                           | -                                                                                                                   |
| Schedule Trigger       | Schedule Trigger                   | Placeholder for scheduled operations    | None                   | No Operation, do nothing (2)    | -                                                                                                                   |
| No Operation, do nothing (2) | No Operation                    | Placeholder after schedule trigger      | Schedule Trigger        | Platform                      | -                                                                                                                   |
| Sticky Note            | Sticky Note                       | Instructions on credential setup         | None                   | None                           | Contains detailed credential setup instructions and security notes                                                |
| Sticky Note1           | Sticky Note                       | Workflow overview explanation             | None                   | None                           | Describes workflow purpose and usage                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials (via BotFather)  
   - Set to listen for message updates

2. **Create Webhook Node**  
   - Type: Webhook  
   - Set path to `/v1/promo-seeker` and HTTP method POST  
   - Set response mode to respond via a Respond to Webhook node

3. **Create On form submission Node**  
   - Type: Form Trigger  
   - Title: "Promo Seeker"  
   - Fields:  
     - Platform (string, required)  
     - email (string, required)  
   - Description: "temukan promocode disini"

4. **Create No Operation Node (No Operation, do nothing #1)**  
   - Type: No Operation  
   - Connect Telegram Trigger, Webhook, and On form submission nodes to this node

5. **Create Platform Set Node**  
   - Type: Set  
   - Add two fields:  
     - `query`: Use expression to extract platform from incoming JSON:  
       `{{$json.Platform || $json.message?.text || $json.body.platform || ''}}`  
     - `receiver`: Extract receiver ID/email similarly:  
       `{{$json.chatId || $json.email || $json.body.email || ''}}`  
   - Connect No Operation node (1) output here

6. **Create Data Table Get row(s) Node**  
   - Type: Data Table  
   - Operation: Get rows from your data table "Kode Promo Valid"  
   - Filter: `platform == {{$json.query}}`  
   - Limit: 3 rows  
   - Connect Platform node output here

7. **Create If Node (Code Exist?)**  
   - Type: If  
   - Condition: Check if `$json.platform` exists (string exists)  
   - Connect Get row(s) node output here

8. **Create Telegram Send Message Node (notify telegram)**  
   - Type: Telegram  
   - Configure with your Telegram bot credentials  
   - Message: Use HTML format with promo details (see original workflow for template)  
   - Chat ID: Use `{{$('Telegram Trigger').item.json.message.chat.id}}`  
   - Connect If node’s True output here

9. **Create Gmail Send Email Node (Send a message)**  
   - Type: Gmail  
   - Configure Gmail OAuth2 credentials  
   - Send to: `{{$('Webhook').item.json.body.email}}`  
   - Subject: "Kode Promo {{ $json.platform.toUpperCase() || $json.platform }} Sudah Tersedia"  
   - Message: Use full HTML email template with promo details (copy from original)  
   - Connect If node’s True output here

10. **Create Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Initially disable this node (optional)  
    - Connect If node’s True output here (optional)

11. **Create LangChain Promo Seeker Agent Node**  
    - Type: LangChain Agent  
    - Set prompt with system message instructing promo code search and formatting (copy from original)  
    - Input text: `platform: {{$('Platform').item.json.query}}`  
    - Connect If node’s False output here

12. **Create SerpAPI Node**  
    - Type: LangChain SerpAPI tool  
    - Configure with SerpAPI credentials (API key from https://serpapi.com/)  
    - Connect output to Promo Seeker Agent’s ai_tool input

13. **Create Gemini 2.5Pro Node**  
    - Type: LangChain LM Chat (OpenRouter)  
    - Model: google/gemini-2.5-pro  
    - Configure with OpenRouter API key (from https://openrouter.ai/)  
    - Connect output to Promo Seeker Agent’s ai_languageModel input

14. **Create Structured Output Parser Node**  
    - Type: LangChain Output Parser Structured  
    - Define JSON schema example with fields: platform, code1, value, terms, validUntil  
    - Connect output to Promo Seeker Agent’s ai_outputParser input

15. **Connect Promo Seeker Agent output to Upsert row(s) Node**  
    - Type: Data Table (Upsert)  
    - Map parsed fields from AI output to data table columns: platform, promoCode, value, termsConditions, validUntil  
    - Match on promoCode to update or insert  
    - Connect Promo Seeker Agent output here

16. **Connect Upsert row(s) output to notify telegram and Send a message Nodes (optional)**  
    - To notify users of newly found promos

17. **Create Schedule Trigger Node** (Optional)  
    - For periodic workflow runs, connect to No Operation node (2) as placeholder

18. **Add Sticky Notes**  
    - Create Sticky Notes with detailed instructions on credentials setup and workflow overview as per original content

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Detailed guide on obtaining and configuring credentials for SerpAPI, OpenRouter (Gemini 2.5 Pro), Telegram bots (via BotFather), Gmail OAuth2, and setting up required Data Table columns (`platform`, `promoCode`, `value`, `termsConditions`, `validUntil`). Emphasizes keeping API keys secure and not sharing publicly.                                                                                                                                                                                                                      | See Sticky Note node content in workflow                                                       |
| Workflow purpose and usage explanation: Automates discovery of valid and recent promo codes; users send a message specifying the promo type; AI searches and filters results; responses delivered via chat or email. Designed for simplicity and automation.                                                                                                                                                                                                                                                                                                                               | See Sticky Note1 node content in workflow                                                      |
| Telegram Bot creation instructions available at https://t.me/botfather                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Telegram official bot creation guide                                                           |
| SerpAPI signup and API key retrieval at https://serpapi.com/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | SerpAPI official website                                                                        |
| OpenRouter API key signup at https://openrouter.ai/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | OpenRouter official website                                                                    |
| Gmail OAuth2 setup is performed from within n8n credentials panel; requires Google account authorization                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Google OAuth2 documentation                                                                    |
| Data Table "Kode Promo Valid" schema requires columns: `platform` (string), `promoCode` (string), `value` (string), `termsConditions` (string), `validUntil` (dateTime)                                                                                                                                                                                                                                                                                                                                                                                                                      | n8n Data Tables documentation                                                                  |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow and fully complies with content policies. It contains no illegal, offensive, or protected elements. All handled data is legal and public.