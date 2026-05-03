Landing Page Conversion Optimizer with Gemini 2.5-pro & Telegram

https://n8nworkflows.xyz/workflows/landing-page-conversion-optimizer-with-gemini-2-5-pro---telegram-6962


# Landing Page Conversion Optimizer with Gemini 2.5-pro & Telegram

### 1. Workflow Overview

This n8n workflow, titled **Landing Page Conversion Optimizer with Gemini 2.5-pro & Telegram**, is designed to analyze landing pages to improve their conversion rates by providing a detailed critique ("roast") and actionable optimization recommendations. The workflow caters primarily to business founders and CMOs aiming to boost landing page performance via data-driven insights powered by AI.

**Target Use Cases:**
- Users submit a landing page URL either via a web form or through a Telegram bot.
- The workflow scrapes the landing page HTML content.
- It processes the content using Google Gemini 2.5-pro AI to generate a personalized roast and 10 specific conversion rate optimization (CRO) ideas.
- The AI-generated insights are returned to the user via Telegram or displayed after form submission.

**Logical Blocks:**

- **1.1 Entry via Web Form:** Receives landing page URL input from a user-submitted web form.
- **1.2 Entry via Telegram:** Receives landing page URL input from Telegram messages.
- **1.3 URL Normalization:** Ensures URLs include the proper scheme (http/https).
- **1.4 Web Page Scraping:** Requests and retrieves the raw HTML content of the landing page.
- **1.5 AI Processing (Gemini 2.5-pro):** Uses an AI agent to analyze the scraped content and generate the roast and CRO recommendations.
- **1.6 Result Delivery:** Sends the AI-generated output back to the user via Telegram or form response.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry via Web Form

- **Overview:** Captures the landing page URL from a web user through a form interface titled "Conversion Rate Optimizer".
- **Nodes Involved:**
  - Landing Page Url (Form Trigger)
  - Code1 (URL Normalization)
  - Scrape Website (HTTP Request)
  - AI Agent (Langchain Agent)
  - Send a text message (Telegram Node)
  - Google Gemini Chat Model (Language Model Node)

- **Node Details:**

  - **Landing Page Url**
    - Type: Form Trigger
    - Role: Entry point accepting user input via web form.
    - Configuration: Single required field "Landing Page Url" with placeholder example.
    - Inputs: External web form submission.
    - Outputs: JSON with submitted URL.
    - Potential Failures: Form not submitted, blank input.

  - **Code1**
    - Type: Code Node (JavaScript)
    - Role: Normalizes URL by ensuring it includes "http://" or "https://".
    - Key Logic: Checks fields like 'text', 'url', etc., prepends "https://" if missing.
    - Inputs: JSON from form submission.
    - Outputs: Normalized URL JSON.
    - Edge cases: Malformed URLs may not be corrected fully.

  - **Scrape Website**
    - Type: HTTP Request
    - Role: Retrieves raw HTML content of the landing page URL.
    - Configuration: URL dynamically taken from normalized input.
    - Inputs: Normalized URL.
    - Outputs: Raw HTML data as JSON.
    - Failures: HTTP errors (404, timeout, 500), inaccessible or non-public pages.

  - **AI Agent**
    - Type: Langchain Agent Node
    - Role: Processes scraped HTML content with a detailed prompt to perform a landing page roast and generate CRO ideas.
    - Configuration: Detailed multi-criteria prompt instructing tone, structure, and output constraints (English, max 3000 characters).
    - Inputs: Landing page HTML content.
    - Outputs: Textual roast and recommendations.
    - Edge cases: AI rate limits, model errors, prompt execution timeouts.

  - **Google Gemini Chat Model**
    - Type: Langchain Google Gemini Chat Model
    - Role: Underlying AI model used by the AI Agent node.
    - Configuration: Uses "models/gemini-2.5-pro" with Google PaLM API credentials.
    - Inputs/Outputs: Handles text generation for AI Agent.
    - Failures: API authentication errors, quota limits.

  - **Send a text message**
    - Type: Telegram Node
    - Role: Sends the AI-generated message to the Telegram user who triggered the workflow.
    - Configuration: Sends output text to chat ID derived from Telegram Trigger node.
    - Inputs: AI Agent output.
    - Outputs: Message sent confirmation.
    - Failures: Telegram API errors, invalid chat IDs.

---

#### 2.2 Entry via Telegram

- **Overview:** Accepts landing page URLs sent as messages to a Telegram bot, normalizes the URL, scrapes the page, processes with AI, and replies back on Telegram.
- **Nodes Involved:**
  - Telegram Trigger
  - Code (URL Normalization)
  - Scrape Website1 (HTTP Request)
  - AI Agent1 (Langchain Agent)
  - Send a text message1 (Telegram Node)
  - Google Gemini Chat Model1 (Language Model Node)

- **Node Details:**

  - **Telegram Trigger**
    - Type: Telegram Trigger
    - Role: Entry point that listens for new messages to the Telegram bot.
    - Configuration: Monitors "message" updates.
    - Inputs: Telegram message containing URL.
    - Outputs: JSON with message data.
    - Failures: Telegram API connectivity issues.

  - **Code**
    - Type: Code Node
    - Role: Normalizes URLs in Telegram messages, ensuring "https://" prefix.
    - Logic: Same as Code1 node but applied to Telegram message fields.
    - Inputs: Telegram message JSON.
    - Outputs: Normalized URL JSON.
    - Edge cases: Messages without URLs, malformed URLs.

  - **Scrape Website1**
    - Type: HTTP Request
    - Role: Fetches HTML content of normalized URL.
    - Inputs: Normalized URL from Code node.
    - Outputs: Raw HTML content.
    - Failures: Same as Scrape Website node.

  - **AI Agent1**
    - Type: Langchain Agent Node
    - Role: Processes the scraped HTML with identical prompt and rules to AI Agent.
    - Inputs: Landing page HTML.
    - Outputs: Roast and recommendations.
    - Edge cases: Same as AI Agent.

  - **Google Gemini Chat Model1**
    - Type: Langchain Google Gemini Chat Model
    - Role: AI model for AI Agent1.
    - Configuration: Same as Gemini Chat Model.
    - Failures: Same as Gemini Chat Model.

  - **Send a text message1**
    - Type: Telegram Node
    - Role: Sends AI output back to Telegram user, replying to original message.
    - Configuration: Includes reply_to_message_id for context.
    - Inputs: AI Agent1 output.
    - Outputs: Telegram message sent.
    - Failures: Telegram API errors.

---

#### 2.3 Sticky Notes (Documentation Nodes)

- **Sticky Note (Entry via Form)**
  - Content: Marks the form entry block visually.

- **Sticky Note1 (Entry via Telegram)**
  - Content: Marks the Telegram entry block.

- **Sticky Note2 (Overview & Quick Guide)**
  - Content: Describes the overall workflow purpose, usage instructions, entry methods, processing logic, and usage tips.
  - Notes:
    - Highlights that output language is Turkish (though prompt states English — this is a discrepancy).
    - Emphasizes one URL per request and public page requirement.
    - Provides operational guidance for end users and developers.

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                                 | Input Node(s)       | Output Node(s)             | Sticky Note                                                                                         |
|-----------------------|-----------------------------------|------------------------------------------------|---------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| Landing Page Url       | Form Trigger                      | User input form for landing page URL            | External form input  | Code1                      | Entry via Form                                                                                    |
| Code1                 | Code Node                        | Normalize URL from form input                    | Landing Page Url     | Scrape Website             | Entry via Form                                                                                    |
| Scrape Website        | HTTP Request                    | Scrape landing page HTML                         | Code1                | AI Agent                   | Entry via Form                                                                                    |
| AI Agent              | Langchain Agent                 | Generate roast & CRO ideas via Gemini 2.5-pro  | Scrape Website       | Send a text message        | Entry via Form                                                                                    |
| Google Gemini Chat Model | Langchain Google Gemini Model | AI model powering AI Agent                       | AI Agent (internal)  | AI Agent                   |                                                                                                   |
| Send a text message   | Telegram Node                   | Send AI output to Telegram user                  | AI Agent             | None                       | Entry via Form                                                                                    |
| Telegram Trigger      | Telegram Trigger                | Telegram bot message listener                     | External Telegram    | Code                       | Entry via Telegram                                                                               |
| Code                  | Code Node                      | Normalize URL from Telegram message               | Telegram Trigger     | Scrape Website1            | Entry via Telegram                                                                              |
| Scrape Website1       | HTTP Request                  | Scrape landing page HTML for Telegram input       | Code                 | AI Agent1                  | Entry via Telegram                                                                              |
| AI Agent1             | Langchain Agent               | Generate roast & CRO ideas via Gemini 2.5-pro    | Scrape Website1      | Send a text message1       | Entry via Telegram                                                                              |
| Google Gemini Chat Model1 | Langchain Google Gemini Model | AI model powering AI Agent1                        | AI Agent1 (internal) | AI Agent1                  |                                                                                                   |
| Send a text message1  | Telegram Node                 | Send AI output reply message on Telegram          | AI Agent1            | None                       | Entry via Telegram                                                                              |
| Sticky Note           | Sticky Note                   | Documentation - Entry via Form                     | None                 | None                       | Entry via Form block visual marker                                                             |
| Sticky Note1          | Sticky Note                   | Documentation - Entry via Telegram                  | None                 | None                       | Entry via Telegram block visual marker                                                         |
| Sticky Note2          | Sticky Note                   | Documentation - Workflow overview and quick guide | None                 | None                       | Overview and usage instructions for entire workflow                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Name: `Landing Page Url`
   - Type: Form Trigger
   - Configure form with one required field:
     - Label: "Landing Page Url"
     - Placeholder: "https://yuzuu.co"
   - Form title: "Conversion Rate Optimizer"
   - Form description: "Your Landing Page is Leaking Sales—Fix It Now"

2. **Add URL Normalization Code Node:**
   - Name: `Code1`
   - Use JavaScript to check if URL fields start with "http://" or "https://"
   - If missing, prepend "https://"
   - Input from `Landing Page Url`
   - Fields checked: 'text', 'url', 'link', 'address'

3. **Add HTTP Request Node to Scrape Website:**
   - Name: `Scrape Website`
   - Set URL to the normalized URL from `Code1`
   - Method: GET (default)
   - Output: Full page HTML content

4. **Add Langchain Agent Node for AI Processing:**
   - Name: `AI Agent`
   - Use the detailed prompt to instruct the AI to roast and recommend CRO improvements for the landing page content.
   - Input: `data` field set to the scraped HTML content (`$json.data`)
   - Output: Text with roast and recommendations

5. **Add Google Gemini Chat Model Node:**
   - Name: `Google Gemini Chat Model`
   - Model: "models/gemini-2.5-pro"
   - Set credentials to Google PaLM API account
   - Connect AI Agent node to this model node as the language model backend

6. **Add Telegram Node to Send Message:**
   - Name: `Send a text message`
   - Text: Output from `AI Agent`
   - Chat ID: Use the Telegram user ID from incoming request (`$('Telegram Trigger').item.json.message.from.id`)
   - Credentials: Telegram Bot API credentials

7. **Connect the nodes in order:**
   - `Landing Page Url` → `Code1` → `Scrape Website` → `AI Agent` → `Send a text message`
   - Link `AI Agent` to `Google Gemini Chat Model` as language model

8. **Create Telegram Trigger Node:**
   - Name: `Telegram Trigger`
   - Listen for new message updates
   - Credentials: Telegram Bot API

9. **Add URL Normalization Code Node for Telegram:**
   - Name: `Code`
   - Same logic as `Code1` to normalize URL from Telegram message text

10. **Add HTTP Request Node for Telegram:**
    - Name: `Scrape Website1`
    - URL from normalized Telegram message

11. **Add AI Agent Node for Telegram:**
    - Name: `AI Agent1`
    - Same prompt and configuration as `AI Agent`
    - Connect with Gemini Chat Model1

12. **Add Google Gemini Chat Model Node for Telegram:**
    - Name: `Google Gemini Chat Model1`
    - Same model and credentials as previous Gemini model

13. **Add Telegram Send Message Node for Telegram:**
    - Name: `Send a text message1`
    - Send reply message to same user with AI Agent1 output
    - Include reply_to_message_id for context

14. **Connect Telegram nodes:**
    - `Telegram Trigger` → `Code` → `Scrape Website1` → `AI Agent1` → `Send a text message1`
    - Link `AI Agent1` to `Google Gemini Chat Model1`

15. **Add sticky notes for documentation** (optional):
    - Mark "Entry via Form" around form-based nodes
    - Mark "Entry via Telegram" around Telegram-based nodes
    - Add a workflow overview note describing usage, triggers, and processing steps

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini 2.5-pro via Google PaLM API as the AI engine for generating personalized landing page critiques. | Credential setup for Google PaLM API required; ensure API keys and quota are correctly configured.                                    |
| Telegram bot @MertSiteRaporBot is used for user interaction; Telegram Bot API credentials must be set up in n8n.                | https://core.telegram.org/bots/api                                                                                                   |
| URLs must be publicly accessible and include scheme (http or https). The code nodes add "https://" if missing.                  | This avoids request failures due to malformed URLs.                                                                                   |
| The AI prompt enforces a strict 3000-character limit on the output and requires answers in English, while sticky notes mention Turkish. | Confirm final language requirements and adjust prompt accordingly to avoid output language mismatches.                                |
| Output messages are sent back to users on Telegram, either in direct messages or replies, providing immediate feedback.        | Enables real-time interaction and quick CRO advice delivery.                                                                          |
| Usage tip: One URL per request; very long pages may be trimmed due to model input size limits.                                  | Consider adding additional logic to handle very large pages or multiple URLs in future enhancements.                                  |
| The detailed prompt enforces a fun, unconventional style for the roast and highly specific, creative CRO recommendations.       | Ensures outputs stand out from generic marketing advice, adding real value for users.                                                 |
| Workflow offers two main triggering methods: web form and Telegram bot, catering to different user preferences and scenarios.   | Telegram approach is faster and preferred for frequent users.                                                                          |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.