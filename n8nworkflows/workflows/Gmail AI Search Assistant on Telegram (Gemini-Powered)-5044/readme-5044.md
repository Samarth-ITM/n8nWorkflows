Gmail AI Search Assistant on Telegram (Gemini-Powered)

https://n8nworkflows.xyz/workflows/gmail-ai-search-assistant-on-telegram--gemini-powered--5044


# Gmail AI Search Assistant on Telegram (Gemini-Powered)

---
### 1. Workflow Overview

This workflow implements a **Gmail AI Search Assistant on Telegram** powered by Google Gemini (PaLM) models. It allows Telegram users to send natural language requests for email searches, which are then intelligently parsed, converted into Gmail search queries, executed, and the results formatted and sent back as concise Telegram messages.

The workflow is logically divided into these main blocks:

- **1.1 User Input Reception via Telegram:** Captures user messages from Telegram via a webhook trigger.
- **1.2 AI Request Parsing and Structuring:** Uses a Google Gemini AI agent to analyze the user’s natural language request and extract structured search parameters (sender, keywords, date filters).
- **1.3 Gmail Query Construction:** Converts the AI-parsed parameters into a valid Gmail search query string using a second AI chain.
- **1.4 Gmail Email Retrieval:** Queries the Gmail API for emails matching the constructed search query and date/sender filters.
- **1.5 Email Data Formatting:** Formats the retrieved email data into a Telegram-friendly message using a third AI chain.
- **1.6 Response Delivery:** Sends the formatted email summary messages back to the user's Telegram chat.

Supporting nodes include structured output parsers for JSON validation and several sticky notes providing setup instructions and configuration tips.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception via Telegram

- **Overview:**  
  Captures incoming Telegram messages from users to initiate the Gmail search workflow.

- **Nodes Involved:**  
  - User Request - Telegram Trigger

- **Node Details:**  
  - **User Request - Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Listens for new Telegram messages (`message` update type).  
    - Config: Uses Telegram API credentials with a registered bot token.  
    - Input/Output: Receives Telegram message JSON; outputs message data for downstream nodes.  
    - Edge Cases: Telegram webhook misconfiguration, invalid or missing API token, message format changes.

#### 2.2 AI Request Parsing and Structuring

- **Overview:**  
  Analyzes the user’s natural language request to extract structured search parameters (sender, keywords, date ranges) using a Google Gemini AI agent.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model2  
  - Structured Output Parser

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Parses user's raw request text and outputs JSON with search fields.  
    - Config: Uses a prompt instructing strict JSON output with fields: sender, keywords, afterDate, beforeDate.  
    - Expressions: Uses `{{ $json.message.text }}` from Telegram Trigger as input.  
    - Output: JSON with extracted fields.  
    - Edge Cases: Ambiguous dates, incomplete requests, model timeout or API failures.  
    - Model: Connects to Google Gemini Chat Model2.  

  - **Google Gemini Chat Model2**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Executes the AI model query for the agent.  
    - Config: `models/gemini-2.0-flash` model with Google PaLM credentials.  
    - Edge Cases: API rate limits, authentication errors.  

  - **Structured Output Parser**  
    - Type: JSON Structured Output Parser  
    - Role: Validates and parses AI agent’s JSON output.  
    - Config: Example schema includes sender, keywords, afterDate, beforeDate.  
    - Edge Cases: Parsing errors if AI outputs malformed JSON.

#### 2.3 Gmail Query Construction

- **Overview:**  
  Converts the structured parameters into a valid Gmail search query string, formatted according to Gmail’s search syntax using AI.

- **Nodes Involved:**  
  - Set Parameters From User Request (for fetching emails)  
  - Basic LLM Chain  
  - Google Gemini Chat Model  
  - Structured Output Parser1

- **Node Details:**  
  - **Set Parameters From User Request (for fetching emails)**  
    - Type: Set Node  
    - Role: Maps AI agent output fields (sender, keywords, afterDate, beforeDate) into workflow parameters.  
    - Config: Assigns values from previous JSON to named variables.  
    - Edge Cases: Missing or empty fields.  

  - **Basic LLM Chain**  
    - Type: LangChain Basic LLM Chain  
    - Role: AI prompt to generate Gmail search query JSON from user parameters.  
    - Config: Prompt instructs to produce JSON with fields "search" (Gmail query string) and "metadata".  
    - Input: User request text from Telegram and parameters set above.  
    - Output: Gmail query string and metadata JSON.  
    - Edge Cases: Incorrect query syntax, date formatting errors.  
    - Connected Model: Google Gemini Chat Model.  

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Executes the AI prompt for query construction.  
    - Config: Same Gemini model as before.  

  - **Structured Output Parser1**  
    - Type: JSON Structured Output Parser  
    - Role: Parses and validates the generated Gmail query JSON.  

#### 2.4 Gmail Email Retrieval

- **Overview:**  
  Uses the constructed Gmail search query and filters to fetch matching emails via Gmail API.

- **Nodes Involved:**  
  - Gets Requested Email(s)

- **Node Details:**  
  - **Gets Requested Email(s)**  
    - Type: Gmail Node  
    - Role: Queries Gmail mailbox using search query and filters (sender, unread status, date) from previous nodes.  
    - Config:  
      - Limit: 10 emails by default  
      - Filters: Uses `q` parameter (search query string), sender, unread status, afterDate, beforeDate  
      - Operation: `getAll` to retrieve email data  
      - Gmail OAuth2 credentials required  
    - Edge Cases: OAuth refresh failures, API quota exceeded, no emails found, malformed query.

#### 2.5 Email Data Formatting

- **Overview:**  
  Prepares retrieved emails into concise Telegram-ready summary messages using AI.

- **Nodes Involved:**  
  - Sets Parameters For Response  
  - Basic LLM Chain1  
  - Google Gemini Chat Model1  
  - Structured Output Parser2

- **Node Details:**  
  - **Sets Parameters For Response**  
    - Type: Set Node  
    - Role: Extracts and formats raw Gmail email fields into normalized parameters for AI formatting.  
    - Config: Converts internal date to ISO string, extracts sender info, subject, snippet.  
    - Edge Cases: Missing fields, malformed email metadata.  

  - **Basic LLM Chain1**  
    - Type: LangChain Basic LLM Chain  
    - Role: AI prompt to convert email data into a formatted Telegram message with date, sender, subject, and snippet under 400 chars.  
    - Config: Uses emojis and specific formatting rules (date DD/MM/YYYY, full snippet max 4 lines).  
    - Connected Model: Google Gemini Chat Model1.  

  - **Google Gemini Chat Model1**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Executes AI formatting prompt.  

  - **Structured Output Parser2**  
    - Type: JSON Structured Output Parser  
    - Role: Parses formatted Telegram message text from AI output.  

#### 2.6 Response Delivery

- **Overview:**  
  Sends the formatted summary messages back to the user's Telegram chat.

- **Nodes Involved:**  
  - Sends (requested emails) via Telegram back

- **Node Details:**  
  - **Sends (requested emails) via Telegram back**  
    - Type: Telegram Node  
    - Role: Sends text messages to Telegram chat using chat ID from original Telegram trigger.  
    - Config: Uses Telegram API credentials, disables attribution append.  
    - Input: Formatted message text from AI node, chat ID from Telegram Trigger node.  
    - Edge Cases: Invalid chat ID, Telegram API errors, message length limits.

---

### 3. Summary Table

| Node Name                                  | Node Type                        | Functional Role                         | Input Node(s)                                   | Output Node(s)                                                | Sticky Note                                                                                                                                                             |
|--------------------------------------------|---------------------------------|---------------------------------------|------------------------------------------------|---------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| User Request - Telegram Trigger             | Telegram Trigger                | Captures Telegram user input          |                                                | AI Agent                                                     | ## 1. Telegram Bot (API) Setup steps with BotFather and credentials                                                                                                   |
| AI Agent                                    | LangChain AI Agent             | Parses user request into structured JSON | User Request - Telegram Trigger                | Set Parameters From User Request (for fetching emails)        | Replace API with your own Gemini or OpenAI model depending on choice (applies to AI nodes)                                                                             |
| Google Gemini Chat Model2                    | LangChain Gemini Model         | Executes AI parsing prompt             | AI Agent                                        | AI Agent                                                     | Replace API with your own Gemini or OpenAI model (applies to AI nodes)                                                                                                |
| Structured Output Parser                     | JSON Structured Parser         | Validates AI JSON output               | AI Agent                                        | Set Parameters From User Request (for fetching emails)        |                                                                                                                                                                       |
| Set Parameters From User Request (for fetching emails) | Set Node                      | Maps AI output to variables            | AI Agent                                        | Basic LLM Chain                                              |                                                                                                                                                                       |
| Basic LLM Chain                             | LangChain LLM Chain            | Converts parameters to Gmail query JSON | Set Parameters From User Request (for fetching emails) | Gets Requested Email(s)                                       | Replace API with your own Gemini or OpenAI model (applies to AI nodes)                                                                                                |
| Google Gemini Chat Model                     | LangChain Gemini Model         | Executes AI query construction prompt | Basic LLM Chain                                 | Basic LLM Chain                                              | Replace API with your own Gemini or OpenAI model (applies to AI nodes)                                                                                                |
| Structured Output Parser1                    | JSON Structured Parser         | Validates Gmail query JSON             | Basic LLM Chain                                 | Gets Requested Email(s)                                       |                                                                                                                                                                       |
| Gets Requested Email(s)                      | Gmail                         | Retrieves matching emails              | Basic LLM Chain                                 | Sets Parameters For Response                                 | ## 2. Connecting Your Gmail Account. Instructions on email limit, read status                                                                                         |
| Sets Parameters For Response                 | Set Node                      | Extracts and formats email data        | Gets Requested Email(s)                         | Basic LLM Chain1                                             |                                                                                                                                                                       |
| Basic LLM Chain1                            | LangChain LLM Chain            | Formats email into Telegram message    | Sets Parameters For Response                     | Sends (requested emails) via Telegram back                   |                                                                                                                                                                       |
| Google Gemini Chat Model1                     | LangChain Gemini Model         | Executes AI formatting prompt          | Basic LLM Chain1                                | Basic LLM Chain1                                             | Replace API with your own Gemini or OpenAI model (applies to AI nodes)                                                                                                |
| Structured Output Parser2                    | JSON Structured Parser         | Validates formatted message JSON       | Basic LLM Chain1                                | Sends (requested emails) via Telegram back                   |                                                                                                                                                                       |
| Sends (requested emails) via Telegram back  | Telegram                      | Sends final messages back to user      | Basic LLM Chain1, User Request - Telegram Trigger |                                                               | ## 3. Chat ID usage instructions for Telegram message sending                                                                                                         |
| Sticky Note                                  | Sticky Note                   | Telegram Bot Setup instructions        |                                                |                                                               | ## 1. Telegram Bot (API) Setup steps with BotFather and credentials                                                                                                   |
| Sticky Note1                                 | Sticky Note                   | API replacement reminder for Gemini/OpenAI |                                                |                                                               | Replace the API with your own API of Gemini or choose an OpenAi node if you want to use openai model. (depends on your choice).                                        |
| Sticky Note2                                 | Sticky Note                   | API replacement reminder for Gemini/OpenAI |                                                |                                                               | Replace the API with your own API of Gemini or choose an OpenAi node if you want to use openai model. (depends on your choice).                                        |
| Sticky Note3                                 | Sticky Note                   | Gmail account connection instructions  |                                                |                                                               | ## 2. Connecting Your Gmail Account. Instructions on email limit, read status                                                                                         |
| Sticky Note4                                 | Sticky Note                   | Chat ID extraction and usage instructions |                                                |                                                               | ## 3. Chat ID. How to obtain and use chat ID from Telegram trigger output                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credential:**  
   - Use Telegram @BotFather to create a new bot, obtain the API token.  
   - In n8n, create Telegram API credentials with the token and base URL `https://api.telegram.org`.

2. **Add Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Trigger on `message` updates.  
   - Use the Telegram credential.  
   - This node listens for user input messages.

3. **Add LangChain AI Agent Node:**  
   - Type: LangChain Agent  
   - Configure prompt to parse user request text into JSON with keys: sender, keywords, afterDate, beforeDate.  
   - Use expression: `User request: {{ $json.message.text }}`  
   - Connect LangChain agent to a Google Gemini Chat Model node (next step).  
   - Set retry on fail to handle transient errors.

4. **Add Google Gemini Chat Model Node (for AI Agent):**  
   - Type: LangChain Google Gemini Chat Model  
   - Model: `models/gemini-2.0-flash`  
   - Use Google PaLM API credentials.  
   - Connect output back to AI Agent node.

5. **Add Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser  
   - Provide example JSON schema matching AI Agent output.  
   - Connect output from AI Agent to this parser for validation.

6. **Add Set Node to Assign Parameters:**  
   - Map parsed JSON fields (`sender`, `keywords`, `afterDate`, `beforeDate`) into variables named accordingly for subsequent nodes.

7. **Add Basic LLM Chain Node to Build Gmail Query:**  
   - Prompt instructs AI to convert extracted parameters into a valid Gmail search query JSON with `search` and `metadata` fields.  
   - Input text includes user request from Telegram and extracted parameters.  
   - Connect this to another Google Gemini Chat Model node.

8. **Add Google Gemini Chat Model Node (for Query Construction):**  
   - Same configuration as before, linked to the Basic LLM Chain node.

9. **Add Structured Output Parser Node:**  
   - Validate the JSON Gmail query output.

10. **Add Gmail Node to Fetch Emails:**  
    - Operation: `getAll` emails.  
    - Set search filters:  
      - Query string from Gmail query JSON field `search`.  
      - Sender filter from Set node.  
      - Read status: unread  
      - Date filters: `receivedAfter` and `receivedBefore` from Set node.  
    - Use Gmail OAuth2 credentials.

11. **Add Set Node to Format Email Data:**  
    - Extract `internalDate` (convert to ISO string), `From`, `Subject`, `snippet` fields from Gmail node output.  
    - These serve as input for final formatting AI.

12. **Add Basic LLM Chain Node to Format Telegram Message:**  
    - Prompt to convert email data into a Telegram message with emojis, date in DD/MM/YYYY, sender name/email, subject, snippet limited to 4 lines and max 400 characters.  
    - Connect to another Google Gemini Chat Model node.

13. **Add Google Gemini Chat Model Node (for Formatting):**  
    - Same model and credentials as previous Gemini nodes.

14. **Add Structured Output Parser Node:**  
    - To parse and validate the formatted Telegram message text.

15. **Add Telegram Node to Send Message:**  
    - Use Telegram API credentials.  
    - Set chat ID from the original Telegram trigger node's message chat ID `{{ $('User Request - Telegram Trigger').item.json.message.chat.id }}`.  
    - Set text to the formatted message output from the previous node.  
    - Disable attribution append.

16. **Connect all nodes according to the logical flow:**  
    - Telegram Trigger → AI Agent → Structured Output Parser → Set Parameters → Basic LLM Chain (Query) → Structured Output Parser → Gmail Node → Set Node (Email Data) → Basic LLM Chain (Format) → Structured Output Parser → Telegram Send Node

17. **Test with sample Telegram messages to validate parsing, query construction, email retrieval, and messaging.**

---

### 5. General Notes & Resources

| Note Content                                                                              | Context or Link                                         |
|-------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Telegram bot setup requires using @BotFather to create the bot and obtain API tokens.     | Sticky Note near Telegram Trigger node                  |
| Replace Google Gemini API credentials with your own or switch to OpenAI nodes if preferred. | Sticky Notes near Gemini Chat Model nodes               |
| Gmail credentials require OAuth2 setup with appropriate scopes to read emails.             | Sticky Note near Gmail Node                              |
| Chat ID extraction: obtain from Telegram trigger output and pass to Telegram send node.   | Sticky Note near Telegram Send node                      |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n workflow automation designed for Gmail search via Telegram using AI. It complies fully with content policies and handles only legal and public data.