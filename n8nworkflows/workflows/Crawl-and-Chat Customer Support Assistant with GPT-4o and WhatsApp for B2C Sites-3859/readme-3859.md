Crawl-and-Chat Customer Support Assistant with GPT-4o and WhatsApp for B2C Sites

https://n8nworkflows.xyz/workflows/crawl-and-chat-customer-support-assistant-with-gpt-4o-and-whatsapp-for-b2c-sites-3859


# Crawl-and-Chat Customer Support Assistant with GPT-4o and WhatsApp for B2C Sites

### 1. Workflow Overview

This workflow implements a **Crawl-and-Chat Customer Support Assistant** that leverages GPT-4o and WhatsApp to provide real-time, AI-driven customer support for any B2C website. It automatically crawls the target business site on demand, extracts relevant content, and answers customer queries via WhatsApp, maintaining conversational context through persistent chat memory stored in Supabase/Postgres.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming WhatsApp messages via webhook.
- **1.2 AI Processing:** Uses LangChain’s AI Agent with GPT-4o to interpret user queries, crawl the website dynamically, and generate answers.
- **1.3 Web Crawling Tools:** Two HTTP request tools (`list_links` and `get_page`) that provide the AI Agent with internal site navigation and page content.
- **1.4 Chat Memory:** Stores and retrieves conversation history from Supabase/Postgres to maintain context across sessions.
- **1.5 Response Post-Processing:** Cleans AI-generated answers to remove unwanted formatting.
- **1.6 WhatsApp Messaging:** Sends replies back to users, including handling Meta’s 24-hour messaging window by optionally sending a pre-approved template message to reopen conversations.
- **1.7 Setup & Documentation:** A sticky note node contains detailed setup instructions and membership information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming WhatsApp messages via webhook to trigger the workflow.
- **Nodes Involved:** `WhatsApp Trigger`
- **Node Details:**
  - **Type:** WhatsApp Trigger node (Webhook listener)
  - **Configuration:** Listens for "messages" updates from WhatsApp Cloud API.
  - **Key Expressions:** None; raw incoming message JSON is passed downstream.
  - **Input:** External WhatsApp messages via webhook.
  - **Output:** JSON containing message text, sender info, timestamp.
  - **Edge Cases:** Webhook misconfiguration, invalid message formats, credential/auth errors.
  - **Sub-workflow:** None.

#### 2.2 AI Processing

- **Overview:** Processes the user query text, dynamically crawls the website using tools, and generates a context-aware answer.
- **Nodes Involved:** `AI Agent`, `OpenAI Chat Model`
- **Node Details:**

  - **AI Agent**
    - **Type:** LangChain Agent node
    - **Role:** Core AI logic orchestrator; receives user text, calls tools, and generates answers.
    - **Configuration:**
      - Input text: Extracted from incoming WhatsApp message body.
      - System message: Defines the assistant’s role as the company’s real-time website assistant.
      - Tools available:
        - `list_links(url)`: returns up to 100 internal links from a page.
        - `get_page(url)`: returns visible, tag-free text of a page.
      - Search strategy: Starts crawling from root URL, selects up to 5 relevant links, fetches page text, repeats up to two rounds or eight page fetches.
      - Answer rules: Friendly tone, no markdown or formatting symbols, quotes exact site wording, fallback message if info not found.
      - Max iterations: 10.
      - Returns intermediate steps for debugging.
    - **Input:** User message text.
    - **Output:** AI-generated answer and intermediate crawl data.
    - **Edge Cases:** API rate limits, tool call failures, malformed URLs, infinite loops if search strategy misfires.
    - **Version:** 1.7.

  - **OpenAI Chat Model**
    - **Type:** LangChain OpenAI Chat Model node
    - **Role:** Provides GPT-4o-mini language model for AI Agent.
    - **Configuration:** Model set to `gpt-4o-mini`.
    - **Credentials:** OpenAI API key required.
    - **Input:** Prompt from AI Agent.
    - **Output:** Model-generated text.
    - **Edge Cases:** API key invalid, rate limits, network errors.
    - **Version:** 1.2.

#### 2.3 Web Crawling Tools

- **Overview:** Provide the AI Agent with site navigation and content extraction capabilities.
- **Nodes Involved:** `list_links`, `get_page`
- **Node Details:**

  - **list_links**
    - **Type:** LangChain HTTP Request Tool node
    - **Role:** Returns up to 100 unique internal links from a given page URL.
    - **Configuration:**
      - POST to `https://lemolex.app.n8n.cloud/webhook/list-links`
      - Body parameters: `url` (root domain), `auth-token` (membership key)
      - Filters out off-site, mailto:, tel:, javascript: links.
      - Deduplicates links.
    - **Input:** URL from AI Agent.
    - **Output:** JSON array of internal URLs.
    - **Edge Cases:** Invalid URL, expired auth token, network errors.
    - **Version:** 1.1.

  - **get_page**
    - **Type:** LangChain HTTP Request Tool node
    - **Role:** Fetches fully rendered plain text content of a single web page.
    - **Configuration:**
      - POST to `https://lemolex.app.n8n.cloud/webhook/get_text`
      - Body parameters: `url`, `auth-token`
      - Returns visible text with all HTML tags removed.
    - **Input:** URL from AI Agent.
    - **Output:** JSON with page text.
    - **Edge Cases:** Off-site or invalid URLs, auth failures, network timeouts.
    - **Version:** 1.1.

#### 2.4 Chat Memory

- **Overview:** Maintains conversation context by storing and retrieving message history per user.
- **Nodes Involved:** `Postgres Users Memory`
- **Node Details:**
  - **Type:** LangChain Postgres Chat Memory node
  - **Role:** Persists chat history in a Postgres or Supabase database.
  - **Configuration:**
    - Table: `message_history`
    - Session key: WhatsApp user ID (`contacts[0].wa_id`)
    - Session ID type: custom key (user-specific)
  - **Credentials:** Postgres or Supabase connection required.
  - **Input:** Incoming message and AI responses.
  - **Output:** Chat history for AI Agent context.
  - **Edge Cases:** DB connection failures, schema mismatches, data corruption.
  - **Version:** 1.3.

#### 2.5 Response Post-Processing

- **Overview:** Cleans AI-generated answers to remove unwanted markdown or formatting symbols before sending.
- **Nodes Involved:** `cleanAnswer`
- **Node Details:**
  - **Type:** Code node (JavaScript)
  - **Role:** Sanitizes AI output by:
    - Removing bold/italic/strike markers (`*`, `_`, `~`)
    - Converting markdown links `[text](url)` to `text url`
    - Collapsing multiple blank lines
  - **Input:** AI Agent’s raw answer.
  - **Output:** Cleaned answer text.
  - **Edge Cases:** Unexpected text formats, empty answers.
  - **Version:** 2.

#### 2.6 WhatsApp Messaging

- **Overview:** Sends replies back to users on WhatsApp, respecting Meta’s 24-hour messaging window.
- **Nodes Involved:** `24-hour window check`, `If`, `Send Pre-approved Template Message to Reopen the Conversation`, `Send AI Agent's Answer`
- **Node Details:**

  - **24-hour window check**
    - **Type:** Code node
    - **Role:** Checks if the last user message timestamp is within Meta’s 24-hour service window.
    - **Logic:** Compares current time with message timestamp.
    - **Output:** Boolean flag `withinWindow`.
    - **Edge Cases:** Timestamp missing or malformed.

  - **If**
    - **Type:** If node
    - **Role:** Routes workflow based on `withinWindow` flag.
    - **True branch:** Sends AI Agent’s answer directly.
    - **False branch:** Sends a pre-approved WhatsApp template message to reopen the conversation before replying.

  - **Send Pre-approved Template Message to Reopen the Conversation**
    - **Type:** WhatsApp node
    - **Role:** Sends a Meta-approved template message (e.g., "hello_world") to reopen chat outside the 24-hour window.
    - **Configuration:** Template name and recipient phone number.
    - **Credentials:** WhatsApp API OAuth2.
    - **Edge Cases:** Template not approved, API errors.

  - **Send AI Agent's Answer**
    - **Type:** WhatsApp node
    - **Role:** Sends the cleaned AI-generated answer as a free-form WhatsApp message.
    - **Configuration:** Text body from `cleanAnswer`, recipient phone number.
    - **Credentials:** WhatsApp API OAuth2.
    - **Edge Cases:** Message length limits, API failures.

#### 2.7 Setup & Documentation

- **Overview:** Provides detailed setup instructions, membership info, and usage notes.
- **Nodes Involved:** `Sticky Note`
- **Node Details:**
  - **Type:** Sticky Note node
  - **Content:** Step-by-step setup guide, membership activation link, credential instructions, customization tips.
  - **Purpose:** Assists users in configuring the workflow correctly.
  - **Edge Cases:** None (informational only).

---

### 3. Summary Table

| Node Name                                   | Node Type                             | Functional Role                                  | Input Node(s)                 | Output Node(s)                                      | Sticky Note                                                                                              |
|---------------------------------------------|-------------------------------------|-------------------------------------------------|------------------------------|----------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger                            | WhatsApp Trigger                    | Receives incoming WhatsApp messages             | External webhook             | AI Agent                                           |                                                                                                        |
| AI Agent                                   | LangChain Agent                    | Core AI processing, orchestrates crawling & answering | WhatsApp Trigger, list_links, get_page, Postgres Users Memory, OpenAI Chat Model | 24-hour window check                              |                                                                                                        |
| OpenAI Chat Model                          | LangChain OpenAI Chat Model        | Provides GPT-4o-mini language model              | AI Agent                     | AI Agent                                           |                                                                                                        |
| list_links                                 | LangChain HTTP Request Tool        | Returns internal site links for crawling         | AI Agent (tool call)         | AI Agent                                           |                                                                                                        |
| get_page                                   | LangChain HTTP Request Tool        | Fetches plain text content of a page              | AI Agent (tool call)         | AI Agent                                           |                                                                                                        |
| Postgres Users Memory                      | LangChain Postgres Chat Memory     | Stores and retrieves chat history                 | WhatsApp Trigger             | AI Agent                                           |                                                                                                        |
| 24-hour window check                       | Code                              | Checks if message is within Meta’s 24-hour window | AI Agent                    | If                                                 |                                                                                                        |
| If                                         | If                                | Routes based on 24-hour window check              | 24-hour window check         | cleanAnswer (true), Send Pre-approved Template Message (false) |                                                                                                        |
| cleanAnswer                                | Code                              | Cleans AI-generated answer text                    | If (true branch)             | Send AI Agent's Answer                             |                                                                                                        |
| Send Pre-approved Template Message to Reopen the Conversation | WhatsApp                          | Sends template message to reopen conversation     | If (false branch)            |                                                    |                                                                                                        |
| Send AI Agent's Answer                     | WhatsApp                          | Sends AI-generated answer to user                  | cleanAnswer                  |                                                    |                                                                                                        |
| Sticky Note                                | Sticky Note                      | Setup instructions and membership info            | None                        | None                                               | # Step by Step Setup Guide with membership link: https://lemolex.gumroad.com/l/ejsnx                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node:**
   - Type: WhatsApp Trigger
   - Configure webhook to listen for "messages" updates.
   - Connect WhatsApp OAuth2 credentials.
   - Position: Start of workflow.

2. **Create OpenAI Chat Model node:**
   - Type: LangChain OpenAI Chat Model
   - Set model to `gpt-4o-mini`.
   - Connect OpenAI API credentials.
   - Position: Mid workflow.

3. **Create `list_links` HTTP Request Tool node:**
   - Type: LangChain HTTP Request Tool
   - Method: POST
   - URL: `https://lemolex.app.n8n.cloud/webhook/list-links`
   - Body parameters:
     - `url`: your company root URL (e.g., https://www.your-company-url.com)
     - `auth-token`: your membership key
   - Position: Mid-lower workflow.

4. **Create `get_page` HTTP Request Tool node:**
   - Type: LangChain HTTP Request Tool
   - Method: POST
   - URL: `https://lemolex.app.n8n.cloud/webhook/get_text`
   - Body parameters:
     - `url`: dynamic URL from AI Agent
     - `auth-token`: your membership key
   - Position: Mid-lower workflow.

5. **Create Postgres Users Memory node:**
   - Type: LangChain Postgres Chat Memory
   - Table name: `message_history`
   - Session key: `={{ $json.contacts[0].wa_id }}`
   - Session ID type: custom key
   - Connect Supabase/Postgres credentials.
   - Position: Mid workflow.

6. **Create AI Agent node:**
   - Type: LangChain Agent
   - Input text: `={{ $json.messages[0].text.body }}`
   - System message: Customize with your company name and root URL.
   - Tools configured:
     - `list_links` and `get_page` nodes as tools.
   - Max iterations: 10
   - Return intermediate steps: true
   - Connect to:
     - OpenAI Chat Model as language model
     - Postgres Users Memory as memory
     - `list_links` and `get_page` as tools
   - Position: After WhatsApp Trigger.

7. **Create 24-hour window check node:**
   - Type: Code node
   - JavaScript code to compare current time with incoming message timestamp (in ms).
   - Output boolean `withinWindow`.
   - Position: After AI Agent.

8. **Create If node:**
   - Type: If node
   - Condition: `withinWindow` is true
   - Position: After 24-hour window check.

9. **Create cleanAnswer node:**
   - Type: Code node
   - JavaScript code to:
     - Remove markdown symbols (`*`, `_`, `~`)
     - Convert markdown links `[text](url)` to `text url`
     - Collapse multiple blank lines
   - Position: True branch of If node.

10. **Create Send AI Agent's Answer node:**
    - Type: WhatsApp node
    - Operation: send
    - Text body: `={{ $json.answer }}`
    - Recipient phone number: `={{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
    - Connect WhatsApp API credentials.
    - Position: After cleanAnswer.

11. **Create Send Pre-approved Template Message to Reopen the Conversation node:**
    - Type: WhatsApp node
    - Operation: send template
    - Template: e.g., `hello_world|en_US`
    - Recipient phone number: `={{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
    - Connect WhatsApp API credentials.
    - Position: False branch of If node.

12. **Connect nodes:**
    - WhatsApp Trigger → AI Agent
    - AI Agent → 24-hour window check
    - 24-hour window check → If
    - If (true) → cleanAnswer → Send AI Agent's Answer
    - If (false) → Send Pre-approved Template Message to Reopen the Conversation

13. **Add Sticky Note node:**
    - Content: Paste the full setup guide and membership info.
    - Position: Off to the side for reference.

14. **Configure variables:**
    - Replace `[Company Name]` and `https://www.your-company-url.com` in AI Agent’s system message.
    - Insert your membership `auth-token` in `list_links` and `get_page` nodes.
    - Set your root URL in `list_links` node body parameters.

15. **Credentials setup:**
    - Add OpenAI API key.
    - Add WhatsApp OAuth2 credentials.
    - Add Supabase/Postgres credentials.

16. **Test workflow:**
    - Publish webhook URL for WhatsApp.
    - Send “Hi” from WhatsApp to trigger the bot.
    - Verify AI Agent crawls site and replies appropriately.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The AI Agent continuously crawls and extracts fresh data from the website on demand, avoiding the need for document uploads or model fine-tuning.                                                                               | Workflow Overview                                                                                   |
| Membership required to activate crawling tools, priced at $29/month, significantly cheaper than comparable AI-support SaaS platforms charging $150–$500/month.                                                                  | Membership info in Sticky Note: https://lemolex.gumroad.com/l/ejsnx                                |
| Setup tutorials for Supabase/Postgres integration: https://youtu.be/6w5f_jsPYSQ?si=MPdXYUjxv3fghQPj&t=105                                                                                                                       | Supabase credential setup                                                                           |
| Setup tutorials for WhatsApp OAuth2 integration: https://youtu.be/ZrhTQle55LQ?si=MO_leooogO9KchCV                                                                                                                               | WhatsApp credential setup                                                                          |
| To disable the 24-hour window template message feature (not recommended), delete nodes `24-hour window check`, `If`, and `Send Pre-approved Template Message to Reopen the Conversation`, then connect `AI Agent` directly to `cleanAnswer`. | Optional workflow customization                                                                    |
| The system prompt enforces strict no-markdown formatting and instructs the AI to quote exact site wording, ensuring clear and compliant customer communication.                                                                  | AI Agent system message                                                                             |
| The workflow is channel-agnostic; swapping WhatsApp nodes with Telegram, Slack, or web chat nodes enables multi-channel support.                                                                                                | Extensibility note                                                                                  |
| Monetization ideas include replacing costly SaaS seats, selling white-label bots, upselling premium human hand-off, and embedding affiliate links in answers.                                                                   | Business strategy                                                                                   |

---

This documentation provides a complete, structured reference for understanding, reproducing, and extending the "Crawl-and-Chat Customer Support Assistant with GPT-4o and WhatsApp for B2C Sites" workflow. It covers all nodes, logic flows, configuration details, and operational considerations.