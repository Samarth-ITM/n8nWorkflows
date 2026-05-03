Curate Learning Content from Reddit & RSS with GPT-4.1-mini and Google Sheets

https://n8nworkflows.xyz/workflows/curate-learning-content-from-reddit---rss-with-gpt-4-1-mini-and-google-sheets-10007


# Curate Learning Content from Reddit & RSS with GPT-4.1-mini and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Personalized Learning Content Aggregator with AI Filtering,"** automates the collection, filtering, and storage of educational content from multiple online sources. It targets learners, educators, and professionals seeking curated learning materials tailored around specific keywords. The workflow runs twice daily and integrates RSS feeds, Reddit searches, Google Sheets, and an AI filtering layer to ensure relevant, high-quality educational content is saved for review.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Configuration:** Initiates the workflow on a schedule and loads configuration data (RSS feeds and keywords).
- **1.2 Keyword Processing:** Retrieves keywords from Google Sheets and prepares them for iteration.
- **1.3 Content Retrieval:** Searches RSS feeds and Reddit for articles/posts matching the keywords.
- **1.4 Content Preparation:** Normalizes and prepares the aggregated content into a uniform format for AI analysis.
- **1.5 AI Filtering:** Uses a GPT-4.1-mini powered AI agent to filter out irrelevant or low-quality content based on a defined prompt.
- **1.6 Output Processing & Storage:** Parses AI output and saves curated articles back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Configuration

**Overview:**  
This block triggers the workflow twice daily and loads user-defined RSS feed URLs and keywords configuration.

**Nodes Involved:**  
- Schedule Trigger - Twice Daily  
- Workflow Configuration  
- Get Keywords from Google Sheets  

**Node Details:**

- **Schedule Trigger - Twice Daily**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at 8:00 AM and 6:00 PM daily  
  - Configuration: Two trigger times set by hour only; no minutes or seconds specified  
  - Inputs: None (trigger node)  
  - Outputs: Connected to Workflow Configuration  
  - Edge Cases: If n8n instance is down at trigger time, workflow may miss execution  
  - Version: 1.2  

- **Workflow Configuration**  
  - Type: Set Node  
  - Role: Holds key configuration data, primarily a placeholder for RSS feed URLs as a comma-separated string  
  - Configuration: Assigns a string parameter `rssFeeds` with placeholder text to be replaced by actual RSS URLs  
  - Inputs: From Schedule Trigger  
  - Outputs: To Get Keywords from Google Sheets  
  - Edge Cases: Placeholder must be replaced with valid RSS URLs for correct operation  
  - Version: 3.4  

- **Get Keywords from Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Fetches user-defined keywords from a Google Sheet for content searching  
  - Configuration: Reads from sheet with `gid=0` in a specific Google Sheet document by ID  
  - Inputs: From Workflow Configuration  
  - Outputs: To Loop Over Keywords  
  - Credentials: Requires Google Sheets OAuth2 credentials properly configured in n8n  
  - Edge Cases: Connectivity issues or permission errors with Google Sheets; empty or malformed keyword list  
  - Version: 4.7  

---

#### 2.2 Keyword Processing

**Overview:**  
Processes the retrieved keywords and prepares them for iterative content searching.

**Nodes Involved:**  
- Loop Over Keywords  

**Node Details:**

- **Loop Over Keywords**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts and normalizes keywords from Google Sheets data to ensure each is a clean, non-empty string  
  - Configuration: Custom JavaScript code iterates over all input rows, extracts a keyword field (case-insensitive), trims whitespace, and outputs a list of keyword objects  
  - Inputs: From Get Keywords from Google Sheets  
  - Outputs: Two parallel outputs feeding RSS and Reddit search nodes  
  - Edge Cases: Handles cases where keyword fields might be objects or missing; skips empty or invalid keywords  
  - Version: 2  

---

#### 2.3 Content Retrieval

**Overview:**  
Performs searches for each keyword across multiple content sources: RSS feeds and Reddit API.

**Nodes Involved:**  
- Search RSS Feeds  
- Search Reddit via API  
- Merge All Search Results  

**Node Details:**

- **Search RSS Feeds**  
  - Type: RSS Feed Read  
  - Role: Fetches latest articles from the first RSS feed URL specified in the Workflow Configuration node  
  - Configuration: URL dynamically set to first RSS feed URL from the comma-separated list in the Workflow Configuration node  
  - Inputs: From Loop Over Keywords (first output)  
  - Outputs: To Merge All Search Results (first input)  
  - Edge Cases: If RSS URL is invalid/unreachable; only processes first RSS URL (additional feeds require extension)  
  - Version: 1.2  

- **Search Reddit via API**  
  - Type: HTTP Request  
  - Role: Searches Reddit posts via Reddit's public search API using the keyword, limited to 10 results sorted by relevance  
  - Configuration:  
    - Method: GET  
    - URL: https://www.reddit.com/search.json  
    - Query Parameters: q = keyword, limit = 10, sort = relevance  
  - Inputs: From Loop Over Keywords (second output)  
  - Outputs: To Merge All Search Results (second input)  
  - Edge Cases: Public Reddit API rate limits or downtime; JSON response structure changes  
  - Version: 4.2  

- **Merge All Search Results**  
  - Type: Merge Node  
  - Role: Combines results from RSS and Reddit searches into a single unified dataset for further processing  
  - Configuration: Combine mode, combining all inputs into one output array  
  - Inputs: From Search RSS Feeds and Search Reddit via API  
  - Outputs: To Prepare Articles for AI  
  - Edge Cases: Empty inputs from either source result in partial data; no deduplication performed here  
  - Version: 3.2  

---

#### 2.4 Content Preparation

**Overview:**  
Transforms heterogeneous content from RSS, Reddit, and Twitter formats into a unified article structure with relevant metadata.

**Nodes Involved:**  
- Prepare Articles for AI  

**Node Details:**

- **Prepare Articles for AI**  
  - Type: Code Node (JavaScript)  
  - Role: Parses raw API responses and RSS items to create standardized article objects with title, url, description, source, and published date  
  - Configuration:  
    - Detects Reddit data structures (posts inside `data.children` arrays or individual posts)  
    - Handles Twitter-like data if present  
    - Defaults to RSS format for others  
    - Limits description preview text length for AI prompt  
  - Inputs: From Merge All Search Results  
  - Outputs: To AI Content Filter  
  - Edge Cases: Partial or unexpected data formats; missing fields default to placeholders or current date  
  - Version: 2  

---

#### 2.5 AI Filtering

**Overview:**  
Uses a GPT-4.1-mini AI agent to analyze the prepared articles and select only those meeting strict relevance and quality criteria.

**Nodes Involved:**  
- AI Content Filter  
- OpenAI Chat Model  
- Parse AI Response  

**Node Details:**

- **AI Content Filter**  
  - Type: LangChain Agent Node (AI Language Model Agent)  
  - Role: Sends a carefully crafted prompt to the AI to select article indices that are relevant, informative, and non-promotional  
  - Configuration:  
    - Custom prompt referencing the keyword and article summaries  
    - Instructs AI to return a JSON array of article numbers matching criteria, or empty array if none  
    - Output parser enabled to process AI response  
  - Inputs: From Prepare Articles for AI  
  - Outputs: To Parse AI Response  
  - Version: 2.2  
  - Edge Cases: AI may return malformed JSON, empty results, or unexpected output - downstream parsing handles exceptions  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model Node  
  - Role: Provides the GPT-4.1-mini language model backend for the AI Content Filter node  
  - Configuration: Model preset `gpt-4.1-mini` selected; no additional options  
  - Inputs: Connected as AI model for AI Content Filter  
  - Outputs: To AI Content Filter  
  - Credentials: Requires configured OpenAI API credentials in n8n  
  - Version: 1.2  

- **Parse AI Response**  
  - Type: Code Node (JavaScript)  
  - Role: Parses the AI’s JSON output array, maps indices back to original articles, and returns only selected articles  
  - Configuration:  
    - Extracts JSON array from AI text output using regex  
    - Converts to numeric indices, filters undefined  
  - Inputs: From AI Content Filter  
  - Outputs: To Save to Google Sheets  
  - Edge Cases: Handles parse errors gracefully, returns empty list if parsing fails  
  - Version: 2  

---

#### 2.6 Output Processing & Storage

**Overview:**  
Saves the filtered, curated articles into a dedicated Google Sheet for review and further use.

**Nodes Involved:**  
- Save to Google Sheets  

**Node Details:**

- **Save to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Appends or updates rows in a designated Google Sheet tab with curated article information  
  - Configuration:  
    - Maps fields: URL, Title, Source, Added Date (current timestamp), Description  
    - Uses "appendOrUpdate" operation matching by Title to avoid duplicates  
    - Writes to a specific sheet tab by `gid=1891354699` within the same Google Sheet document as keyword retrieval  
  - Inputs: From Parse AI Response  
  - Credentials: Requires same Google Sheets credentials as Get Keywords node  
  - Edge Cases: Potential failure if sheet permissions change or if titles are not unique enough for matching  
  - Version: 4.7  

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                            | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                     |
|----------------------------|----------------------------------|--------------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger - Twice Daily | Schedule Trigger                | Starts workflow twice daily                 | None                             | Workflow Configuration          |                                                                                                                |
| Workflow Configuration      | Set                              | Holds RSS URLs config placeholder           | Schedule Trigger - Twice Daily   | Get Keywords from Google Sheets | Holds the RSS feed URLs and user-defined keywords from Google Sheets.                                          |
| Get Keywords from Google Sheets | Google Sheets                  | Reads keywords for searching                 | Workflow Configuration           | Loop Over Keywords              |                                                                                                                |
| Loop Over Keywords          | Code                             | Normalizes and outputs clean keyword list   | Get Keywords from Google Sheets  | Search RSS Feeds, Search Reddit via API |                                                                                                                |
| Search RSS Feeds            | RSS Feed Read                    | Retrieves articles from first RSS feed URL  | Loop Over Keywords               | Merge All Search Results         |                                                                                                                |
| Search Reddit via API       | HTTP Request                    | Searches Reddit for posts matching keywords | Loop Over Keywords               | Merge All Search Results         | Fetches learning-related posts from Reddit via public API based on keywords.                                   |
| Merge All Search Results    | Merge                           | Combines RSS and Reddit search results      | Search RSS Feeds, Search Reddit via API | Prepare Articles for AI         | Combines results from all content sources (RSS + Reddit) for unified AI filtering.                             |
| Prepare Articles for AI     | Code                            | Standardizes articles for AI input           | Merge All Search Results         | AI Content Filter               |                                                                                                                |
| AI Content Filter           | LangChain Agent                 | Filters articles using GPT-4.1-mini          | Prepare Articles for AI          | Parse AI Response               | AI analyzes articles and keeps only high-quality, relevant educational content.                                |
| OpenAI Chat Model           | LangChain OpenAI Chat Model     | Provides GPT-4.1-mini model                   | AI Content Filter (as AI model) | AI Content Filter               |                                                                                                                |
| Parse AI Response           | Code                            | Parses AI output, selects filtered articles  | AI Content Filter                | Save to Google Sheets           |                                                                                                                |
| Save to Google Sheets       | Google Sheets                   | Saves curated articles to Google Sheet       | Parse AI Response               | None                           | Saves the final curated articles to your connected Google Sheet automatically.                                 |
| Template Overview           | Sticky Note                     | Documentation and workflow summary            | None                            | None                           | See detailed workflow overview and setup instructions.                                                        |
| Configuration Note          | Sticky Note                     | Notes on configuration data                    | None                            | None                           | Holds the RSS feed URLs and user-defined keywords from Google Sheets.                                          |
| Fetch Reddit Note           | Sticky Note                     | Notes on Reddit content fetching               | None                            | None                           | Fetches learning-related posts from Reddit via public API based on keywords.                                   |
| Merge Note                 | Sticky Note                     | Notes on merging content sources                | None                            | None                           | Combines results from all content sources (RSS + Reddit) for unified AI filtering.                             |
| AI Filter Note              | Sticky Note                     | Notes on AI filtering                            | None                            | None                           | AI analyzes articles and keeps only high-quality, relevant educational content.                                |
| Save Results Note           | Sticky Note                     | Notes on saving curated content                  | None                            | None                           | Saves the final curated articles to your connected Google Sheet automatically.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to trigger at 8:00 and 18:00 hours daily (two trigger times).  

2. **Add a Set node named "Workflow Configuration":**  
   - Add a string parameter called `rssFeeds` with placeholder text for comma-separated RSS feed URLs.  
   - Connect Schedule Trigger output to this node.

3. **Add a Google Sheets node named "Get Keywords from Google Sheets":**  
   - Configure credentials for Google Sheets OAuth2.  
   - Set document ID to your Google Sheet containing keywords; choose sheet with `gid=0`.  
   - Connect output of "Workflow Configuration" to this node.

4. **Add a Code node named "Loop Over Keywords":**  
   - Paste the provided JavaScript code to extract and trim keywords from the Google Sheets data.  
   - Connect output of "Get Keywords from Google Sheets" to this node.

5. **Add an RSS Feed Read node named "Search RSS Feeds":**  
   - Set the RSS URL dynamically from the first RSS URL in the `rssFeeds` variable extracted from "Workflow Configuration". Use expression:  
     `={{ $('Workflow Configuration').first().json.rssFeeds.split(',')[0] }}`  
   - Connect first output of "Loop Over Keywords" (index 0) to this node.

6. **Add an HTTP Request node named "Search Reddit via API":**  
   - Method: GET  
   - URL: `https://www.reddit.com/search.json`  
   - Query parameters:  
     - `q` = `={{ $json.keyword }}` (dynamic keyword from input)  
     - `limit` = 10  
     - `sort` = relevance  
   - Configure response to be parsed as JSON.  
   - Connect second output of "Loop Over Keywords" (index 1) to this node.

7. **Add a Merge node named "Merge All Search Results":**  
   - Mode: Combine (combine all inputs)  
   - Connect outputs of "Search RSS Feeds" and "Search Reddit via API" as inputs.  

8. **Add a Code node named "Prepare Articles for AI":**  
   - Paste the provided JavaScript code that normalizes Reddit, Twitter, and RSS content into a unified format.  
   - Connect output of "Merge All Search Results" to this node.

9. **Add a LangChain Agent node named "AI Content Filter":**  
   - Configure prompt as provided, which instructs the AI to select relevant educational articles only.  
   - Enable output parser.  
   - Connect output of "Prepare Articles for AI" to this node.

10. **Add a LangChain OpenAI Chat Model node named "OpenAI Chat Model":**  
    - Model: Select `gpt-4.1-mini` from options.  
    - Connect this node as the AI language model source to "AI Content Filter".  
    - Configure OpenAI API credentials in n8n.

11. **Add a Code node named "Parse AI Response":**  
    - Paste JavaScript code that parses the AI JSON array response and maps selected results back to original articles.  
    - Connect output of "AI Content Filter" to this node.

12. **Add a Google Sheets node named "Save to Google Sheets":**  
    - Configure credentials with the same Google Sheets OAuth2 credentials as before.  
    - Set document ID to your Google Sheet for saving curated content.  
    - Choose the sheet tab (`gid=1891354699` or appropriate) for output.  
    - Use "appendOrUpdate" operation with matching column "Title" to avoid duplicates.  
    - Map columns: URL, Title, Source, Added Date (use `{{$now.toISO()}}`), Description.  
    - Connect output of "Parse AI Response" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is designed for learners, educators, and professionals to automatically curate personalized educational content using AI filtering.                                                                                                                                              | Template Overview sticky note content                                                                     |
| Requires valid Google Sheets OAuth2 credentials configured in n8n.                                                                                                                                                                                                                            | Google Sheets nodes (Get Keywords, Save Results)                                                           |
| AI filtering is powered by LangChain integration with OpenAI GPT-4.1-mini model. Proper OpenAI API credentials must be set in n8n credentials manager.                                                                                                                                         | OpenAI Chat Model node                                                                                     |
| The workflow currently only uses the first RSS feed URL from the configuration. To add more RSS feeds, extend the loop or add multiple RSS Feed Read nodes accordingly.                                                                                                                        | Workflow Configuration and Search RSS Feeds nodes                                                        |
| Reddit API calls rely on the public Reddit search endpoint; rate limits may apply. Consider adding authentication or error handling for production use.                                                                                                                                       | Search Reddit via API node                                                                                  |
| The AI prompt is designed to exclude promotional or low-quality content and selects posts that contain new, valuable learning information. Adjust the prompt to match specific learning goals or topics as needed.                                                                               | AI Content Filter node                                                                                      |
| Results are saved to Google Sheets for easy review and integration with other tools or dashboards.                                                                                                                                                                                            | Save to Google Sheets node                                                                                  |
| Sticky notes throughout the workflow provide contextual explanations and setup tips.                                                                                                                                                                                                          | Multiple sticky notes attached to relevant nodes                                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created in n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.