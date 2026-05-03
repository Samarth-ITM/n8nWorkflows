Firecrawl AI-Powered Market Intelligence Bot: Automated News Insights Delivery

https://n8nworkflows.xyz/workflows/firecrawl-ai-powered-market-intelligence-bot--automated-news-insights-delivery-4588


# Firecrawl AI-Powered Market Intelligence Bot: Automated News Insights Delivery

---

# Firecrawl AI-Powered Market Intelligence Bot: Automated News Insights Delivery

---

### 1. Workflow Overview

This workflow automates the daily process of gathering, filtering, and summarizing market-relevant news articles from TechCrunch using FireCrawl’s API, then delivers concise AI-generated summaries to a Slack channel. It targets market researchers, analysts, and business teams seeking timely insights on technology trends, startups, and AI developments without manual effort.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Scheduling:** Initiates the workflow daily at a set time.
- **1.2 Crawling & Filtering:** Crawls TechCrunch for new articles and filters them for domain relevance.
- **1.3 Summarization & Delivery:** Uses AI to summarize filtered articles and posts the summaries to Slack.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger & Scheduling

**Overview:**  
Automatically triggers the workflow every day at 8:00 AM to ensure fresh market intelligence is gathered daily.

**Nodes Involved:**  
- Daily Market Research Trigger

**Node Details:**

- **Node Name:** Daily Market Research Trigger  
- **Type:** Schedule Trigger  
- **Technical Role:** Starting point for the workflow based on time scheduling.  
- **Configuration:**  
  - Interval: Daily trigger at 08:00 (8 AM) local time.  
- **Expressions/Variables:** None.  
- **Input Connections:** None (entry point).  
- **Output Connections:** Connects to "Crawl TechCrunch (FireCrawl)" node.  
- **Version-specific Requirements:** Version 1.2 of Schedule Trigger node used.  
- **Potential Failures:**  
  - Misconfigured trigger time could cause missed executions.  
  - Workflow disabled state would prevent triggering.  
- **Sub-workflow:** None.

---

#### 1.2 Crawling & Filtering

**Overview:**  
This block performs web crawling on TechCrunch via FireCrawl API to retrieve articles, then filters those articles to retain only those relevant to AI, machine learning, startups, and generative technologies.

**Nodes Involved:**  
- Crawl TechCrunch (FireCrawl)  
- Filter Relevant Articles

**Node Details:**

- **Node Name:** Crawl TechCrunch (FireCrawl)  
- **Type:** HTTP Request  
- **Technical Role:** Sends a POST request to FireCrawl API to scrape and extract articles from TechCrunch.  
- **Configuration:**  
  - URL: `https://api.firecrawl.dev/v1/crawl`  
  - Method: POST  
  - Body Parameters:  
    - url = `https://techcrunch.com`  
    - crawl_type = `"scrape"`  
    - extract_article = `true`  
  - Headers: Authorization with Bearer token (requires valid FireCrawl API key).  
- **Expressions:** None dynamic; static URL and parameters.  
- **Input Connections:** From "Daily Market Research Trigger".  
- **Output Connections:** To "Filter Relevant Articles".  
- **Version-specific Requirements:** HTTP Request version 4.2.  
- **Potential Failures:**  
  - Authentication failure due to invalid/missing API key.  
  - Network timeouts or API rate limits.  
  - Unexpected API response format changes.  
- **Sub-workflow:** None.

- **Node Name:** Filter Relevant Articles  
- **Type:** Code (JavaScript)  
- **Technical Role:** Filters the retrieved articles based on keyword relevance to domain-specific topics.  
- **Configuration:**  
  - Code checks article titles and content for presence of keywords: `AI`, `machine learning`, `startup`, `generative`.  
  - Converts text to lowercase for case-insensitive matching.  
  - Returns only articles matching at least one keyword.  
- **Expressions:** Uses `items` array from previous node output; accesses `item.json.article.title` and `.content`.  
- **Input Connections:** From "Crawl TechCrunch (FireCrawl)".  
- **Output Connections:** To "Summarizer Agent".  
- **Version-specific Requirements:** Code node version 2.  
- **Potential Failures:**  
  - Missing article fields (title or content) causing undefined errors (mitigated by optional chaining).  
  - Empty input array results in no filtered articles.  
  - Logic error if keywords list is modified incorrectly.  
- **Sub-workflow:** None.

---

#### 1.3 Summarization & Delivery

**Overview:**  
This block uses AI to generate concise summaries of each filtered article and sends these summaries as formatted messages to a designated Slack channel for team consumption.

**Nodes Involved:**  
- Summarizer Agent  
- OpenAI Summarizer  
- Send Summary to Slack

**Node Details:**

- **Node Name:** Summarizer Agent  
- **Type:** LangChain AI Agent  
- **Technical Role:** Coordinates AI summarization by passing article content and meta to OpenAI model.  
- **Configuration:**  
  - Prompt Template:  
    ```
    Summarize the following article in 3 bullet points:
    Title: {{ $json.article.title }}
    description: {{ $json.meta.description }}
    Content: {{ $json.article.content }}
    ```  
  - Prompt type set to "define".  
- **Expressions:** Uses Mustache syntax to insert article fields dynamically.  
- **Input Connections:** From "Filter Relevant Articles".  
- **Output Connections:** To "Send Summary to Slack".  
- **Version-specific Requirements:** LangChain Agent version 1.9.  
- **Potential Failures:**  
  - Missing article fields leading to incomplete prompts.  
  - AI model rate limits or errors.  
- **Sub-workflow:** None.

- **Node Name:** OpenAI Summarizer  
- **Type:** LangChain LM Chat OpenAI Node  
- **Technical Role:** Executes the actual summarization using OpenAI GPT-4o-mini model.  
- **Configuration:**  
  - Model: `gpt-4o-mini` (optimized GPT-4 variant).  
  - No additional options configured.  
  - Credentials: OpenAI API key configured under "OpenAi account 2".  
- **Input Connections:** Feeds AI language model to "Summarizer Agent" node (ai_languageModel input).  
- **Output Connections:** Connected back as AI model for agent.  
- **Version-specific Requirements:** Version 1.2 of LangChain LM Chat OpenAI node.  
- **Potential Failures:**  
  - API key expiration or revocation.  
  - Network or service downtime.  
  - Rate limiting by OpenAI.  
- **Sub-workflow:** None.

- **Node Name:** Send Summary to Slack  
- **Type:** Slack Node  
- **Technical Role:** Sends the summarized article content as a message to a designated Slack channel.  
- **Configuration:**  
  - Text format uses template:  
    ```
    🔍 AI Research Summary: 
    Title: {{ $('Filter Relevant Articles').item.json.article.title }}
    Link: {{ $('Filter Relevant Articles').item.json.url }}
    Summary: {{ $json.output }}
    ```  
  - Channel: Slack channel ID `C08TTV0CC3E` (market research or similar channel).  
  - Webhook ID configured for Slack integration.  
  - Credentials: Slack API token configured under "Slack account 2".  
- **Expressions:** Dynamic data extracted from previous nodes to populate message.  
- **Input Connections:** From "Summarizer Agent".  
- **Output Connections:** None (end node).  
- **Version-specific Requirements:** Slack node version 2.3.  
- **Potential Failures:**  
  - Slack API token invalid or expired.  
  - Channel ID misconfigured or lacking permissions.  
  - Slack rate limiting or connectivity issues.  
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                  | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                   |
|----------------------------|----------------------------------|--------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Daily Market Research Trigger | Schedule Trigger                 | Initiates workflow daily        | None                         | Crawl TechCrunch (FireCrawl) |                                                                                               |
| Crawl TechCrunch (FireCrawl) | HTTP Request                    | Crawls TechCrunch articles via FireCrawl API | Daily Market Research Trigger | Filter Relevant Articles      | 🌐💻 2. Crawling & Filtering: Uses FireCrawl API to scrape TechCrunch; inputs URL and parameters; retrieves article data. |
| Filter Relevant Articles    | Code                             | Filters articles by keywords    | Crawl TechCrunch (FireCrawl) | Summarizer Agent             | 🧠 Filters articles for relevance based on keywords like "AI", "machine learning", "startup".  |
| Summarizer Agent            | LangChain AI Agent               | Creates AI summarization prompt | Filter Relevant Articles      | Send Summary to Slack         | 🤖💬 3. Summarization & Delivery: Prepares article for AI summarization with GPT prompt.        |
| OpenAI Summarizer           | LangChain LM Chat OpenAI         | Performs AI language model chat | Connected as ai_languageModel input to Summarizer Agent | Summarizer Agent (ai_languageModel) | Uses GPT-4o-mini model to generate summary with OpenAI credentials.                           |
| Send Summary to Slack       | Slack Node                      | Posts summary to Slack channel  | Summarizer Agent             | None                        | 💬 Sends AI summary to designated Slack channel (e.g., #market-research).                      |
| Sticky Note                 | Sticky Note                     | Documentation and explanation   | None                         | None                        | 🌐💻 2. Crawling & Filtering description and code snippet.                                    |
| Sticky Note1                | Sticky Note                     | Documentation and explanation   | None                         | None                        | 🤖💬 3. Summarization & Delivery explanation with prompt and Slack message example.            |
| Sticky Note9                | Sticky Note                     | Workflow assistance contact info| None                         | None                        | Support contact and tutorial links.                                                         |
| Sticky Note4                | Sticky Note                     | Workflow overview and detailed explanation | None                         | None                        | Full detailed explanation of workflow sections with emojis and code snippet.                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "Daily Market Research Trigger"**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 08:00 (8 AM) local time.  
   - No credentials needed.  
   - Connect its main output to the next node.

2. **Create HTTP Request Node: "Crawl TechCrunch (FireCrawl)"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/crawl`  
   - Body Parameters (JSON):  
     - `url`: `https://techcrunch.com`  
     - `crawl_type`: `"scrape"`  
     - `extract_article`: `true`  
   - Headers: Add `Authorization` header with value `Bearer YOUR_FIRECRAWL_API_KEY` (replace with valid API key).  
   - Connect input from "Daily Market Research Trigger".  
   - No credentials required beyond API key in header.

3. **Create Code Node: "Filter Relevant Articles"**  
   - Type: Code (JavaScript)  
   - Paste the following code to filter articles by keywords:  
     ```javascript
     const keywords = ['AI', 'machine learning', 'startup', 'generative'];
     const results = [];

     for (const item of items) {
       const title = item.json.article?.title?.toLowerCase() || '';
       const content = item.json.article?.content?.toLowerCase() || '';

       const isRelevant = keywords.some(keyword =>
         title.includes(keyword.toLowerCase()) ||
         content.includes(keyword.toLowerCase())
       );

       if (isRelevant) {
         results.push(item);
       }
     }
     return results;
     ```  
   - Connect input from "Crawl TechCrunch (FireCrawl)".  
   - Output connects to the summarization agent node.

4. **Create LangChain AI Agent Node: "Summarizer Agent"**  
   - Type: LangChain Agent  
   - Prompt Type: Define  
   - Prompt Text:  
     ```
     Summarize the following article in 3 bullet points:
     Title: {{ $json.article.title }}
     description: {{ $json.meta.description }}
     Content: {{ $json.article.content }}
     ```  
   - Connect input from "Filter Relevant Articles".  
   - Connect ai_languageModel input to the OpenAI Summarizer node (next step).

5. **Create LangChain LM Chat OpenAI Node: "OpenAI Summarizer"**  
   - Type: LangChain LM Chat OpenAI  
   - Model: Select `gpt-4o-mini` from the list.  
   - Credentials: Set OpenAI API credentials (create or select existing OpenAI API key credential).  
   - Connect output to the ai_languageModel input of "Summarizer Agent".

6. **Connect "Summarizer Agent" output to Slack Node.**

7. **Create Slack Node: "Send Summary to Slack"**  
   - Type: Slack  
   - Credentials: Configure with a valid Slack API token with permission to post messages.  
   - Channel: Set channel ID to target Slack channel (e.g., `C08TTV0CC3E`).  
   - Message Text Template:  
     ```
     🔍 AI Research Summary: 
     Title: {{ $('Filter Relevant Articles').item.json.article.title }}
     Link: {{ $('Filter Relevant Articles').item.json.url }}
     Summary: {{ $json.output }}
     ```  
   - Connect input from "Summarizer Agent".  
   - No output connections needed (final node).

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow assistance and support: Contact Yaron at Yaron@nofluff.online                                                  | Contact email                                                                                     |
| Explore workflow tips and tutorials by Yaron Been on YouTube and LinkedIn                                               | YouTube: https://www.youtube.com/@YaronBeen/videos <br> LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| FireCrawl API documentation and usage requires valid API key for authentication                                        | FireCrawl API: https://api.firecrawl.dev                                                         |
| OpenAI API usage requires API key with access to GPT-4o-mini or similar models                                          | OpenAI API: https://platform.openai.com/docs/api-reference                                       |
| Slack API token must have chat:write scope to post messages to channels                                                | Slack API docs: https://api.slack.com/scopes                                                     |

---

**Disclaimer:** The text above is exclusively derived from an automated n8n workflow. It adheres strictly to content policies and contains no illegal or protected content. All manipulated data is legal and publicly available.

---