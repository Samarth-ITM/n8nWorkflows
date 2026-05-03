Filter Real-time News with Gemini AI and BrowserAct for Telegram Channels

https://n8nworkflows.xyz/workflows/filter-real-time-news-with-gemini-ai-and-browseract-for-telegram-channels-8915


# Filter Real-time News with Gemini AI and BrowserAct for Telegram Channels

### 1. Workflow Overview

This workflow automates the collection, filtering, and distribution of real-time news articles to Telegram channels using AI. Its primary purpose is to scrape the latest news via a web scraping API, filter articles based on user-defined keywords using Google Gemini AI, and then send selected news as rich media messages to Telegram.

Logical blocks:

- **1.1 Trigger & Scraping Initiation**: Scheduled automatic start of the news scraping task using the BrowserAct API.
- **1.2 Scraping Monitoring & Completion Check**: Polling the scraping task status until it's finished.
- **1.3 AI-based News Filtering**: Using Google Gemini AI agent to filter news articles by keywords.
- **1.4 Output Formatting & Telegram Distribution**: Formatting AI output and sending news to Telegram channels as photo messages.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Scraping Initiation

- **Overview:**  
  This block triggers the workflow on a scheduled interval and initiates a news scraping task via the BrowserAct API.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Run WorkFlow  
  - Check For Errors  
  - Wait1  

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Time-based trigger node to start the workflow periodically every hour.  
    - *Config:* Interval set to trigger every hour.  
    - *Connections:* Outputs to "Run WorkFlow".  
    - *Edge Cases:* Misconfigured intervals can cause missed or excessive triggering.

  - **Run WorkFlow**  
    - *Type & Role:* HTTP Request node sending POST request to BrowserAct API to start the scraping task.  
    - *Config:*  
      - URL: `https://api.browseract.com/v2/workflow/run-task`  
      - Method: POST  
      - Body: Includes `workflow_id` parameter (must be set to your BrowserAct workflow ID).  
      - Authentication: HTTP Bearer token (BrowserAct API key).  
      - Retry enabled on failure.  
    - *Connections:* Outputs to "Check For Errors".  
    - *Edge Cases:*  
      - Authentication failure if API key is invalid.  
      - Network or API downtime leading to failed requests.  
      - Missing or incorrect workflow_id causes task not to start.

  - **Check For Errors**  
    - *Type & Role:* If node checking if the scraping task start returned errors.  
    - *Config:* Checks if JSON `error` exists or if `id` is null.  
    - *Connections:*  
      - If no errors: outputs to "Get WorkFlow Data".  
      - If errors: outputs back to "Wait1" node to retry.  
    - *Edge Cases:*  
      - Expression failures if response format changes.  
      - Infinite loops if errors persist and wait time is too short.

  - **Wait1**  
    - *Type & Role:* Wait node pausing workflow for 1 minute before retrying.  
    - *Config:* Wait for 1 minute.  
    - *Connections:* Loops back to "Run WorkFlow".  
    - *Edge Cases:*  
      - Too short wait may cause rapid retries and API rate limits.  
      - Wait node failure is rare but possible.

#### 1.2 Scraping Monitoring & Completion Check

- **Overview:**  
  This block periodically checks the status of the scraping task until it is marked as finished.

- **Nodes Involved:**  
  - Get WorkFlow Data  
  - Check For "Finished" Status  
  - Wait  

- **Node Details:**

  - **Get WorkFlow Data**  
    - *Type & Role:* HTTP Request node to query BrowserAct API for the status of the scraping task.  
    - *Config:*  
      - URL: `https://api.browseract.com/v2/workflow/get-task`  
      - Method: GET  
      - Query Parameter: `task_id` taken from previous node’s JSON `id`.  
      - Authentication: HTTP Bearer token (BrowserAct API key).  
    - *Connections:* Outputs to "Check For \"Finished\" Status".  
    - *Edge Cases:*  
      - API errors or invalid task_id causing missing or malformed data.  
      - Network issues.

  - **Check For "Finished" Status**  
    - *Type & Role:* If node evaluating if the scraping task is finished.  
    - *Config:*  
      - Checks no error in response.  
      - Checks if `status` equals "finished".  
    - *Connections:*  
      - If finished: outputs to "KeyWords filtering".  
      - If not finished: outputs to "Wait".  
    - *Edge Cases:*  
      - Status field missing or values other than expected.  
      - Infinite loops if task never finishes.

  - **Wait**  
    - *Type & Role:* Wait node pausing the workflow before retrying status check.  
    - *Config:* Wait time unit is minutes (default amount unspecified but likely >0).  
    - *Connections:* Loops back to "Get WorkFlow Data".  
    - *Edge Cases:*  
      - Similar to Wait1, improper timing can cause rate limits or long delays.

#### 1.3 AI-based News Filtering

- **Overview:**  
  Once the scraping task is finished, this block uses Google Gemini AI to filter news headlines by the user-defined keywords and outputs structured results.

- **Nodes Involved:**  
  - Google Gemini  
  - Structured Output Parser  
  - KeyWords filtering  

- **Node Details:**

  - **Google Gemini**  
    - *Type & Role:* AI language model node calling Google Gemini (PaLM) API to process news data.  
    - *Config:*  
      - Uses credentials for Google Palm API.  
      - No specific options set, relies on prompt and input configuration.  
    - *Connections:* Outputs to "Structured Output Parser".  
    - *Edge Cases:*  
      - Authentication failures if API key is invalid.  
      - API rate limits or timeouts.  
      - Unexpected AI output format.

  - **Structured Output Parser**  
    - *Type & Role:* Output parser node ensuring AI output matches the expected JSON schema.  
    - *Config:*  
      - JSON schema example expects an array "Matched_News" with objects containing "Headline", "Pic", and "Url" as strings.  
    - *Connections:* Outputs parsed data to "KeyWords filtering".  
    - *Edge Cases:*  
      - AI output not matching schema causing parsing errors.

  - **KeyWords filtering**  
    - *Type & Role:* AI agent node with a defined prompt to analyze AI output, filter headlines by keywords, and produce final filtered JSON.  
    - *Config:*  
      - Takes input parameters including keywords split by ";" and ",".  
      - Checks if headlines contain or relate to keywords.  
      - Outputs JSON with "Matched_News" containing filtered articles.  
      - Uses structured output parser for validation.  
    - *Connections:* Outputs to "Code - Clean Output".  
    - *Edge Cases:*  
      - Keyword parsing errors if input format is incorrect.  
      - Filtering logic may miss relevant news or include false positives.

#### 1.4 Output Formatting & Telegram Distribution

- **Overview:**  
  This block formats the filtered news data into a readable structure and sends the news as photo messages with captions to a specified Telegram channel.

- **Nodes Involved:**  
  - Code - Clean Output  
  - Send a News Photo To Telegram  

- **Node Details:**

  - **Code - Clean Output**  
    - *Type & Role:* Code node formatting AI JSON output into individual items for Telegram.  
    - *Config:*  
      - JavaScript code extracts `Matched_News` array and converts each item to `{ json: { text: item } }`.  
    - *Connections:* Outputs to "Send a News Photo To Telegram".  
    - *Edge Cases:*  
      - Code errors if input JSON is malformed or missing keys.

  - **Send a News Photo To Telegram**  
    - *Type & Role:* Telegram node sending photo messages with captions.  
    - *Config:*  
      - Operation: sendPhoto  
      - Photo URL taken from `text.Url`  
      - Caption includes `text.Headline` and the `Url`.  
      - Chat ID set to a Telegram channel (must be configured).  
      - Uses Telegram API credentials.  
    - *Connections:* Terminal node.  
    - *Edge Cases:*  
      - Invalid Telegram credentials or chat ID causes sending failure.  
      - Improper URL or missing image may cause Telegram errors.  
      - Telegram API rate limits.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                    | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                                     |
|----------------------------|---------------------------------------|----------------------------------|-----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | scheduleTrigger                       | Start workflow on schedule       | —                     | Run WorkFlow                | "## 1. Trigger the News Scraper\n\nThis workflow runs on a set schedule to automatically trigger a web scraping task via the BrowserAct API. This allows you to continuously collect up-to-date news data.\n\n### Don't forget to add your BrowserAct Workflow ID." |
| Run WorkFlow               | httpRequest                          | Start BrowserAct scraping task   | Schedule Trigger, Wait1 | Check For Errors            | Same as above                                                                                                                  |
| Check For Errors           | if                                  | Check for errors after starting task | Run WorkFlow           | Get WorkFlow Data, Wait1    | Same as above                                                                                                                  |
| Wait1                     | wait                                | Pause before retrying start task | Check For Errors       | Run WorkFlow                | Same as above                                                                                                                  |
| Get WorkFlow Data          | httpRequest                         | Get scraping task status         | Check For Errors       | Check For "Finished" Status | "## 2. Wait for Scraping to Finish\n\nThese nodes check the status of the scraping task. The `If` node determines if the task is complete. If it's still running, the `Wait` node pauses the workflow for a set period before retrying to check the status." |
| Check For "Finished" Status | if                                  | Check if scraping is finished    | Get WorkFlow Data      | KeyWords filtering, Wait    | Same as above                                                                                                                  |
| Wait                      | wait                                | Pause before retrying status check | Check For "Finished" Status | Get WorkFlow Data           | Same as above                                                                                                                  |
| Google Gemini             | lmChatGoogleGemini                  | AI processing of news            | Check For "Finished" Status | Structured Output Parser  | "## 3. Use AI to Filter News by Keywords\n\nThis node uses an AI Agent to filter the scraped news articles. It checks each article headline against your specified keywords to find the most relevant content. It's configured with a **Structured Output Parser** to ensure the results are always in the correct format.\n\n### Don't forget to Connect Your Gemini." |
| Structured Output Parser  | outputParserStructured              | Parse AI output                  | Google Gemini          | KeyWords filtering          | Same as above                                                                                                                  |
| KeyWords filtering        | agent                              | Filter news by keywords         | Structured Output Parser, Check For "Finished" Status | Code - Clean Output         | Same as above                                                                                                                  |
| Code - Clean Output       | code                               | Format AI output for Telegram   | KeyWords filtering     | Send a News Photo To Telegram | "## 4. Send News to Telegram\n\nThe scraped data is sometimes hard to read. A **Code** node is used here to transform the AI's output into a more readable format. The **Telegram** node then sends the final, filtered news articles directly to you as a photo message with a caption and link.\n\n### Don't forget to Config your Telegram Api & Telegram Channel ID." |
| Send a News Photo To Telegram | telegram                          | Send news photos to Telegram    | Code - Clean Output    | —                           | Same as above                                                                                                                  |
| Wait                      | wait                               | Pause node in scraping status loop | Check For "Finished" Status | Get WorkFlow Data           | See above                                                                                                                     |
| Sticky Note-Intro          | stickyNote                         | Workflow introduction and summary | —                     | —                           | "## Try It Out!\n### This n8n template automates news content marketing by scraping news and sending relevant articles to Telegram using an AI Agent.\n\n### How it works\n* The workflow is triggered automatically on a schedule.\n* It uses an **HTTP Request** node to start a web scraping task with the **BrowserAct** API to collect the latest news.\n* A series of **If** and **Wait** nodes monitor the scraping job until it's finished.\n* An **AI Agent** node, powered by **Google Gemini**, processes the headlines and filters the news based on a list of keywords you define.\n* A **Code** node then formats the AI's output into a clean format.\n* The final news articles are sent as rich media messages to **Telegram**, including the headline, a picture, and a link.\n\n### Requirements\n* **BrowserAct** API account for web scraping\n* **Gemini** account for the AI Agent\n* **BrowserAct** **“News Content Marketing Automation”** Template\n* **Telegram** credentials for sending messages\n\n### Need Help?\nJoin the **BrowserAct** [Discord](https://discord.com/invite/UpnCKd7GaU) or Visit Our [Blog](https://www.browseract.com/blog)!" |
| Sticky Note-Scraping       | stickyNote                         | Info on scraping trigger        | —                     | —                           | Same as Schedule Trigger node note                                                                                           |
| Sticky Note-Wait           | stickyNote                         | Info on wait and status checking | —                     | —                           | Same as Get WorkFlow Data and Check For "Finished" Status notes                                                             |
| Sticky Note-AI             | stickyNote                         | Info on AI filtering             | —                     | —                           | Same as Google Gemini and KeyWords filtering node notes                                                                     |
| Sticky Note-Notifications  | stickyNote                         | Info on formatting and Telegram  | —                     | —                           | Same as Code - Clean Output and Telegram nodes notes                                                                         |
| Sticky Note-How to Use     | stickyNote                         | Instructions for setup           | —                     | —                           | "## How to use\n1- **Create BrowserAct Workflow:** Set up the **News Content Marketing Automation** template in your BrowserAct account.\n2- **Add BrowserAct Token:** Connect your BrowserAct account credentials to the **HTTP Request** inside **Run Node**.\n3- **Update Workflow ID:** Change the `workflow_id` value in the **HTTP Request** inside **Run Node** to match the one from your BrowserAct workflow.\n4- **Connect Gemini:** Add your Google Gemini credentials to the **AI Agent** node.\n5- **Configure Telegram:** Connect your Telegram account and add your Channel ID to the **Send Node** node." |
| Sticky Note-How to Use1    | stickyNote                         | Additional help video links      | —                     | —                           | "## Need Help ?\n*  [How to Find Your BrowseAct API Key & Workflow ID](https://www.youtube.com/watch?v=pDjoZWEsZlE)\n* [How to Connect n8n to Browseract](https://www.youtube.com/watch?v=RoYMdJaRdcQ)\n* [How to Use & Customize BrowserAct Templates](https://www.youtube.com/watch?v=CPZHFUASncY)" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**  
   - Set to trigger every 1 hour (interval hours = 1).  
   - This starts the workflow automatically on schedule.

3. **Add an `HTTP Request` node named "Run WorkFlow":**  
   - Method: POST  
   - URL: `https://api.browseract.com/v2/workflow/run-task`  
   - Body Parameters: JSON with one parameter `workflow_id` set to your BrowserAct workflow ID (e.g., `"52940169781736034"`).  
   - Authentication: HTTP Bearer Auth with your BrowserAct API token.  
   - Enable "Retry On Fail".  
   - Connect `Schedule Trigger` → `Run WorkFlow`.

4. **Add an `If` node named "Check For Errors":**  
   - Condition: Check if `$json.error` does not exist AND `$json.id` is not "null".  
   - True output connects to next step.  
   - False output connects to a wait retry.  
   - Connect `Run WorkFlow` → `Check For Errors`.

5. **Add a `Wait` node named "Wait1":**  
   - Set wait time to 1 minute.  
   - Connect False output from `Check For Errors` → `Wait1`.  
   - Connect `Wait1` → `Run WorkFlow` (loop to retry start request).

6. **Add an `HTTP Request` node named "Get WorkFlow Data":**  
   - Method: GET  
   - URL: `https://api.browseract.com/v2/workflow/get-task`  
   - Query Parameter: `task_id` set to `{{$json.id}}` from previous node.  
   - Authentication: HTTP Bearer Auth with BrowserAct API token.  
   - Connect True output of `Check For Errors` → `Get WorkFlow Data`.

7. **Add an `If` node named "Check For \"Finished\" Status":**  
   - Condition: Check `$json.error` does not exist AND `$json.status` equals "finished".  
   - True output connects to AI processing.  
   - False output connects to wait and status polling.  
   - Connect `Get WorkFlow Data` → `Check For "Finished" Status`.

8. **Add a `Wait` node named "Wait":**  
   - Set wait time (e.g., few minutes to avoid rapid polling).  
   - Connect False output of `Check For "Finished" Status` → `Wait`.  
   - Connect `Wait` → `Get WorkFlow Data` (loop to poll status).

9. **Add a `Google Gemini` node:**  
   - Use Google Palm API credentials.  
   - Connect True output of `Check For "Finished" Status` → `Google Gemini`.  
   - This node processes the scraped news headlines with AI.

10. **Add a `Structured Output Parser` node:**  
    - Set JSON schema example to expect:  
      ```json
      {
        "Matched_News": [
          {
            "Headline": "<String>",
            "Pic": "<String>",
            "Url": "<String>"
          }
        ]
      }
      ```  
    - Connect `Google Gemini` AI output to this parser.

11. **Add an `Agent` node named "KeyWords filtering":**  
    - Configure prompt to extract keywords from input parameters, split by ";" and ",".  
    - Compare each news headline to keywords and output JSON with matched news.  
    - Set to use structured output parser.  
    - Connect `Structured Output Parser` → `KeyWords filtering`.

12. **Add a `Code` node named "Code - Clean Output":**  
    - Insert JavaScript code to transform `Matched_News` array into individual JSON items with a `text` key.  
    - Connect `KeyWords filtering` → `Code - Clean Output`.

13. **Add a `Telegram` node named "Send a News Photo To Telegram":**  
    - Operation: sendPhoto  
    - File URL: `{{$json.text.Url}}`  
    - Caption: `{{$json.text.Headline}}\n\n{{$json.text.Url}}`  
    - Chat ID: your Telegram channel or chat ID (e.g., `@yourchannel`).  
    - Credentials: Telegram API credentials with bot token.  
    - Connect `Code - Clean Output` → `Send a News Photo To Telegram`.

14. **Add sticky notes for documentation and instructions as needed.**

15. **Activate the workflow and test end-to-end.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses the BrowserAct API and Google Gemini AI for automated news scraping and filtering.                               | Workflow core technologies                                                                         |
| BrowserAct Discord for support: https://discord.com/invite/UpnCKd7GaU                                                               | Support community                                                                                |
| BrowserAct blog with tutorials and updates: https://www.browseract.com/blog                                                         | Additional documentation and examples                                                           |
| How to Find Your BrowserAct API Key & Workflow ID (YouTube): https://www.youtube.com/watch?v=pDjoZWEsZlE                             | Setup help                                                                                       |
| How to Connect n8n to BrowserAct (YouTube): https://www.youtube.com/watch?v=RoYMdJaRdcQ                                              | Setup help                                                                                       |
| How to Use & Customize BrowserAct Templates (YouTube): https://www.youtube.com/watch?v=CPZHFUASncY                                   | Setup help                                                                                       |
| Ensure Google Gemini (PaLM) API credentials are correctly configured in n8n for AI nodes.                                            | Credential requirement                                                                           |
| Telegram bot token and channel ID must be correctly set for Telegram node to send messages successfully.                            | Credential requirement                                                                           |
| Be aware of API rate limits on BrowserAct, Google Gemini, and Telegram which may require adjusting wait timers and retry behavior. | Operational consideration                                                                       |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, a workflow automation tool. This process strictly complies with applicable content policies and contains no illegal or protected elements. All handled data is legal and publicly accessible.