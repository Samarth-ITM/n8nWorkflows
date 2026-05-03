Tech News Aggregator: The Verge & TechCrunch RSS to Notion with GPT-4 Summaries

https://n8nworkflows.xyz/workflows/tech-news-aggregator--the-verge---techcrunch-rss-to-notion-with-gpt-4-summaries-8600


# Tech News Aggregator: The Verge & TechCrunch RSS to Notion with GPT-4 Summaries

### 1. Workflow Overview

This workflow aggregates technology news articles from two major RSS feeds — The Verge and TechCrunch — and processes them into summarized entries in a Notion database. It leverages GPT-4 to generate concise summaries of the full articles, ensuring a digestible overview for users. The workflow is designed to avoid duplicate entries by hashing article URLs and checking existing Notion entries. It also extracts full article content from the source web pages to enhance summary quality.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Triggers and RSS feed readers fetch articles from the two sources.
- **1.2 Deduplication and Looping:** Each article’s URL is hashed, and the hash is checked against the Notion database to skip duplicates.
- **1.3 Article Content Extraction:** For new articles, HTTP requests fetch the full HTML content, which is then parsed to extract the complete article text.
- **1.4 Data Cleaning:** Extracted article paragraphs are cleaned and concatenated into a plain text full article.
- **1.5 AI Summarization:** Cleaned full articles are summarized using the GPT-4 model with a strict prompt to ensure summaries fit Notion’s constraints.
- **1.6 Add to Notion:** Summaries, along with metadata and the full article text, are added as new pages in the Notion database.
- **1.7 Completion:** Ends the loop and marks the fetch as finished.

This process runs either on manual trigger (for testing) or scheduled triggers (disabled by default but configurable). It integrates OpenAI for summarization and Notion API for storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow, fetching RSS feeds from The Verge and TechCrunch.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Manual Trigger  
  - The Verge (RSS Feed Read)  
  - TechCrunch (RSS Feed Read)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automates workflow execution at scheduled intervals (disabled by default).  
    - Config: Hourly trigger (at 11 AM).  
    - Edge Cases: If enabled, improper scheduling could miss articles or cause rate limits.

  - **Manual Trigger**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing purposes.  
    - Edge Cases: None; used only for manual runs.

  - **The Verge (RSS Feed Read)**  
    - Type: RSS Feed Read  
    - Role: Fetches latest articles from The Verge RSS feed URL.  
    - Config: URL set to https://www.theverge.com/rss/index.xml.  
    - Output: List of articles with title, link, date, etc.  
    - Edge Cases: RSS feed unavailability, malformed XML.

  - **TechCrunch (RSS Feed Read)**  
    - Type: RSS Feed Read  
    - Role: Fetches latest articles from TechCrunch RSS feed URL.  
    - Config: URL set to https://techcrunch.com/feed/.  
    - Output: List of articles similar to The Verge node.  
    - Edge Cases: Same as The Verge feed.

---

#### 1.2 Deduplication and Looping

- **Overview:** Iterates over each feed’s articles, creates a SHA256 hash of the article URL, and checks if this article already exists in Notion to avoid duplicates.
- **Nodes Involved:**  
  - Crypto (SHA256 hash) for The Verge  
  - Crypto1 (SHA256 hash) for TechCrunch  
  - Loop Over Items (splitInBatches) for The Verge  
  - Loop Over Items1 (splitInBatches) for TechCrunch  
  - Get many database pages (Notion query) for The Verge  
  - Get many database pages1 (Notion query) for TechCrunch  
  - If (checks if article exists) for The Verge  
  - If1 (checks if article exists) for TechCrunch

- **Node Details:**

  - **Crypto / Crypto1**  
    - Type: Crypto  
    - Role: Generates SHA256 hash from article URL to uniquely identify articles.  
    - Config: Hash value derived from `{{$json.link}}`.  
    - Edge Cases: Hash collision unlikely; failure if URL missing.

  - **Loop Over Items / Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes articles individually in batches (default batch size).  
    - Edge Cases: Large batch size could cause timeouts; empty input list.

  - **Get many database pages / Get many database pages1**  
    - Type: Notion API (Database Page - getAll)  
    - Role: Queries Notion database for existing entries matching the article hash.  
    - Config: Filter on property “Hash” equals current article hash.  
    - Credentials: Notion API with appropriate access.  
    - Edge Cases: API rate limits, connectivity issues, empty or multiple matches.

  - **If / If1**  
    - Type: If  
    - Role: Checks if the Notion query returned any matching page (exists).  
    - Condition: If “id” field exists in the query result, means article exists.  
    - Output: If true — skip to next article; if false — proceed to fetch full article.  
    - Edge Cases: Expression failures if JSON path missing.

---

#### 1.3 Article Content Extraction

- **Overview:** For new articles, fetches the full HTML content from the article link and extracts the article body paragraphs.
- **Nodes Involved:**  
  - HTTP Request (The Verge)  
  - HTML (The Verge)  
  - HTTP Request1 (TechCrunch)  
  - HTML1 (TechCrunch)

- **Node Details:**

  - **HTTP Request / HTTP Request1**  
    - Type: HTTP Request  
    - Role: Loads the full article webpage from the article link.  
    - Config: URL set dynamically from current article’s link.  
    - Edge Cases: Network timeouts, 404 errors, redirects, anti-bot measures.

  - **HTML / HTML1**  
    - Type: HTML Extract  
    - Role: Parses the HTML response to extract paragraphs of the article body.  
    - Config:  
      - For The Verge: CSS selector `.duet--article--article-body-component p` (paragraphs inside article body).  
      - For TechCrunch: CSS selector `.entry-content p`.  
    - Output: Array of paragraph text strings.  
    - Edge Cases: Site layout changes breaking CSS selectors, empty extraction.

---

#### 1.4 Data Cleaning

- **Overview:** Cleans extracted paragraphs by removing empty elements, link references, and extra spaces, then joins them into a single full article text block.
- **Nodes Involved:**  
  - Code in JavaScript (The Verge)  
  - Code in JavaScript1 (TechCrunch)

- **Node Details:**

  - **Code in JavaScript / Code in JavaScript1**  
    - Type: Code (JavaScript)  
    - Role: Cleans and formats extracted paragraphs.  
    - Logic:  
      - Filters out empty paragraphs.  
      - Removes bracketed link references `[like this]`.  
      - Normalizes spaces and trims text.  
      - Joins paragraphs with double newline separators.  
    - Output: JSON with property `fullArticle` containing plain text article.  
    - Edge Cases: Unexpected HTML content causing failed replacements; empty article.

---

#### 1.5 AI Summarization

- **Overview:** Uses GPT-4 via LangChain integration to generate a concise, plain-text summary of the full article within Notion’s character limits.
- **Nodes Involved:**  
  - OpenAI Chat Model (The Verge)  
  - Basic LLM Chain (The Verge)  
  - OpenAI Chat Model1 (TechCrunch)  
  - Basic LLM Chain1 (TechCrunch)

- **Node Details:**

  - **OpenAI Chat Model / OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the language model backend (GPT-4.1-mini).  
    - Credentials: OpenAI API key with free credits configured.  
    - Edge Cases: API limits, network issues, model errors.

  - **Basic LLM Chain / Basic LLM Chain1**  
    - Type: LangChain LLM Chain  
    - Role: Sends the cleaned full article text to GPT-4 with a strict prompt to produce summary.  
    - Prompt Highlights:  
      - Max 1500 characters, plain text only, no links or markdown.  
      - Focus on main arguments, findings, or announcements.  
      - Neutral, professional tone.  
      - Highlights important updates only for product/news articles.  
    - Input: `{{$json.fullArticle}}`.  
    - Output: Summary text.  
    - Edge Cases: Summarization failures, exceeding token limits.

---

#### 1.6 Add to Notion Database

- **Overview:** Creates a new page in the Notion database with article metadata, summary, full article text, and status.
- **Nodes Involved:**  
  - Create a database page (The Verge)  
  - Create a database page1 (TechCrunch)

- **Node Details:**

  - **Create a database page / Create a database page1**  
    - Type: Notion API (Database Page - create)  
    - Role: Inserts new article page into Notion “Tech & Startups rss feed” database.  
    - Configured properties:  
      - Title: article title from feed.  
      - Summary: generated summary text.  
      - Date: article published date.  
      - Hash: SHA256 hash of article URL.  
      - URL: article link.  
      - Digest Status: set to “Pending”.  
      - Source: fixed string (“theVerge” or “Tech Crunch”).  
      - Full Article: full cleaned article text.  
    - Credentials: Connected to Notion account with appropriate permissions.  
    - Edge Cases: API rate limit, invalid property mappings, permission errors.

---

#### 1.7 Completion

- **Overview:** Marks the end of the processing loop for each article batch.
- **Nodes Involved:**  
  - fetch finished (NoOp)

- **Node Details:**

  - **fetch finished**  
    - Type: NoOp (no operation)  
    - Role: Serves as a placeholder node to mark the end of the workflow branch.  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name              | Node Type                      | Functional Role                           | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                   |
|------------------------|--------------------------------|------------------------------------------|-----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger               | Automate workflow runs at scheduled time | —                           | The Verge, TechCrunch        | ## - manual trigger is for testing<br>## - schedule trigger should be set to appropriate intervals so that we do not miss rolling articles |
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual execution for testing               | —                           | The Verge, TechCrunch        | ## - manual trigger is for testing<br>## - schedule trigger should be set to appropriate intervals so that we do not miss rolling articles |
| The Verge              | RSS Feed Read                  | Fetch RSS feed articles from The Verge    | Schedule Trigger, Manual Trigger | Crypto                      |                                                                                                               |
| TechCrunch             | RSS Feed Read                  | Fetch RSS feed articles from TechCrunch   | Schedule Trigger, Manual Trigger | Crypto1                     |                                                                                                               |
| Crypto                 | Crypto (SHA256)                | Create hash from article URL (The Verge)  | The Verge                   | Loop Over Items              | ## create hash from url of article so that we can eliminate repeating ones                                    |
| Crypto1                | Crypto (SHA256)                | Create hash from article URL (TechCrunch) | TechCrunch                  | Loop Over Items1             | ## create hash from url of article so that we can eliminate repeating ones                                    |
| Loop Over Items        | SplitInBatches                 | Loop over The Verge articles               | Crypto                      | fetch finished, Get many database pages | ## loop over articles                                                                                          |
| Loop Over Items1       | SplitInBatches                 | Loop over TechCrunch articles              | Crypto1                     | fetch finished, Get many database pages1 | ## loop over articles                                                                                          |
| Get many database pages| Notion (Get All Pages)         | Check if article exists in Notion (The Verge) | Loop Over Items             | If                          | ## check if article already exist                                                                              |
| Get many database pages1| Notion (Get All Pages)         | Check if article exists in Notion (TechCrunch) | Loop Over Items1            | If1                         | ## check if article already exist                                                                              |
| If                     | If                            | If article exists in Notion (The Verge)     | Get many database pages      | Loop Over Items / HTTP Request | ## if exist then skip it and back to loop                                                                      |
| If1                    | If                            | If article exists in Notion (TechCrunch)    | Get many database pages1     | Loop Over Items1 / HTTP Request1 | ## if exist then skip it and back to loop                                                                      |
| HTTP Request           | HTTP Request                   | Fetch full article HTML (The Verge)          | If (false path)              | HTML                        | ## if does not exist then load full article                                                                     |
| HTTP Request1          | HTTP Request                   | Fetch full article HTML (TechCrunch)         | If1 (false path)             | HTML1                       | ## if does not exist then load full article                                                                     |
| HTML                   | HTML Extract                   | Extract article paragraphs (The Verge)       | HTTP Request                 | Code in JavaScript           | ## extract complete article                                                                                      |
| HTML1                  | HTML Extract                   | Extract article paragraphs (TechCrunch)      | HTTP Request1                | Code in JavaScript1          | ## extract complete article                                                                                      |
| Code in JavaScript     | Code                          | Clean and join paragraphs into full article (The Verge) | HTML                        | Basic LLM Chain              | ## clean data                                                                                                    |
| Code in JavaScript1    | Code                          | Clean and join paragraphs into full article (TechCrunch) | HTML1                       | Basic LLM Chain1             | ## clean data                                                                                                    |
| OpenAI Chat Model      | LangChain OpenAI Chat Model   | GPT-4 model backend (The Verge)               | Basic LLM Chain (ai_languageModel input) | Basic LLM Chain (ai_languageModel output) |                                                                                                               |
| OpenAI Chat Model1     | LangChain OpenAI Chat Model   | GPT-4 model backend (TechCrunch)              | Basic LLM Chain1 (ai_languageModel input) | Basic LLM Chain1 (ai_languageModel output) |                                                                                                               |
| Basic LLM Chain        | LangChain LLM Chain           | Generate article summary (The Verge)           | Code in JavaScript, OpenAI Chat Model | Create a database page        | ## make article count less than 2000                                                                             |
| Basic LLM Chain1       | LangChain LLM Chain           | Generate article summary (TechCrunch)           | Code in JavaScript1, OpenAI Chat Model1 | Create a database page1       | ## make article count less than 2000                                                                             |
| Create a database page | Notion (Create Page)           | Add summarized article to Notion (The Verge)   | Basic LLM Chain              | Loop Over Items              | ## add to notion db page                                                                                         |
| Create a database page1| Notion (Create Page)           | Add summarized article to Notion (TechCrunch)  | Basic LLM Chain1             | Loop Over Items1             | ## add to notion db page                                                                                         |
| fetch finished         | NoOp                          | Marks end of article processing loop           | Loop Over Items / Loop Over Items1 | —                            |                                                                                                               |
| Sticky Notes           | Sticky Note                   | Visual comments and annotations                | —                           | —                            | Various contextual notes covering the workflow stages (see details in sections)                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**  
   - Add a **Manual Trigger** node for testing.  
   - Add a **Schedule Trigger** node (disabled by default) configured to run daily at 11:00 AM (or desired interval).

2. **Add RSS Feed Nodes:**  
   - Create two **RSS Feed Read** nodes named “The Verge” and “TechCrunch”.  
   - Set their URLs to:  
     - The Verge: `https://www.theverge.com/rss/index.xml`  
     - TechCrunch: `https://techcrunch.com/feed/`

3. **Create Hashes:**  
   - Add two **Crypto** nodes (SHA256) named “Crypto” and “Crypto1”.  
   - Set input value to `{{$json.link}}` for each article from the respective feed.

4. **Set Up Loops:**  
   - Add two **SplitInBatches** nodes named “Loop Over Items” and “Loop Over Items1” for The Verge and TechCrunch respectively.  
   - Connect “Crypto” to “Loop Over Items” and “Crypto1” to “Loop Over Items1”.

5. **Check for Existing Articles in Notion:**  
   - Add two **Notion - Get Many Database Pages** nodes named “Get many database pages” and “Get many database pages1”.  
   - Configure to query the Notion database “Tech & Startups rss feed” by filtering the “Hash” property equal to current article hash (`={{ $json.hash }}`).  
   - Connect “Loop Over Items” to “Get many database pages” and “Loop Over Items1” to “Get many database pages1”.

6. **Conditional Check if Article Exists:**  
   - Add two **If** nodes named “If” and “If1”.  
   - Condition: Check if the first item from Notion query result has an “id” property (exists).  
   - True output: Connect back to the respective “Loop Over Items” node to skip processing.  
   - False output: Proceed to fetch full article.

7. **Fetch Full Article HTML:**  
   - Add two **HTTP Request** nodes named “HTTP Request” and “HTTP Request1”.  
   - Set URL dynamically from article link: `={{ $('Crypto').item.json.link }}` or `={{ $('Crypto1').item.json.link }}`.  
   - Connect the false output of “If”/“If1” to these nodes.

8. **Extract Article Text:**  
   - Add two **HTML Extract** nodes named “HTML” and “HTML1”.  
   - Configure CSS selectors:  
     - The Verge: `.duet--article--article-body-component p`  
     - TechCrunch: `.entry-content p`  
   - Connect “HTTP Request” to “HTML” and “HTTP Request1” to “HTML1”.

9. **Clean Extracted Paragraphs:**  
   - Add two **Code** nodes named “Code in JavaScript” and “Code in JavaScript1”.  
   - Use JavaScript to:  
     - Filter out empty paragraphs  
     - Remove bracketed references (`[links]`)  
     - Normalize whitespace  
     - Join paragraphs with double newlines into `fullArticle`.  
   - Connect “HTML” to “Code in JavaScript” and “HTML1” to “Code in JavaScript1”.

10. **Configure GPT-4 Summarization:**  
    - Add two **LangChain OpenAI Chat Model** nodes named “OpenAI Chat Model” and “OpenAI Chat Model1”.  
    - Set model to `gpt-4.1-mini`.  
    - Provide OpenAI API credentials with appropriate key and quota.

11. **Set Up Summarization Chain:**  
    - Add two **Basic LLM Chain** nodes named “Basic LLM Chain” and “Basic LLM Chain1”.  
    - Configure prompt to summarize `{{$json.fullArticle}}` with instructions:  
      - Max 1500 characters, plain text only, no links or markdown.  
      - Focus on main points and updates, neutral tone.  
    - Connect “Code in JavaScript” to “Basic LLM Chain” and “Code in JavaScript1” to “Basic LLM Chain1”.  
    - Connect “OpenAI Chat Model” as AI language model input for “Basic LLM Chain” and similarly for TechCrunch.

12. **Add Summaries to Notion:**  
    - Add two **Notion - Create Database Page** nodes named “Create a database page” and “Create a database page1”.  
    - Connect “Basic LLM Chain” and “Basic LLM Chain1” to these nodes respectively.  
    - Configure properties:  
      - Title: article title  
      - Summary: summary from LLM chain output  
      - Date: published date  
      - Hash: article hash  
      - URL: article link  
      - Digest Status: set as “Pending”  
      - Source: “theVerge” or “Tech Crunch” accordingly  
      - Full Article: cleaned full article text  
    - Set Notion credentials with access to the target database.

13. **Loop Completion:**  
    - Connect the create page nodes back to their respective “Loop Over Items” nodes to continue processing.

14. **Add NoOp Node:**  
    - Add a **NoOp** node named “fetch finished” to mark the end of each loop branch.

15. **Connect Initial Triggers:**  
    - Connect “When clicking ‘Execute workflow’” and “Schedule Trigger” nodes to both “The Verge” and “TechCrunch” RSS feed nodes to start the process.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses SHA256 hashes of article URLs to efficiently deduplicate articles in Notion.  | Sticky Note4: "used sha258 - used hash to identify unique article because querying hash is more efficient" |
| Summaries are strictly limited to 1500 characters to comply with Notion’s 2000 character limit. | Sticky Note9: "make article count less than 2000"                                                    |
| Full article extraction uses CSS selectors specific to each website’s HTML structure.           | Sticky Note7 and Sticky Note8: "if does not exist then load full article" and "extract complete article" |
| The manual trigger is intended for testing; scheduled trigger frequency must be carefully set.  | Sticky Note: "manual trigger is for testing; schedule trigger should be set to appropriate intervals"  |
| The Notion database used is named “Tech & Startups rss feed” with properties for summary, date, URL, hash, source, etc. | Notion node configuration details                                                                 |
| OpenAI API credentials require appropriate quota for GPT-4 usage via LangChain nodes.           | Credentials configured in OpenAI Chat Model and Basic LLM Chain nodes                                |

---

This comprehensive reference should enable users and AI agents to fully understand, reproduce, and maintain the "Tech News Aggregator: The Verge & TechCrunch RSS to Notion with GPT-4 Summaries" workflow with clarity and confidence.