Filter & Summarize AI News from TechCrunch to Slack using OpenAI

https://n8nworkflows.xyz/workflows/filter---summarize-ai-news-from-techcrunch-to-slack-using-openai-7346


# Filter & Summarize AI News from TechCrunch to Slack using OpenAI

---

### 1. Workflow Overview

This workflow, titled **"Firecrawl AI-Powered Market Intelligence Bot: Automated News Insights Delivery"**, is designed to automate the process of monitoring TechCrunch articles for AI-related news. It leverages Firecrawl’s web scraping API to extract structured data from TechCrunch, applies AI analysis to filter and summarize relevant articles, and finally delivers curated insights into a Slack channel for timely team awareness.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Scheduling**: Initiates the workflow daily at a fixed time.
- **1.2 Web Scraping & Data Collection**: Uses Firecrawl API to crawl TechCrunch for the latest articles.
- **1.3 Processing Wait & Fetch Results**: Waits briefly for crawl completion and retrieves results.
- **1.4 Data Splitting & Itemization**: Splits the batch of articles into individual items for processing.
- **1.5 AI-Powered Filtering & Summarization**: Uses OpenAI via LangChain to detect AI relevance and summarize articles.
- **1.6 Filtering Non-AI Content**: Removes articles classified as non-AI related.
- **1.7 Delivery to Slack**: Sends the filtered, summarized AI news to a specified Slack channel.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger & Scheduling

**Overview:**  
This block schedules the workflow to run automatically each day at 8 AM, ensuring fresh market intelligence is captured regularly without manual intervention.

**Nodes Involved:**  
- Daily Market Research Trigger

**Node Details:**

- **Daily Market Research Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow daily at 8:00 AM.  
  - *Configuration:* Interval trigger set to trigger at hour 8 daily.  
  - *Input:* None (time-based trigger)  
  - *Output:* Triggers the next node to start crawling TechCrunch.  
  - *Edge Cases:* Misconfiguration of timezone or trigger time could cause unexpected execution time.

---

#### 1.2 Web Scraping & Data Collection

**Overview:**  
This block calls the Firecrawl API to scrape TechCrunch for the latest articles, focusing on current-year content, limited to 20 articles, and extracting clean markdown-formatted main content.

**Nodes Involved:**  
- Crawl TechCrunch (FireCrawl)

**Node Details:**

- **Crawl TechCrunch (FireCrawl)**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Firecrawl API to initiate scraping of TechCrunch.  
  - *Configuration:*  
    - URL: `https://api.firecrawl.dev/v1/crawl`  
    - Method: POST  
    - Body: JSON specifying:  
      - target URL: `https://techcrunch.com`  
      - limit: 20 articles  
      - includePaths: `["2025/"]` (filters URLs containing year 2025)  
      - scrapeOptions: extracts markdown, main content only, parses PDFs, maxAge 4 hours to cache results  
    - Authentication: HTTP Bearer token (generic credentials)  
  - *Input:* Trigger from schedule node  
  - *Output:* Initiates crawl and returns initial crawl response (likely with a crawl ID).  
  - *Edge Cases:* API rate limits, invalid credentials, network errors, changes in Firecrawl API or TechCrunch site structure.

---

#### 1.3 Processing Wait & Fetch Results

**Overview:**  
After initiating the crawl, this block waits 60 seconds to allow the crawl to complete before requesting the crawl results.

**Nodes Involved:**  
- Wait  
- Receive Firecrawl Results

**Node Details:**

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses the workflow for 60 seconds.  
  - *Configuration:* Wait amount set to 60 seconds.  
  - *Input:* Output from crawl initiation node  
  - *Output:* Triggers the fetch results node after delay.  
  - *Edge Cases:* If crawl takes longer than 60 seconds, results may be incomplete or empty.

- **Receive Firecrawl Results**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the completed crawl results using the crawl ID from initial response.  
  - *Configuration:*  
    - URL dynamically set to `https://api.firecrawl.dev/v1/crawl/{{$json.id}}` (using crawl ID)  
    - Method: GET (default)  
    - Authentication: HTTP Bearer token (generic credentials)  
  - *Input:* Output from Wait node (crawl ID)  
  - *Output:* Returns JSON with array of scraped articles data.  
  - *Edge Cases:* Crawl failure, expired crawl ID, API errors, network issues.

---

#### 1.4 Data Splitting & Itemization

**Overview:**  
This block processes the bulk JSON response of articles and splits it into individual workflow items, each representing a single article with selected fields and truncated content for efficient downstream processing.

**Nodes Involved:**  
- Split Output

**Node Details:**

- **Split Output**  
  - *Type:* Code  
  - *Role:* Splits the array of articles into individual n8n items for parallel processing.  
  - *Configuration:* JavaScript code that:  
    - Checks if data array exists and is non-empty  
    - Maps each article to an item containing:  
      - title (from metadata)  
      - url (source URL)  
      - content (markdown or content truncated to 1000 characters)  
      - description (metadata)  
      - publishDate (metadata)  
  - *Input:* JSON data from Firecrawl results node  
  - *Output:* Array of individual article items  
  - *Edge Cases:* Empty data array, missing fields, very large content causing truncation.

---

#### 1.5 AI-Powered Filtering & Summarization

**Overview:**  
This block analyzes each article individually using an AI agent to determine if it relates to AI topics. If yes, it generates a 3-bullet point summary; if not, it flags it as "NOT_AI_RELATED".

**Nodes Involved:**  
- Summarizer Agent  
- OpenAI Summarizer

**Node Details:**

- **Summarizer Agent**  
  - *Type:* LangChain Agent (AI research assistant)  
  - *Role:* Determines AI relevance and generates summaries accordingly.  
  - *Configuration:*  
    - Prompt instructs the AI to:  
      - Check if article relates to artificial intelligence, ML, AI companies, or AI technology  
      - If AI-related, produce summary in 3 bullet points with article title and link  
      - If not AI-related, respond exactly with "NOT_AI_RELATED"  
    - Input variables used: `title`, `description`, `content`, `url` from article JSON  
  - *Input:* Individual article item from Split Output node  
  - *Output:* AI-generated summary or "NOT_AI_RELATED" string  
  - *Edge Cases:* AI model errors, timeouts, misclassification, incomplete content input.

- **OpenAI Summarizer**  
  - *Type:* LangChain LM Chat OpenAI  
  - *Role:* Provides the language model (GPT-4o-mini) backend to the Summarizer Agent.  
  - *Configuration:*  
    - Model: GPT-4o-mini (cost-effective GPT-4 variant)  
    - Credentials: OpenAI API key configured via n8n credentials  
  - *Input:* Receives prompt from Summarizer Agent  
  - *Output:* Returns AI-generated text to Summarizer Agent  
  - *Edge Cases:* API quota limits, authentication failures, rate limiting, latency.

---

#### 1.6 Filtering Non-AI Content

**Overview:**  
Filters out articles flagged as "NOT_AI_RELATED" by the AI agent to ensure only relevant AI news proceeds to delivery.

**Nodes Involved:**  
- Filter Messages

**Node Details:**

- **Filter Messages**  
  - *Type:* Code  
  - *Role:* Iterates over all items, retains only those with non-empty `output` field not equal to "NOT_AI_RELATED".  
  - *Configuration:* JavaScript code filters items accordingly.  
  - *Input:* Items from Summarizer Agent containing AI summary or non-AI flag  
  - *Output:* Filtered list of AI-related articles only  
  - *Edge Cases:* Mislabeling by AI, empty or missing output fields, no AI articles found (empty output).

---

#### 1.7 Delivery to Slack

**Overview:**  
Sends the curated AI news summaries as formatted messages to a designated Slack channel, facilitating team awareness and discussion.

**Nodes Involved:**  
- Send Summary to Slack

**Node Details:**

- **Send Summary to Slack**  
  - *Type:* Slack node  
  - *Role:* Posts messages to Slack channel `#general` containing the AI summary.  
  - *Configuration:*  
    - Text: "🔍 AI Research Summary:\n{{ $json.output }}" (injects the summary text)  
    - Channel: `#general` (by name)  
    - Authentication: OAuth2 credentials configured for Slack  
    - Webhook ID set for Slack integration  
  - *Input:* Filtered AI article summaries from Filter Messages node  
  - *Output:* Posts message(s) to Slack channel  
  - *Edge Cases:* Slack API errors, invalid OAuth tokens, channel not found, message formatting issues.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                         | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                                                                                                                                                               |
|---------------------------|----------------------------------|---------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Market Research Trigger | Schedule Trigger                 | Initiates daily workflow execution    | None                         | Crawl TechCrunch (FireCrawl) | # AI-Powered Market Intelligence Bot overview and scheduling details                                                                                                                                                                                                      |
| Crawl TechCrunch (FireCrawl) | HTTP Request                    | Initiates TechCrunch scraping         | Daily Market Research Trigger | Wait                        | Details on Firecrawl integration and scraping configuration                                                                                                                                                                                                                |
| Wait                      | Wait                             | Delays to allow crawl completion      | Crawl TechCrunch (FireCrawl) | Receive Firecrawl Results    | Explains wait purpose for respecting server resources                                                                                                                                                                                                                      |
| Receive Firecrawl Results | HTTP Request                    | Retrieves crawl results by crawl ID   | Wait                         | Split Output                | Part of data collection and scraping block                                                                                                                                                                                                                                |
| Split Output              | Code                             | Splits articles into individual items | Receive Firecrawl Results     | Summarizer Agent            | Describes splitting bulk JSON into separate articles for AI processing                                                                                                                                                                                                    |
| Summarizer Agent          | LangChain Agent                  | AI classification & summarization     | Split Output                 | Filter Messages             | Details on AI relevance detection and summarization with OpenAI                                                                                                                                                                                                            |
| OpenAI Summarizer         | LangChain LM Chat OpenAI         | Provides GPT-4o-mini AI model         | Summarizer Agent (ai_languageModel) | Summarizer Agent (ai_languageModel) | AI model configuration and credentials notes                                                                                                                                                                                                                              |
| Filter Messages           | Code                             | Filters out non-AI articles            | Summarizer Agent             | Send Summary to Slack       | Filtering logic to retain only AI-related summaries                                                                                                                                                                                                                        |
| Send Summary to Slack     | Slack                           | Posts AI summaries to Slack channel   | Filter Messages              | None                        | Slack integration details, message formatting, and OAuth2 authentication                                                                                                                                                                                                  |
| Sticky Note               | Sticky Note                      | Documentation and workflow overview   | None                         | None                        | Contains detailed overview, scheduling, scraping, AI analysis, filtering, and delivery blocks descriptions and best practices                                                                                                                                             |
| Sticky Note1              | Sticky Note                      | Configuration and best practices      | None                         | None                        | Covers configuration details, AI detection accuracy, Slack integration, and business impact                                                                                                                                                                               |
| Sticky Note2              | Sticky Note                      | AI analysis and delivery explanation  | None                         | None                        | Explains AI summarization, filtering, and Slack delivery mechanics                                                                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `Daily Market Research Trigger`  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 8:00 AM (TriggerAtHour = 8).

2. **Add HTTP Request Node to Start Crawl**  
   - Name: `Crawl TechCrunch (FireCrawl)`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/crawl`  
   - Body (JSON):  
     ```json
     {
       "url": "https://techcrunch.com",
       "limit": 20,
       "includePaths": ["2025/"],
       "scrapeOptions": {
         "formats": ["markdown"],
         "onlyMainContent": true,
         "parsePDF": true,
         "maxAge": 14400000
       }
     }
     ```  
   - Set `Send Body` to JSON  
   - Authentication: HTTP Bearer Token (configure with Firecrawl API token).

3. **Connect `Daily Market Research Trigger` → `Crawl TechCrunch (FireCrawl)`**

4. **Add Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Duration: 60 seconds

5. **Connect `Crawl TechCrunch (FireCrawl)` → `Wait`**

6. **Add HTTP Request Node to Retrieve Crawl Results**  
   - Name: `Receive Firecrawl Results`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://api.firecrawl.dev/v1/crawl/{{$json.id}}` (dynamic URL using previous node’s crawl ID)  
   - Authentication: HTTP Bearer Token (same as crawl node)

7. **Connect `Wait` → `Receive Firecrawl Results`**

8. **Add Code Node to Split Articles**  
   - Name: `Split Output`  
   - Type: Code  
   - JavaScript:  
     ```javascript
     if (!$json.data || $json.data.length === 0) {
       return [];
     }
     return $json.data.map(article => ({
       json: {
         title: article.metadata?.title || 'No title',
         url: article.sourceURL || '',
         content: (article.markdown || article.content || '').substring(0, 1000),
         description: article.metadata?.description || '',
         publishDate: article.metadata?.publishDate || ''
       }
     }));
     ```

9. **Connect `Receive Firecrawl Results` → `Split Output`**

10. **Add LangChain Agent Node for AI Analysis**  
    - Name: `Summarizer Agent`  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Prompt:  
      ```
      You are an AI research assistant. First, determine if this article is related to artificial intelligence, machine learning, AI companies, or AI technology.

      If the article IS AI-related, provide a summary in 3 bullet points.
      If the article is NOT AI-related, respond with exactly: "NOT_AI_RELATED"

      Article details:
      Title: {{ $json.title }}
      Description: {{ $json.description }}
      Content: {{ $json.content }}

      Format for AI articles:
      {{ $json.title }}

      Summary:
      - [Bullet point 1]
      - [Bullet point 2] 
      - [Bullet point 3]

      Link: {{ $json.url }}
      ```
    - Prompt Type: Define  
    - No additional options needed.

11. **Add LangChain LM Chat OpenAI Node**  
    - Name: `OpenAI Summarizer`  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: Select `gpt-4o-mini`  
    - Credentials: Set your OpenAI API credentials here.

12. **Connect `OpenAI Summarizer` to `Summarizer Agent` as AI language model**

13. **Connect `Split Output` → `Summarizer Agent`**

14. **Add Code Node to Filter Non-AI Articles**  
    - Name: `Filter Messages`  
    - Type: Code  
    - JavaScript:  
      ```javascript
      const filteredItems = [];
      $input.all().forEach(item => {
        if (item.json.output && item.json.output.trim() !== 'NOT_AI_RELATED') {
          filteredItems.push(item);
        }
      });
      return filteredItems;
      ```

15. **Connect `Summarizer Agent` → `Filter Messages`**

16. **Add Slack Node to Send Summaries**  
    - Name: `Send Summary to Slack`  
    - Type: Slack  
    - Authentication: OAuth2 configured for Slack workspace  
    - Channel: `#general` (select by name)  
    - Text:  
      ```
      🔍 AI Research Summary:
      {{ $json.output }}
      ```

17. **Connect `Filter Messages` → `Send Summary to Slack`**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow leverages Firecrawl, an AI-powered web scraping API designed to handle JavaScript rendering and anti-bot detection bypass, ensuring high-quality extraction of TechCrunch articles in markdown format.                                                                                                      | Firecrawl API docs: https://firecrawl.dev                                                                                                               |
| The AI summarization uses LangChain's agent node integrated with OpenAI GPT-4o-mini model for cost-effective yet powerful natural language processing.                                                                                                                                                                | OpenAI API: https://platform.openai.com/docs/models/gpt-4o-mini                                                                                          |
| Slack integration uses OAuth2 authentication and supports posting messages with rich formatting to facilitate team discussions on AI trends.                                                                                                                                                                         | Slack API: https://api.slack.com/authentication/oauth-v2                                                                                                |
| The scraping includes a date path filter `"includePaths": ["2025/"]` to limit to current-year articles. This must be updated annually for continued accuracy.                                                                                                                                                         | Maintenance reminder in sticky notes                                                                                                                     |
| The workflow includes multiple sticky notes with detailed explanations, best practices, and business impact insights, designed to help maintainers understand configuration, AI relevance criteria, and delivery mechanisms.                                                                                          | Sticky notes provide in-depth documentation and operational guidance within the workflow editor                                                         |
| Recommended best practices include monitoring AI relevance accuracy, reviewing summaries for quality, and tracking OpenAI API usage to manage costs effectively.                                                                                                                                                     | Sticky Note1 contains configuration and maintenance best practices                                                                                        |
| The workflow is designed for AI product teams, investors, and execs needing automated, daily AI market intelligence from TechCrunch without manual effort.                                                                                                                                                           | Business impact explained in Sticky Note1                                                                                                               |

---

**Disclaimer:**  
The text and workflow described are derived exclusively from an automated n8n workflow using publicly accessible APIs and legal data sources. All processing complies with content policies and contains no illegal or protected materials.

---