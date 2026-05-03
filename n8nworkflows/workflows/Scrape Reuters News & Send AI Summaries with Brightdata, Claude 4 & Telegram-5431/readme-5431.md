Scrape Reuters News & Send AI Summaries with Brightdata, Claude 4 & Telegram

https://n8nworkflows.xyz/workflows/scrape-reuters-news---send-ai-summaries-with-brightdata--claude-4---telegram-5431


# Scrape Reuters News & Send AI Summaries with Brightdata, Claude 4 & Telegram

---

### 1. Workflow Overview

This workflow automates the process of scraping Reuters news articles based on user-submitted keywords, processing and summarizing the content using Anthropic's Claude 4 large language model (LLM), and delivering summarized news updates via Telegram alerts. It is designed for users who want timely, AI-enhanced news intelligence on specific topics.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Captures user inputs via an n8n form trigger, collecting keywords and preferred news sorting type.

- **1.2 News Scraping Trigger & Monitoring:** Uses Brightdata API to trigger a Reuters news scraping snapshot based on the keywords, then polls Brightdata for scraping progress and fetches data once ready.

- **1.3 AI Processing & Data Formatting:** Sends scraped raw articles to Anthropic Claude 4 model for intelligent extraction and cleaning. Processes the AI output to produce structured, clean article summaries.

- **1.4 Notification Dispatch:** Sends the final cleaned news summaries to a designated Telegram chat.

Supporting these blocks are utility nodes such as a sleep timer to manage polling intervals and various sticky notes documenting node roles.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block handles the initial user input by triggering the workflow upon form submission. It collects keywords to search news for and the preferred sorting order (e.g., newest, oldest, relevance).

**Nodes Involved:**  
- On form submission  
- Sticky Note (description of the trigger)

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Starts workflow when a user submits a form with fields "Keywords" (text, required) and "News Type" (dropdown with options: newest, oldest, relevance, required).  
  - *Key Configurations:*  
    - `formTitle`: "Reuters News Intelligence"  
    - Fields:  
      - Keywords (required)  
      - News Type dropdown (required)  
  - *Inputs:* None (webhook trigger)  
  - *Outputs:* Form data as JSON to next nodes  
  - *Potential Failures:* Form webhook misconfiguration, missing required inputs, or network issues.  
  - *Sticky Note:* Describes this node as the trigger for workflow start.

- **Sticky Note** (attached to the above)  
  - Explains that this node triggers the workflow on form submission with required fields "Keywords" and "News Type."

---

#### 2.2 News Scraping Trigger & Monitoring

**Overview:**  
This block initiates a scraping snapshot on Brightdata’s Reuters News dataset with the user’s keyword, monitors the scraping progress, waits if necessary, and fetches the snapshot data when ready.

**Nodes Involved:**  
- HTTP Request (Trigger scraping snapshot)  
- Check Scraping Status (poll snapshot status)  
- sleep tool (wait 1 minute before rechecking status)  
- Fetch Snapshot Data (retrieve scraped articles)  
- Sticky Notes describing the role of each node

**Node Details:**

- **HTTP Request**  
  - *Type:* HTTP Request Tool  
  - *Role:* Sends a POST request to Brightdata API to trigger a new Reuters news scraping snapshot with the user’s keyword(s).  
  - *Key Config:*  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Method: POST  
    - JSON Body: Contains keyword input, sessionId, action, chatInput formatted for Brightdata collector. Example keyword: "Gas shocks"  
    - Query Params: dataset_id (Reuters dataset), include_errors=true, type=discover_new, discover_by=keyword, limit_per_input=2  
    - Header: Authorization with Brightdata API key  
  - *Inputs:* User form data (keywords)  
  - *Outputs:* snapshot_id to be used in next nodes  
  - *Potential Failures:* Authentication errors, invalid API key, malformed request body, network timeouts.  
  - *Sticky Note:* Describes as triggering a new Reuters news scraping snapshot.

- **Check Scraping Status**  
  - *Type:* HTTP Request Tool  
  - *Role:* Queries Brightdata’s API to check if the previously triggered snapshot is ready or still running.  
  - *Key Config:*  
    - URL dynamically constructed using snapshot_id, e.g. `https://api.brightdata.com/datasets/v3/progress/{snapshot_id}`  
    - Query param: format=json  
    - Header: Authorization with Brightdata API key  
  - *Inputs:* snapshot_id from previous node  
  - *Outputs:* Snapshot status (e.g., "ready", "running")  
  - *Potential Failures:* Snapshot ID invalid or expired, API errors, network issues.  
  - *Sticky Note:* Describes this node as verifying snapshot readiness.

- **sleep tool**  
  - *Type:* Tool Code (JavaScript)  
  - *Role:* Waits for 60 seconds before rechecking snapshot status to allow for scraping completion.  
  - *Key Config:*  
    - JS code: `await new Promise(resolve => setTimeout(resolve, 60 * 1000)); return "1 minute wait done";`  
  - *Inputs:* Triggered if snapshot status is still "running"  
  - *Outputs:* Signal to retry status check after delay  
  - *Potential Failures:* Node execution errors, or workflow timeout if waiting too long.  
  - *Sticky Note:* Indicates this node is used to wait 1 minute before polling again.

- **Fetch Snapshot Data**  
  - *Type:* HTTP Request Tool  
  - *Role:* Fetches the actual scraped news articles from Brightdata once snapshot status is "ready".  
  - *Key Config:*  
    - URL dynamically constructed using snapshot_id, e.g. `https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}`  
    - Query param: format=json  
    - Header: Authorization with Brightdata API key  
  - *Inputs:* snapshot_id confirmed ready  
  - *Outputs:* Raw scraped news data JSON  
  - *Potential Failures:* Snapshot data not found, API errors, malformed JSON responses.  
  - *Sticky Note:* Describes this node as retrieving data from Brightdata once ready.

---

#### 2.3 AI Processing & Data Formatting

**Overview:**  
This block sends the raw scraped data to Anthropic Claude 4 LLM via LangChain to extract, clean, and structure article details into a readable format. It then formats the AI output into a clean JSON array of news articles.

**Nodes Involved:**  
- Anthropic Chat Model  
- MCP for Data Fetching through Anthropic (LangChain agent)  
- Data Formatting (Code node for JSON extraction and cleaning)  
- Sticky Notes explaining AI processing and formatting

**Node Details:**

- **Anthropic Chat Model**  
  - *Type:* LangChain Anthropic LLM node  
  - *Role:* Provides the underlying Claude 4 large language model used by the LangChain agent for language-based processing of scraped news.  
  - *Key Config:*  
    - Model: `claude-sonnet-4-20250514` (Claude 4 Sonnet)  
    - Credentials: Anthropic API key  
  - *Inputs:* Text prompts from LangChain agent node  
  - *Outputs:* AI-generated completions (news summaries and extraction)  
  - *Potential Failures:* API quota limits, authentication errors, model downtime, prompt errors.  
  - *Sticky Note:* Describes node as sending input to Claude for language processing.

- **MCP for Data Fetching through Anthropic**  
  - *Type:* LangChain Agent node  
  - *Role:* Orchestrates AI prompt to scrape latest Reuters news articles based on keywords and news type. Extracts specified fields (article_title, headline, description, content, article_url) ensuring clean output without duplicates.  
  - *Key Config:*  
    - Prompt text dynamically includes user keyword and news type  
    - Output JSON structured per article with specified fields  
  - *Inputs:* User form data and output from Brightdata scraping nodes  
  - *Outputs:* Raw AI-generated text with embedded JSON block of articles  
  - *Potential Failures:* Prompt misinterpretation by AI, incomplete extraction, model API errors.  
  - *Sticky Note:* Describes this node as using LangChain AI agent for orchestrated extraction.

- **Data Formatting**  
  - *Type:* Code (JavaScript) node  
  - *Role:* Parses the AI output text to extract JSON embedded inside triple backticks, cleans and maps fields into simplified article objects (heading, article_url, description, content), and outputs each article as individual item for further processing.  
  - *Key Config:*  
    - JS code extracts JSON from AI output pattern ```json ... ```  
    - Throws error if JSON block missing or malformed  
  - *Inputs:* Raw AI output text from LangChain agent node  
  - *Outputs:* Array of cleaned article JSON objects as separate items  
  - *Potential Failures:* Malformed AI output, JSON parse errors, missing fields.  
  - *Sticky Note:* Describes node as extracting and cleaning article fields.

---

#### 2.4 Notification Dispatch

**Overview:**  
This block sends the formatted news article summaries as messages to a specified Telegram chat to notify users.

**Nodes Involved:**  
- Telegram  
- Sticky Note describing Telegram node usage

**Node Details:**

- **Telegram**  
  - *Type:* Telegram Node  
  - *Role:* Sends article summaries as Telegram messages to a configured chat ID using a Telegram Bot.  
  - *Key Config:*  
    - Text composed by concatenating article_url, heading, description, content fields from each article JSON  
    - Chat ID: configured with demo placeholder `DEMO_CHAT_ID`  
    - Credentials: Telegram Bot API key  
  - *Inputs:* Cleaned article JSON objects from Data Formatting node  
  - *Outputs:* Confirmation of message sent (optional)  
  - *Potential Failures:* Invalid chat ID, Telegram API rate limits, network issues, bot permissions.  
  - *Sticky Note:* Describes node as sending cleaned news data to Telegram chat.

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                                      | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                      |
|--------------------------------|--------------------------------|-----------------------------------------------------|--------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                   | Triggers workflow on user form submission           | None                           | MCP for Data Fetching through Anthropic | Trigger node that starts the workflow when a form is submitted with "Keywords" and "News Type". |
| MCP for Data Fetching through Anthropic | LangChain Agent               | Orchestrates AI extraction of news articles         | On form submission, HTTP Request, Check Scraping Status, Fetch Snapshot Data, sleep tool, Anthropic Chat Model | Data Formatting                     | Uses LangChain's AI agent to orchestrate actions based on input and other node outputs.          |
| Anthropic Chat Model           | LangChain LLM                 | Provides Claude 4 LLM completions                    | MCP for Data Fetching through Anthropic | MCP for Data Fetching through Anthropic | Sends the input to Claude (Anthropic's LLM) for language-based processing or prompt handling.    |
| HTTP Request                  | HTTP Request Tool              | Triggers Brightdata Reuters news scraping snapshot  | On form submission             | MCP for Data Fetching through Anthropic | Sends a request to BrightData to trigger a new Reuters news scraping snapshot by keyword.       |
| Check Scraping Status          | HTTP Request Tool              | Checks status of Brightdata snapshot                 | MCP for Data Fetching through Anthropic | MCP for Data Fetching through Anthropic | Verifies if the BrightData snapshot has finished processing and is ready to be fetched.         |
| sleep tool                    | Tool Code (JS)                 | Waits 1 minute before polling Brightdata again      | MCP for Data Fetching through Anthropic | MCP for Data Fetching through Anthropic | Waits for 1 minute before checking again to ensure snapshot has time to complete.                |
| Fetch Snapshot Data            | HTTP Request Tool              | Fetches scraped news articles from Brightdata       | MCP for Data Fetching through Anthropic | MCP for Data Fetching through Anthropic | Retrieves the actual data (articles) from BrightData once the snapshot is ready.                 |
| Data Formatting               | Code (JavaScript)              | Extracts and cleans AI output JSON into article objects | MCP for Data Fetching through Anthropic | Telegram                          | JavaScript code node to extract, clean, and structure only the required article fields.          |
| Telegram                      | Telegram                      | Sends cleaned news articles as Telegram messages    | Data Formatting               | None                              | Sends the final cleaned news article data to a specified Telegram chat.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node:**  
   - Type: Form Trigger  
   - Configure form with title "Reuters News Intelligence"  
   - Add two fields:  
     - "Keywords" (text, required)  
     - "News Type" (dropdown: newest, oldest, relevance, required)  
   - This node will start the workflow upon user submission.

2. **Create "HTTP Request" node to trigger Brightdata scraping:**  
   - Type: HTTP Request Tool  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Query Parameters:  
     - dataset_id: `gd_lyptx9h74wtlvpnfu` (Reuters dataset)  
     - include_errors: `true`  
     - type: `discover_new`  
     - discover_by: `keyword`  
     - limit_per_input: `2`  
   - Headers: Authorization with Brightdata API key (credential setup required)  
   - Body (JSON): Include user keyword(s) and sorting preference, e.g.:  
     ```json
     [
       {
         "keyword": "Gas shocks",
         "sort": "newest"
       }
     ]
     ```  
   - Connect "On form submission" output to this node.

3. **Create "Check Scraping Status" node:**  
   - Type: HTTP Request Tool  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{snapshot_id}` (use expression to inject snapshot_id from previous node)  
   - Query Parameter: format=json  
   - Header: Authorization with Brightdata API key  
   - This node checks if scraping snapshot is ready.

4. **Create "sleep tool" node:**  
   - Type: Tool Code (JavaScript)  
   - JS Code:  
     ```js
     await new Promise(resolve => setTimeout(resolve, 60 * 1000));
     return "1 minute wait done";
     ```  
   - Used to wait 1 minute between status checks.

5. **Set up looping/polling logic:**  
   - After "Check Scraping Status", if status is "running", connect to "sleep tool" then back to "Check Scraping Status" for retry.  
   - If status is "ready", proceed to next step.

6. **Create "Fetch Snapshot Data" node:**  
   - Type: HTTP Request Tool  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}` (inject snapshot_id)  
   - Query Parameter: format=json  
   - Header: Authorization with Brightdata API key  
   - Retrieves the scraped news articles.

7. **Create "Anthropic Chat Model" node:**  
   - Type: LangChain Anthropic LLM node  
   - Model: `claude-sonnet-4-20250514`  
   - Credentials: Anthropic API key setup required  
   - Used as the language model backend for LangChain agent.

8. **Create "MCP for Data Fetching through Anthropic" node:**  
   - Type: LangChain Agent  
   - Text Prompt:  
     ```
     Scrape the latest Reuters news articles based on the keyword: "{{ $json.Keywords }}".
     News Type:"{{ $json['News Type'] }}"
     For each matching article, return the following details in the output JSON:

     article_title: The title of the article
     headline: The headline shown on Reuters site
     description: The short summary or meta description of the article
     content: The full article content/body text
     article_url: The full URL to the article

     Note:- Make sure there is no duplication and that each field is clean and readable.
     ```  
   - Connect inputs: user form data, HTTP requests, and Anthropic Chat Model node  
   - Output: raw AI text including JSON block with articles

9. **Create "Data Formatting" node:**  
   - Type: Code (JavaScript)  
   - JS Code:  
     ```js
     const rawOutput = $json["output"];
     const matches = rawOutput.match(/```json\s+([\s\S]*?)\s+```/);
     if (!matches || matches.length < 2) {
       throw new Error("Could not find JSON block in output");
     }
     const articles = JSON.parse(matches[1]);
     const cleanedArticles = articles.map(article => ({
       heading: article.headline || article.article_title || "",
       article_url: article.article_url || "",
       description: article.description || "",
       content: article.content || ""
     }));
     return cleanedArticles.map(item => ({ json: item }));
     ```  
   - Parses AI output into cleaned article objects.

10. **Create "Telegram" node:**  
    - Type: Telegram  
    - Text: concatenate fields: `{{$json.article_url}}{{$json.heading}}{{$json.description}}{{$json.content}}`  
    - Chat ID: your Telegram chat ID (credential setup required)  
    - Credentials: Telegram Bot API key  
    - Sends formatted news articles as Telegram messages.

11. **Connect nodes sequentially:**  
    - On form submission → HTTP Request (trigger) → Check Scraping Status →  
      If "running": sleep tool → back to Check Scraping Status  
      If "ready": Fetch Snapshot Data → MCP for Data Fetching through Anthropic → Data Formatting → Telegram

12. **Credential Setup:**  
    - Brightdata API key: create and configure in n8n credentials  
    - Anthropic API key: create and configure in n8n credentials  
    - Telegram Bot API key: create Telegram bot, obtain token, configure in n8n credentials

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow uses Brightdata’s Reuters News dataset to perform real-time scraping based on user keywords.              | Brightdata API docs: https://brightdata.com/docs/api          |
| Anthropic Claude 4 model is leveraged via LangChain integration for advanced summarization and extraction.             | Anthropic API and LangChain integration in n8n                |
| Telegram Bot API is used for real-time delivery of news summaries.                                                    | Telegram Bot API docs: https://core.telegram.org/bots/api     |
| Workflow includes a polling mechanism with delay (sleep tool) to avoid excessive API requests while waiting for data. | Best practice for API rate limiting and polling               |
| Form trigger input validation ensures required fields are provided before workflow execution.                          | n8n Form Trigger node documentation                            |

---

This completes a comprehensive, structured, and self-contained reference for the "Scrape Reuters News & Send AI Summaries with Brightdata, Claude 4 & Telegram" workflow.