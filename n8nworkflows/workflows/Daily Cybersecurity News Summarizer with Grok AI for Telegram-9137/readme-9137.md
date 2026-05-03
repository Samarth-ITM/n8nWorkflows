Daily Cybersecurity News Summarizer with Grok AI for Telegram

https://n8nworkflows.xyz/workflows/daily-cybersecurity-news-summarizer-with-grok-ai-for-telegram-9137


# Daily Cybersecurity News Summarizer with Grok AI for Telegram

### 1. Workflow Overview

This workflow automates the daily collection, summarization, and Telegram posting of cybersecurity news articles. It targets cybersecurity professionals and enthusiasts who want concise daily updates aggregated from various sources and summarized by AI.

The workflow is logically divided into these blocks:

- **1.1 Feed Fetch and Scraping (Source Aggregation):** Scheduled retrieval of RSS feeds from cybersecurity news sites, fetching full article contents via HTTP requests, and filtering relevant content.
- **1.2 Sponsored Content Filtering:** Removal of sponsored or promotional posts to maintain content quality.
- **1.3 AI Summarization:** Use of Grok AI (OpenRouter Chat Model) and LangChain agent nodes to generate concise two-sentence summaries of news items, enhanced by session memory.
- **1.4 Telegram Posting Loop:** Iterative sending of summarized news articles with images and metadata to a Telegram channel, with controlled pacing (wait between messages).

---

### 2. Block-by-Block Analysis

#### 2.1 Feed Fetch and Scraping (Source Aggregation)

**Overview:**  
This block initiates the workflow daily at 9 AM, reads multiple cybersecurity RSS feeds, fetches full HTML content for each news article, and filters out irrelevant HTML elements, preparing clean content for AI summarization.

**Nodes Involved:**  
- 9 AM - Schedule Trigger  
- Bleeping Computer Security Bulletin (RSS Feed Read)  
- HTTP Request  
- Filter Body from Full HTML (Set)  
- Filter Image Links From Body (Set)  
- Merge

**Node Details:**

- **9 AM - Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at 9:00 AM daily.  
  - Parameters: Triggers once daily at hour 9.  
  - Connections: Outputs to Bleeping Computer Security Bulletin.  
  - Edge Cases: If n8n instance is down at trigger time, no run occurs; no retry for missed runs.  

- **Bleeping Computer Security Bulletin**  
  - Type: RSS Feed Read  
  - Role: Reads RSS feed from "https://www.bleepingcomputer.com/feed/" to retrieve recent cybersecurity articles.  
  - Parameters: RSS URL, default options.  
  - Connections: Outputs to HTTP Request, Merge, and AI Agent nodes.  
  - Edge Cases: RSS feed downtime or format changes could cause failures or empty data.  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Retrieves full HTML content of the article using the URL from the RSS item (`{{ $json.link }}`).  
  - Parameters: URL dynamically set from RSS feed item link; no additional options.  
  - Connections: Outputs to Filter Body from Full HTML.  
  - Edge Cases: HTTP errors (404, 500), timeouts, or invalid URLs could cause failures.  

- **Filter Body from Full HTML**  
  - Type: Set  
  - Role: Extracts and sets the main body content from the full HTML response.  
  - Parameters: No explicit assignments visible in JSON; presumably cleans or extracts specific HTML parts.  
  - Connections: Outputs to Filter Image Links From Body.  
  - Edge Cases: HTML structure changes may cause extraction errors or incomplete content.  

- **Filter Image Links From Body**  
  - Type: Set  
  - Role: Filters out image URLs from the article body to isolate media for posting.  
  - Parameters: Sets `data` field to an empty string initially, likely later replaced with extracted image URLs.  
  - Connections: Outputs to Merge.  
  - Edge Cases: Articles without images may result in empty `data`, which downstream nodes must handle gracefully.  

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from multiple nodes by position to assemble enriched article data (content, images, metadata).  
  - Parameters: Combine mode by position with three inputs expected.  
  - Connections: Inputs from Sponsored Removal (filtered articles), Filter Image Links From Body, and AI Agent (summarization). Outputs to Sponsored Removal.  
  - Edge Cases: Mismatch in the number of items from inputs may cause data alignment issues.

---

#### 2.2 Sponsored Content Filtering

**Overview:**  
Removes articles identified as sponsored content based on the creator field to ensure only organic news is processed and posted.

**Nodes Involved:**  
- Sponsored Removal (If)  

**Node Details:**

- **Sponsored Removal**  
  - Type: If  
  - Role: Checks if the `creator` field contains the string "Sponsored".  
  - Parameters: Condition uses `contains` operator on `{{ $json.creator }}` field.  
  - Connections: True branch drops items; False branch passes items to Loop Over Items node.  
  - Edge Cases: If creator field is missing or malformed, condition may fail or incorrectly pass sponsored content. Case sensitivity is enabled, so variants like "sponsored" lowercase won't match.  

---

#### 2.3 AI Summarization

**Overview:**  
Summarizes news articles into two short sentences using Grok AI and maintains contextual memory to improve summarization consistency.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- OpenRouter Chat Model (OpenAI-compatible model)  
- Simple Memory (LangChain Memory Buffer Window)

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Processes input text to generate summaries based on the prompt.  
  - Parameters: Prompt set to "Simplify the {{ $json.contentSnippet }} in 2 short sentences."  
  - Connections: Receives AI language model input from OpenRouter Chat Model and memory input from Simple Memory. Outputs to Merge and further workflow.  
  - Edge Cases: Input text too short or missing may result in incomplete summaries. AI API rate limits or errors can cause failures.  

- **OpenRouter Chat Model**  
  - Type: LangChain OpenRouter Chat Model  
  - Role: Provides Grok AI language model (x-ai/grok-4-fast:free) for summarization.  
  - Parameters: Model specified as Grok 4 Fast (free tier).  
  - Credentials: Requires OpenRouter API key configured.  
  - Connections: Outputs to AI Agent as AI language model source.  
  - Edge Cases: API quota limits, connectivity issues, or model deprecations may cause errors.  

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation/session memory keyed by the article publication date to provide context.  
  - Parameters: Session key based on `{{ $json.pubDate }}`, context window length 13.  
  - Connections: AI memory input to AI Agent.  
  - Edge Cases: Session key collisions or missing keys can impact memory effectiveness.

---

#### 2.4 Telegram Posting Loop

**Overview:**  
Iterates over filtered and summarized articles, posting each as a photo message with caption metadata to a Telegram channel, and waits 1 minute between posts to avoid rate limits or spamming.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Send a photo message (Telegram)  
- Wait 1 min (Wait)  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes items one by one or in batches to control the posting rate.  
  - Parameters: Default options; batch size likely 1 (implicit by connections).  
  - Connections: Input from Sponsored Removal (filtered articles), output to Send a photo message (main) and empty output (second) to mark completion.  
  - Edge Cases: Large numbers of items may cause delays or memory issues; no explicit batch size set in JSON.  

- **Send a photo message**  
  - Type: Telegram  
  - Role: Sends photo messages to Telegram channel @DailySecurityNewss with caption including title, creator, publication date, brief AI summary, categories, and article link.  
  - Parameters:  
    - Operation: sendPhoto  
    - File: `{{ $json.data }}` (image URL or content)  
    - Chat ID: @DailySecurityNewss  
    - Caption: Formatted with multiple dynamic fields from JSON.  
  - Credentials: Telegram API with PrivateMessageTesterBot_API credentials.  
  - Connections: Outputs to Wait 1 min.  
  - Edge Cases: Invalid or empty image data may cause message send failures; Telegram API limits or connectivity issues may cause errors.  

- **Wait 1 min**  
  - Type: Wait  
  - Role: Delays subsequent message sends by 1 minute to prevent spamming or hitting Telegram rate limits.  
  - Parameters: Wait 1 minute.  
  - Connections: Outputs back to Loop Over Items to process next item or complete.  
  - Edge Cases: Workflow pause may accumulate delay for large number of items.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                     | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                  |
|----------------------------------|----------------------------------|-----------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------|
| 9 AM - Schedule Trigger           | Schedule Trigger                 | Trigger workflow daily at 9 AM    | -                                | Bleeping Computer Security Bulletin | ## Feed Fetch on Scheduled time                                                             |
| Bleeping Computer Security Bulletin | RSS Feed Read                   | Read cybersecurity RSS feed       | 9 AM - Schedule Trigger           | HTTP Request, Merge, AI Agent    | ## Wesbite Scrapper                                                                          |
| HTTP Request                     | HTTP Request                    | Fetch full article HTML content   | Bleeping Computer Security Bulletin | Filter Body from Full HTML       | ## Wesbite Scrapper                                                                          |
| Filter Body from Full HTML        | Set                             | Extract main article body from HTML | HTTP Request                    | Filter Image Links From Body     | ## Wesbite Scrapper                                                                          |
| Filter Image Links From Body      | Set                             | Filter image links from body      | Filter Body from Full HTML        | Merge                           | ## Wesbite Scrapper                                                                          |
| Merge                           | Merge                           | Combine article data and AI output | Sponsored Removal, Filter Image Links From Body, AI Agent | Sponsored Removal               |                                                                                             |
| Sponsored Removal                | If                              | Remove sponsored content          | Merge                           | Loop Over Items (false branch)  | ## Sponsored posts Removal                                                                   |
| Loop Over Items                 | SplitInBatches                  | Iterate over articles for posting | Sponsored Removal                | Send a photo message, (end)      | ## Post all messages on telegram                                                             |
| Send a photo message             | Telegram                        | Send article photo and caption   | Loop Over Items                  | Wait 1 min                      | ## Post all messages on telegram                                                             |
| Wait 1 min                     | Wait                           | Wait 1 minute between posts       | Send a photo message             | Loop Over Items                 | ## Post all messages on telegram                                                             |
| AI Agent                       | LangChain Agent                 | Generate article summary          | Bleeping Computer Security Bulletin | Merge                          | ## Sumarize with AI                                                                          |
| OpenRouter Chat Model            | LangChain OpenRouter Chat Model | Provide Grok AI language model   | -                              | AI Agent                       | ## Sumarize with AI                                                                          |
| Simple Memory                  | LangChain Memory Buffer Window  | Maintain context memory for AI   | -                              | AI Agent (memory input)         | ## Sumarize with AI                                                                          |
| Sticky Note                     | Sticky Note                    | Comment block                    | -                              | -                              | ## Wesbite Scrapper                                                                          |
| Sticky Note1                    | Sticky Note                    | Comment block                    | -                              | -                              | ## Post all messages on telegram                                                             |
| Sticky Note2                    | Sticky Note                    | Comment block                    | -                              | -                              | ## Sponsored posts Removal                                                                   |
| Sticky Note3                    | Sticky Note                    | Comment block                    | -                              | -                              | ## Sumarize with AI                                                                          |
| Sticky Note4                    | Sticky Note                    | Comment block                    | -                              | -                              | ## Feed Fetch on Scheduled time                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: "9 AM - Schedule Trigger"  
   - Trigger: Daily at 9:00 AM (set `triggerAtHour` to 9).

2. **Create an RSS Feed Read node:**
   - Name: "Bleeping Computer Security Bulletin"  
   - RSS URL: `https://www.bleepingcomputer.com/feed/`  
   - Connect input from "9 AM - Schedule Trigger".

3. **Create an HTTP Request node:**
   - Name: "HTTP Request"  
   - URL: Set dynamically to `={{ $json.link }}` (article link from RSS item)  
   - Connect input from "Bleeping Computer Security Bulletin".

4. **Create a Set node to filter body from full HTML:**
   - Name: "Filter Body from Full HTML"  
   - Configure to extract or clean article body from HTTP response (custom logic or expressions as needed).  
   - Connect input from "HTTP Request".

5. **Create a Set node to filter image links from the body:**
   - Name: "Filter Image Links From Body"  
   - Initialize field `data` as an empty string or implement extraction logic for images from article body.  
   - Connect input from "Filter Body from Full HTML".

6. **Create a Merge node:**
   - Name: "Merge"  
   - Mode: Combine  
   - Combine by position with 3 inputs.  
   - Connect inputs from:  
     - "Sponsored Removal" (for filtered articles, see below)  
     - "Filter Image Links From Body" (image data)  
     - "AI Agent" (summary output)  

7. **Create an If node to remove sponsored content:**
   - Name: "Sponsored Removal"  
   - Condition: Check if `{{ $json.creator }}` contains "Sponsored" (case sensitive).  
   - Connect input from "Merge".  
   - False output connects to "Loop Over Items".  
   - True output is discarded (breaks flow).

8. **Create a SplitInBatches node:**
   - Name: "Loop Over Items"  
   - Default batch size (1 implied).  
   - Connect input from "Sponsored Removal" (false branch).

9. **Create a Telegram node to send photo messages:**
   - Name: "Send a photo message"  
   - Operation: sendPhoto  
   - Chat ID: `@DailySecurityNewss`  
   - File: `{{ $json.data }}` (image URL or media)  
   - Caption:  
     ```
     {{ $json.title }}    

     By - {{ $json.creator }}   
     Published on {{ $json.pubDate.match(/\d{1,2} \w{3} \d{4}/)[0] }}    

     Brief Summary - {{ $json.output }}    

     Categories -  {{ $json.categories }} 

     Read more - {{ $json.link }}
     ```  
   - Credentials: Set Telegram API credentials for your bot.  
   - Connect input from "Loop Over Items".

10. **Create a Wait node:**
    - Name: "Wait 1 min"  
    - Wait for 1 minute.  
    - Connect input from "Send a photo message".  
    - Connect output back to "Loop Over Items" to continue processing next item.

11. **Create an OpenRouter Chat Model node:**
    - Name: "OpenRouter Chat Model"  
    - Model: `x-ai/grok-4-fast:free`  
    - Credentials: OpenRouter API with valid key.  
    - Connect output to "AI Agent" as language model input.

12. **Create a LangChain Agent node:**
    - Name: "AI Agent"  
    - Prompt: `Simplify the {{ $json.contentSnippet }} in 2 short sentences.`  
    - Connect input from "Bleeping Computer Security Bulletin".  
    - Connect AI language model input from "OpenRouter Chat Model".  
    - Connect AI memory input from "Simple Memory" (see next).  
    - Connect output to "Merge".

13. **Create a LangChain Memory Buffer Window node:**
    - Name: "Simple Memory"  
    - Session key: `{{ $json.pubDate }}`  
    - Context window length: 13  
    - Connect output to AI Agent as memory input.

14. **Link nodes as per above connections to complete the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                   |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow runs daily at 9 AM to fetch and summarize cybersecurity news.                         | Scheduling detail                                  |
| Uses Grok AI via OpenRouter API for natural language summarization.                           | OpenRouter API, Grok AI model                      |
| Telegram bot posts messages at channel @DailySecurityNewss including images and summaries.    | Telegram API, PrivateMessageTesterBot_API          |
| Sponsored posts are filtered out by checking for "Sponsored" in the creator field.            | Content quality control                            |
| Wait node added to avoid Telegram message rate limits by spacing posts by 1 minute.           | Rate limiting and API best practices               |
| RSS feed source is Bleeping Computer's official cybersecurity feed.                           | https://www.bleepingcomputer.com/feed/             |
| Sticky notes in workflow provide contextual grouping for blocks: Scraper, Sponsored Removal, AI Summarization, Posting. | Visual grouping in n8n editor                       |

---

*Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, respecting all current content policies, and contains no illegal, offensive, or protected elements. All data handled is legal and public.*