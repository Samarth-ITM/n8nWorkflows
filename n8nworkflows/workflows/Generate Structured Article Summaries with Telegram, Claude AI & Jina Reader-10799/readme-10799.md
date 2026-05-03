Generate Structured Article Summaries with Telegram, Claude AI & Jina Reader

https://n8nworkflows.xyz/workflows/generate-structured-article-summaries-with-telegram--claude-ai---jina-reader-10799


# Generate Structured Article Summaries with Telegram, Claude AI & Jina Reader

### 1. Workflow Overview

This workflow implements a Telegram bot that receives article URLs from users, fetches the clean article content via Jina AI's Reader API, generates an AI-powered structured summary using Claude AI via OpenRouter, and sends the formatted summary back to the user on Telegram. Optionally, summaries can be saved to Google Sheets for tracking.

**Target Use Cases:**  
- Quickly summarizing online articles shared via Telegram  
- Multilingual support matching article language (English, Spanish)  
- Clean content extraction removing ads and clutter  
- Structured output with title, one-sentence summary, and key points  

**Logical Blocks:**  
- **1.1 Input Reception:** Telegram message reception and filtering for valid URLs  
- **1.2 URL Extraction and Article Fetching:** Extract URL and chat info, fetch article markdown content  
- **1.3 AI Processing:** Summarize article content using Claude AI model and parse output  
- **1.4 Output Delivery:** Send summary to Telegram user and optionally save to Google Sheets  
- **1.5 Error Handling:** Notify user on failures during fetching or summarization  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives Telegram messages and filters messages to process only those starting with a valid URL (http/https).

**Nodes Involved:**  
- After message is received  
- Check if URL  
- Sticky Note1 (documentation)

**Node Details:**  

- **After message is received**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for new messages on Telegram bot webhook  
  - Config: Triggers on "message" updates only  
  - Credentials: Telegram API token for bot authentication  
  - Input: Incoming Telegram message JSON  
  - Output: Message JSON forwarded to filtering  
  - Edge Cases: Telegram API downtime, webhook misconfiguration, invalid message formats  
  - Retry: Enabled on failure  

- **Check if URL**  
  - Type: Filter  
  - Role: Filters messages that are URLs starting with "http://" or "https://"  
  - Config: Regex match on message text `^https?://` (case sensitive, strict validation)  
  - Input: Message JSON from Telegram trigger  
  - Output: Only messages passing regex proceed; others ignored  
  - Edge Cases: Non-URL messages, malformed URLs  
  - Notes: Ensures downstream nodes only process valid URLs  

- **Sticky Note1**  
  - Provides documentation on URL filtering behavior  

---

#### 1.2 URL Extraction and Article Fetching

**Overview:**  
Extracts the URL and chat ID from the Telegram message, then fetches the article content as clean markdown using Jina AI’s Reader API.

**Nodes Involved:**  
- Extract URL  
- Fetch Article Markdown  
- Notify an error while fetching (error handler)  
- Sticky Note3 (fetching explanation)

**Node Details:**  

- **Extract URL**  
  - Type: Set  
  - Role: Extracts and stores the URL and chat ID into workflow variables  
  - Config: Assigns `url` from message text and `chat_id` from message chat object  
  - Input: Filtered message JSON with URL  
  - Output: JSON with `url` and `chat_id` fields  
  - Edge Cases: Empty or malformed URLs (though minimized by prior filter)  

- **Fetch Article Markdown**  
  - Type: HTTP Request  
  - Role: Calls Jina AI Reader API to get clean markdown content of the article  
  - Config: HTTP GET request to `https://r.jina.ai/{{ $json.url }}`  
  - Input: JSON containing `url`  
  - Output: JSON with field `data` containing markdown article content  
  - OnError: Continues to error output to allow graceful handling  
  - Edge Cases: Paywalls, login-required pages, bot-blocking sites, timeouts  
  - Retry: Not specified, but error output handled downstream  

- **Notify an error while fetching**  
  - Type: Telegram  
  - Role: Sends failure notification to user if article fetching fails  
  - Config: Static error message explaining common fetch failures  
  - Input: Error output from Fetch Article Markdown node  
  - Output: Telegram message to user chat ID  
  - Retry: Enabled  
  - Edge Cases: Telegram API failures  

- **Sticky Note3**  
  - Explains purpose of clean content fetching and error cases  

---

#### 1.3 AI Processing

**Overview:**  
Sends the markdown article content to Claude AI model via OpenRouter for summarization, parses the structured JSON output.

**Nodes Involved:**  
- OpenRouter Chat Model  
- Summarize Article  
- Output Parser  
- Notify an error while summarizing (error handler)  
- Sticky Note4 (AI summary explanation)

**Node Details:**  

- **OpenRouter Chat Model**  
  - Type: LangChain LM Chat OpenRouter  
  - Role: Connects to Claude AI model ("anthropic/claude-3.5-haiku") via OpenRouter API  
  - Config: Model set to Claude 3.5 Haiku  
  - Credentials: OpenRouter API key with access to Claude model  
  - Input: Receives prompt from Summarize Article node (via chaining)  
  - Output: AI raw chat output forwarded to Summarize Article node  
  - Edge Cases: API key invalid, rate limits, model errors  

- **Summarize Article**  
  - Type: LangChain Chain LLM  
  - Role: Constructs prompt, sends article markdown to AI, expects structured JSON response  
  - Config:  
    - Prompt instructs AI to extract title, one-sentence summary, and 3-5 key points in same language as article (English or Spanish)  
    - Output parser enabled to validate JSON format  
    - Max retries: 2  
  - Input: Article markdown content from Fetch Article Markdown  
  - Output: Parsed summary JSON  
  - Error Handling: On failure, triggers Notify an error while summarizing node  
  - Edge Cases: AI output missing fields, language mismatch, prompt failures  

- **Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Validates and fixes AI JSON output to ensure correct schema  
  - Config:  
    - Manual JSON schema requiring title (string), summary (string), key_points (array of strings 3-5 items)  
    - AutoFix enabled to correct minor format errors  
  - Input: Raw AI output from OpenRouter Chat Model  
  - Output: Clean structured JSON for downstream use  

- **Notify an error while summarizing**  
  - Type: Telegram  
  - Role: Sends failure notification if AI summarization fails  
  - Config: Static error message advising alternative URLs or support contact  
  - Input: Error output from Summarize Article node  
  - Output: Telegram message to user chat ID  
  - Retry: Enabled  

- **Sticky Note4**  
  - Documents AI summary process and expected JSON output fields  

---

#### 1.4 Output Delivery

**Overview:**  
Formats the AI summary into a user-friendly message with emojis and sends it back to Telegram user. Optionally saves the summary data to Google Sheets.

**Nodes Involved:**  
- Send Summary to Telegram  
- Save to Google Sheets (disabled by default)  
- Sticky Note5 (output explanation)

**Node Details:**  

- **Send Summary to Telegram**  
  - Type: Telegram  
  - Role: Sends formatted article summary message back to user chat  
  - Config:  
    - Message text includes emojis, title, summary, and up to 5 key points rendered with numbered emojis  
    - Uses expressions to conditionally show 4th and 5th key points if present  
    - Appends original article URL at bottom  
    - Chat ID sourced from Extract URL node  
    - Attribution disabled to avoid extra bot signature  
  - Input: Structured summary JSON from Summarize Article node  
  - Output: Telegram message sent to user  
  - Retry: Enabled  

- **Save to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends or updates a Google Sheet with summary data for record keeping  
  - Config:  
    - Disabled by default  
    - Columns: URL, current date/time, title, summary  
    - Requires Google Sheets OAuth2 credentials and a specific Sheet ID + Sheet name  
  - Input: Structured summary JSON and URL data  
  - Output: Data appended to Google Sheet  
  - Edge Cases: Credential expiry, Sheet ID errors, quota limits  

- **Sticky Note5**  
  - Explains output formatting and optional saving  

---

#### 1.5 Documentation and Metadata

**Nodes Involved:**  
- Sticky Note (large main documentation note)

**Node Details:**  

- **Sticky Note**  
  - Contains detailed documentation on workflow purpose, setup instructions, credential requirements, customization tips, and requirements  
  - Covers: Telegram bot creation, API keys, AI model configuration, output parser schema, storage options, language support  

---

### 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                         | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                                                     |
|--------------------------------|---------------------------------------|---------------------------------------|------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| After message is received       | Telegram Trigger                      | Entry point, receives Telegram messages | -                            | Check if URL                     |                                                                                                                                 |
| Check if URL                   | Filter                                | Filters messages to URLs only          | After message is received     | Extract URL                      | ## Filter URLs Only processes messages that start with http:// or https:// Non-URL messages are ignored.                      |
| Extract URL                   | Set                                   | Extract URL and chat ID from message   | Check if URL                 | Fetch Article Markdown           |                                                                                                                                 |
| Fetch Article Markdown         | HTTP Request                         | Fetch clean article markdown content   | Extract URL                  | Summarize Article, Notify an error while fetching | ## Fetch article Uses Jina AI to convert webpage to clean markdown. Removes navigation, ads, and clutter automatically. If fetch fails, sends error message. |
| Notify an error while fetching | Telegram                             | Notify user if article fetch fails     | Fetch Article Markdown (error) | -                              |                                                                                                                                 |
| OpenRouter Chat Model          | LangChain LM Chat OpenRouter         | Connects to Claude AI model via OpenRouter API | Summarize Article (ai_languageModel) | Summarize Article, Output Parser |                                                                                                                                 |
| Summarize Article             | LangChain Chain LLM                   | Sends prompt to AI, receives summary   | Fetch Article Markdown, OpenRouter Chat Model | Send Summary to Telegram, Save to Google Sheets, Notify an error while summarizing | ##  AI summary Sends markdown to llm model via OpenRouter. Returns structured JSON with: - Title - One-sentence summary - 3-5 key points Matches the article's original language. |
| Output Parser                 | LangChain Output Parser Structured    | Validates and fixes AI JSON output     | OpenRouter Chat Model        | Summarize Article               |                                                                                                                                 |
| Notify an error while summarizing | Telegram                             | Notify user if AI summarization fails  | Summarize Article (error)    | -                              |                                                                                                                                 |
| Send Summary to Telegram      | Telegram                             | Sends formatted summary back to user  | Summarize Article            | Save to Google Sheets (optional) | ## Send summary Formats the summary with emojis and sends to Telegram. Optionally saves to Google Sheets for tracking.          |
| Save to Google Sheets         | Google Sheets                        | Optionally save summary data to sheet | Send Summary to Telegram     | -                              |                                                                                                                                 |
| Sticky Note                  | Sticky Note                          | Documentation and instructions         | -                            | -                              | ## Article summarizer bot Send any URL to your Telegram bot and get an AI summary instantly. What it does - Receives URLs via Telegram - Fetches clean article content (removes ads, navbars) - Generates AI summary - Sends formatted summary back to Telegram How to set up 1. Create Telegram bot 2. Get API keys 3. Configure AI model and output parser format Requirements Telegram bot token OpenRouter API key or any other LLM How to customize - Change summary format: Edit prompt in "Summarize Article" node - Update Output Parser schema - Save to database: Enable Google Sheets or add Notion/Airtable - Different language: Modify prompt to force specific language |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use @BotFather to create a bot and get the bot token  
   - Add Telegram API credentials in n8n with the token  

2. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Trigger on "message" updates  
   - Use the Telegram credentials created  
   - Position: Entry node  

3. **Add Filter Node "Check if URL"**  
   - Type: Filter  
   - Condition: Message text matches regex `^https?://` (case sensitive, strict)  
   - Connect Telegram Trigger output to Filter input  

4. **Add Set Node "Extract URL"**  
   - Type: Set  
   - Assign two variables:  
     - `url` = `{{$json.message.text}}`  
     - `chat_id` = `{{$json.message.chat.id}}`  
   - Connect Filter "true" output to this node  

5. **Add HTTP Request Node "Fetch Article Markdown"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://r.jina.ai/{{$json.url}}`  
   - On Error: Set to continue error output (to allow error handling)  
   - Connect "Extract URL" output to this node  

6. **Add Telegram Node "Notify an error while fetching"**  
   - Type: Telegram  
   - Text: Static message about fetch failure reasons (paywall, invalid URL, etc.)  
   - Chat ID: `{{$json.chat_id}}` from "Extract URL" node  
   - Retry enabled  
   - Connect error output of "Fetch Article Markdown" to this node  

7. **Add LangChain LM Chat OpenRouter Node "OpenRouter Chat Model"**  
   - Type: LangChain LM Chat OpenRouter  
   - Model: `anthropic/claude-3.5-haiku`  
   - Credentials: OpenRouter API key (must have access to Claude model)  
   - Position: After Fetch Article Markdown success output  

8. **Add LangChain Output Parser Node "Output Parser"**  
   - Type: LangChain Output Parser Structured  
   - Schema (manual): JSON requiring `title` (string), `summary` (string), `key_points` (array of strings, 3-5 items)  
   - AutoFix enabled  
   - Connect OpenRouter node output to this node  

9. **Add LangChain Chain LLM Node "Summarize Article"**  
   - Type: LangChain Chain LLM  
   - Prompt:  
     ```
     Extract from this article and respond in the SAME LANGUAGE as the article:

     {{ $json.data }}

     Return JSON with: title (string), summary (1 sentence), key_points (array of 3-5 strings)

     IMPORTANT: If article is in Spanish, respond in Spanish. If English, respond in English. Match the article's language.
     ```  
   - Enable output parser, link to "Output Parser" node  
   - Max retries: 2  
   - Connect:  
     - Input from "Fetch Article Markdown" node (article content)  
     - AI language model input from "OpenRouter Chat Model" node  
     - AI output parser input from "Output Parser" node  

10. **Add Telegram Node "Send Summary to Telegram"**  
    - Type: Telegram  
    - Text template:  
      ```
      📰 Article summary

      Title: {{ $json.output.title }}

      💡 Summary: {{ $json.output.summary }}

      ━━━━━━━━━━━━━━━
      📌 Key points:

      1️⃣ {{ $json.output.key_points[0] }}

      2️⃣ {{ $json.output.key_points[1] }}

      3️⃣ {{ $json.output.key_points[2] }}

      {{ $json.output.key_points[3] ? '4️⃣ ' + $json.output.key_points[3] : '' }}

      {{ $json.output.key_points[4] ? '5️⃣ ' + $json.output.key_points[4] : '' }}

      ━━━━━━━━━━━━━━━
      🔗 {{ $('Extract URL').item.json.url }}
      ```  
    - Chat ID: `{{ $('Extract URL').item.json.chat_id }}`  
    - Disable append attribution  
    - Retry enabled  
    - Connect "Summarize Article" success output to this node  

11. **Add Telegram Node "Notify an error while summarizing"**  
    - Type: Telegram  
    - Text: Static message about summarization failure and advice  
    - Chat ID: `{{ $('Extract URL').item.json.chat_id }}`  
    - Retry enabled  
    - Connect "Summarize Article" error output to this node  

12. **Optional: Add Google Sheets Node "Save to Google Sheets"**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Document ID and Sheet Name set accordingly  
    - Columns: URL, current date/time, Title, Summary  
    - Connect success output of "Send Summary to Telegram" to this node  
    - Add Google Sheets OAuth2 credentials  

13. **Add Sticky Notes for documentation** (optional)  
    - Include notes explaining URL filtering, fetching, AI summarization, and output sending  

14. **Activate the workflow and test end-to-end**  
    - Send valid article URL in Telegram to bot  
    - Verify article summary is received  
    - Handle errors gracefully as per notifications  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                        | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| How to create a Telegram bot: message @BotFather, use /newbot, copy token.                                                                                                                                                                          | Telegram Bot setup instructions                   |
| Jina AI Reader API converts webpages to clean markdown, removing ads and clutter; useful for clean article extraction.                                                                                                                             | https://jina.ai/reader                             |
| OpenRouter API provides access to Claude AI models for natural language tasks.                                                                                                                                                                      | https://openrouter.ai                              |
| Customize summary format and output schema by editing the prompt and output parser nodes respectively.                                                                                                                                             | Workflow customization tips                        |
| Optionally save summaries to Google Sheets, Notion, or Airtable for tracking.                                                                                                                                                                        | Data storage options                               |
| Supports multi-language article summarization, currently handles English and Spanish with language matching in prompt.                                                                                                                             | Multilingual support                               |
| Error handling includes user notifications for fetch failures (paywall, invalid URLs) and summarization failures.                                                                                                                                   | User experience consideration                      |

---

*Disclaimer:* The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. The processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.