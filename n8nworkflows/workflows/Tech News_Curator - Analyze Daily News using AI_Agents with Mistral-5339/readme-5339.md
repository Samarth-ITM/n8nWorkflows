Tech News_Curator - Analyze Daily News using AI_Agents with Mistral

https://n8nworkflows.xyz/workflows/tech-news_curator---analyze-daily-news-using-ai_agents-with-mistral-5339


# Tech News_Curator - Analyze Daily News using AI_Agents with Mistral

---

### 1. Workflow Overview

This workflow, titled **"Tech News Curator - Analyze Daily News using AI Agents and Generate Highlights using Mistral V0"**, automates the aggregation, filtering, AI-driven analysis, and summarization of technology and science news from multiple online sources on a daily basis. It is designed for news curators, researchers, or enthusiasts who want a concise, relevant digest of the most important tech-related news each weekday, delivered through intelligent AI summarization.

The workflow is logically divided into the following key blocks:

- **1.1 News Gathering and Standardization:** Scheduled triggers initiate RSS feed and news search queries from various reputable sources, retrieving raw news items.
- **1.2 Data Preparation and Filtering:** Standardizes data fields, removes unwanted content based on keywords, and filters news by recency (daily news).
- **1.3 Content Extraction:** Retrieves full content of news articles, especially when RSS feeds provide only metadata.
- **1.4 AI News Analysis and Selection:** Uses an AI agent powered by Mistral Cloud chat models to analyze titles and select the top 10 most relevant news articles with a summary and headline.
- **1.5 Detailed News Summarization:** Fetches and summarizes the full content of selected top news articles using AI, producing concise, stylistically varied one-sentence summaries.
- **1.6 Final Aggregation:** Aggregates all final summaries into a structured output for downstream consumption or distribution.

---

### 2. Block-by-Block Analysis

---

#### 2.1 News Gathering and Standardization

**Overview:**  
This block triggers the workflow on weekdays, collects news items from multiple RSS feeds and DuckDuckGo news search, then standardizes the fields for consistent processing.

**Nodes Involved:**  
- Schedule  
- lobste.rs Feed  
- ScienceDaily Feed  
- TheHackersNews* Feed  
- AlternativeTo Feed  
- MIT Technology Review Feed  
- Silicone Republic Technology Feed  
- Silicone Republic Science Feed (disabled)  
- DuckDuckGo  
- Filter Fields (1 to 9)  
- Merge

**Node Details:**

- **Schedule**  
  - Type: Schedule trigger  
  - Role: Initiates the workflow every weekday (Mon-Fri) at 12:00 PM  
  - Config: Trigger days set to weekdays, hour 12  
  - Possible Failures: Misconfiguration could cause no trigger or incorrect timing  

- **RSS Feed Nodes (lobste.rs, ScienceDaily, TheHackersNews*, AlternativeTo, MIT Technology Review, Silicone Republic Technology, Silicone Republic Science [disabled])**  
  - Type: RSS Feed Read  
  - Role: Fetch latest news items from respective RSS URLs  
  - Config: URLs set per source, SSL verification enabled, continue on errors to avoid breaking workflow if a feed fails  
  - Notes: Silicone Republic Science feed is disabled—can be enabled for additional science content  
  - Failure Modes: Network issues, feed format changes, timeouts  

- **DuckDuckGo**  
  - Type: DuckDuckGo News Search (custom node)  
  - Role: Search for technology news via DuckDuckGo, capped at 100 results, safe search off, region wildcard  
  - Config: Enables caching (5 min TTL) to reduce API calls  
  - Note: Requires installation of DuckDuckGo node  
  - Failures: API changes, rate limits  

- **Filter Fields (1-9)**  
  - Type: Set node  
  - Role: Standardize extracted fields from each feed/search into uniform properties: title, pubDate (or date), link (or url), and content  
  - Config: Extracts relevant fields, assigns empty string for missing content where applicable  
  - Note: Some Filter Fields nodes disabled (e.g., Filter Fields 2 and 8) – likely legacy or experimental  
  - Failures: Field mismatches or missing data could cause null references downstream  

- **Merge**  
  - Type: Merge (Multiple Inputs)  
  - Role: Combines all standardized news lists into a single unified list  
  - Config: Set for 9 inputs to parallel connect all Filter Fields outputs  
  - Note: Risk of premature execution if some inputs are delayed; ensure all are ready for complete merge  
  - Failures: Partial inputs may cause incomplete data aggregation  

---

#### 2.2 Data Preparation and Filtering

**Overview:**  
Filters news items by publication date (only today’s news), removes entries containing unwanted keywords or empty content to clean the dataset before AI analysis.

**Nodes Involved:**  
- Filter by Datetime  
- Remove Certain Content  
- Sticky Note (Filtering Step)

**Node Details:**

- **Filter by Datetime**  
  - Type: Filter node  
  - Role: Passes only news items with publication date after today's date (effectively today’s news)  
  - Config: Uses expression comparing pubDate with current date (`$today.minus({ days: 0 })`)  
  - Failures: Incorrect date formats can cause filtering failure  

- **Remove Certain Content**  
  - Type: Filter node  
  - Role: Removes news containing unwanted keywords (`opinion|rumour|layoff|porn|marijuana`) in link, title or content; also ensures content is not empty  
  - Config: Regex-based negative filtering on multiple fields  
  - Failures: Overly broad regex could remove legitimate news; missing content fields could cause false negatives  

- **Sticky Note** (Filtering Step)  
  - Contains guidance on adding/removing keywords for filtering  
  - Helps users understand filtering rationale  

---

#### 2.3 Content Extraction

**Overview:**  
Retrieves full article content when not provided directly by RSS feeds, ensuring AI agents have substantive text to analyze.

**Nodes Involved:**  
- HTTP Request (to fetch page)  
- Markdown Conversion (to convert HTML content to markdown/plain text)  
- Rejoin Data  
- Call Get Webpage Content to Data (Sub-Workflow execution)  
- Remove Elements with Empty Content  
- Sticky Note (Sub-Workflow Explanation)

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request node  
  - Role: Fetches raw HTML content of news articles from their links (notably for lobste.rs feed)  
  - Config: 10s timeout, continues on error to avoid stopping workflow  
  - Failures: Timeout, 404 errors, network failures  

- **Markdown Conversion**  
  - Type: Markdown node  
  - Role: Converts fetched HTML into markdown or plain text to standardize content representation  
  - Failures: Malformed HTML may produce poor markdown output  

- **Rejoin Data**  
  - Type: Set node  
  - Role: Reassigns title, pubDate, link, and now the extracted content back into standard fields  
  - Failures: Data mismatches if prior nodes fail  

- **Call Get Webpage Content to Data**  
  - Type: Execute Workflow node (sub-workflow)  
  - Role: Invokes a sub-workflow to retrieve webpage content when not directly available  
  - Config: Passes title, pubDate, link as inputs; expects processed content back  
  - Failure: Sub-workflow errors propagate here; missing sub-workflow disables content extraction  

- **Remove Elements with Empty Content**  
  - Type: Filter node  
  - Role: Filters out any items that have empty content after extraction  
  - Failures: Could filter out legitimate articles if content extraction fails silently  

- **Sticky Note (Sub-Workflow)**  
  - Explains purpose of sub-workflow to deal with incomplete content from original sources  

---

#### 2.4 AI News Analysis and Selection

**Overview:**  
Uses an AI agent with Mistral Cloud chat model to analyze the aggregated news titles, produce an overall headline and summary for the day, and select the top 10 most relevant news articles with their links.

**Nodes Involved:**  
- Aggregate  
- News Analyzer (LangChain Agent)  
- JSON Output Parser  
- Split Out News Data  
- Sticky Note (News Selection)

**Node Details:**

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates all filtered news into a single array with fields title, pubDate, and link  
  - Failures: Empty input leads to AI receiving no data  

- **News Analyzer**  
  - Type: LangChain Agent node  
  - Role: Analyzes article titles to produce: a fitting title for the day, a summary, and the top 10 news articles (one per website)  
  - Config:  
    - System message instructs prioritization of technological, scientific, applied math, and data science news  
    - Enforces no duplicate websites in top 10 if possible  
    - Uses Mistral Cloud chat model instead of OpenAI for API friendliness  
    - Has output parser for JSON structured response (ensures consistency)  
  - Retries: Up to 2 max tries on failure  
  - Failures: API errors, malformed AI output, exceeding token limits  

- **JSON Output Parser**  
  - Type: Output parser node  
  - Role: Enforces structured JSON output according to a defined schema  
  - Failure: Parsing errors if AI output deviates  

- **Split Out News Data**  
  - Type: Split Out node  
  - Role: Extracts the top 10 news array from AI output for further processing  
  - Failure: Empty or malformed array leads to downstream issues  

- **Sticky Note (News Selection)**  
  - Explains AI’s role, use of Mistral Cloud, and importance of JSON output parsing for consistent AI answers  

---

#### 2.5 Detailed News Summarization

**Overview:**  
For each top news article, joins its original content, summarizes the content into concise sentences using AI, and prepares structured summaries.

**Nodes Involved:**  
- Join Outputs  
- Website Summarizer (LangChain Agent)  
- JSON Output Parser 2  
- Final Aggregate  
- Sticky Note (News Summarizer)

**Node Details:**

- **Join Outputs**  
  - Type: Merge node (advanced, merge by fields)  
  - Role: Joins top news metadata with their full textual content by matching link to website field  
  - Failure: Join mismatches cause missing content for some news  

- **Website Summarizer**  
  - Type: LangChain Agent node  
  - Role: Generates a one-sentence summary for each news article’s content, with stylistic variability (formal, informal, cheeky, serious)  
  - Config: Uses Mistral Cloud chat model and JSON output parser for structured response  
  - Retries: 2 retries on failure  
  - Failures: AI output parsing errors or incomplete summaries  

- **JSON Output Parser 2**  
  - Type: Output parser node  
  - Role: Parses AI output to ensure each summary conforms to the JSON schema with title, author, link, and summary fields  
  - Failure: Parsing errors if AI response malformed  

- **Final Aggregate**  
  - Type: Aggregate node  
  - Role: Combines all summarized news into a single array under “output” field for final output  
  - Failure: Empty input results in no final summaries  

- **Sticky Note (News Summarizer)**  
  - Details the role of each node in final summarization and notes that improperly structured AI responses are discarded  

---

#### 2.6 Final Aggregation and Output Preparation

**Overview:**  
Final merging of all summarized news data into combined output for downstream use or export.

**Nodes Involved:**  
- Final Join Outputs

**Node Details:**

- **Final Join Outputs**  
  - Type: Merge node (combine by position)  
  - Role: Combines the final aggregated summaries with any other outputs if applicable  
  - Failure: No outputs connected downstream indicates this is likely the workflow’s end point  

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                                     | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                   |
|-------------------------------|---------------------------------------|----------------------------------------------------|-------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule                      | Schedule Trigger                      | Starts workflow on weekdays at 12:00 PM            | -                                   | lobste.rs Feed, ScienceDaily Feed, etc. | # Tech News Gathering block notes                                                             |
| lobste.rs Feed                | RSS Feed Read                        | Fetches news from lobste.rs RSS feed                | Schedule                            | Filter Fields                      | Notes on standardizing RSS outputs                                                            |
| ScienceDaily Feed             | RSS Feed Read                        | Fetches news from ScienceDaily RSS feed             | Schedule                            | Filter Fields 3                    | Notes on standardizing RSS outputs                                                            |
| TheHackersNews* Feed          | RSS Feed Read                        | Fetches news from TheHackersNews RSS feed           | Schedule                            | Filter Fields 4                    | Notes on standardizing RSS outputs                                                            |
| AlternativeTo Feed            | RSS Feed Read                        | Fetches news from AlternativeTo RSS feed            | Schedule                            | Filter Fields 5                    | Notes on standardizing RSS outputs                                                            |
| MIT Technology Review Feed    | RSS Feed Read                        | Fetches news from MIT Technology Review RSS feed    | Schedule                            | Filter Fields 6                    | Notes on standardizing RSS outputs                                                            |
| Silicone Republic Technology Feed | RSS Feed Read                   | Fetches news from Silicone Republic Technology feed | Schedule                            | Filter Fields 7                    | Notes on standardizing RSS outputs                                                            |
| Silicone Republic Science Feed (disabled) | RSS Feed Read            | Fetches news from Silicone Republic Science feed    | Schedule                            | Filter Fields 8 (disabled)        | Notes on standardizing RSS outputs                                                            |
| DuckDuckGo                   | DuckDuckGo News Search               | Searches news via DuckDuckGo (technology query)     | Schedule                           | Filter Fields 9                   | Notes on DuckDuckGo node installation                                                         |
| Filter Fields (1-9)           | Set                                 | Standardizes RSS/news search output fields          | Respective Feed/Search nodes        | Merge or next processing nodes    | Notes on Field Prep for standardization                                                       |
| Merge                        | Merge                               | Combines all standardized news entries               | Filter Fields nodes                 | Filter by Datetime                | Notes on merging feeds                                                                        |
| Filter by Datetime            | Filter                             | Filters news to only today’s news                     | Merge                             | Remove Certain Content            | Filtering Step note                                                                           |
| Remove Certain Content        | Filter                             | Removes news with unwanted keywords/content          | Filter by Datetime                | Aggregate, Join Outputs           | Filtering Step note                                                                           |
| HTTP Request                 | HTTP Request                       | Fetches full HTML content for news articles          | Filter Fields                    | Markdown Conversion              | Content extraction explanation                                                               |
| Markdown Conversion           | Markdown                          | Converts HTML content to markdown/plain text          | HTTP Request                    | Rejoin Data                     | Content extraction explanation                                                               |
| Rejoin Data                  | Set                               | Combines title, pubDate, link, and content           | Markdown Conversion              | Merge                           | Content extraction explanation                                                               |
| Call Get Webpage Content to Data | Execute Workflow (Sub-Workflow) | Calls sub-workflow to fetch webpage content          | Filter Fields 9                 | Remove Elements with Empty Content | Sub-workflow explanation note                                                                |
| Remove Elements with Empty Content | Filter                       | Removes news items with empty content after extraction | Call Get Webpage Content to Data | Merge                         | Content extraction explanation                                                               |
| Aggregate                    | Aggregate                         | Aggregates news field data for AI analysis            | Remove Certain Content           | News Analyzer                   | News Selection note                                                                           |
| News Analyzer                | LangChain Agent                  | AI agent that analyzes news titles and selects top 10 | Aggregate                      | Split Out News Data, Final Join Outputs | News Selection note                                                                           |
| JSON Output Parser            | AI Output Parser                | Ensures AI output conforms to expected JSON schema    | News Analyzer                   | News Analyzer (ai_outputParser) | News Selection note                                                                           |
| Split Out News Data          | Split Out                       | Extracts top 10 news array from AI output             | News Analyzer                   | Join Outputs                   | News Summarizer note                                                                         |
| Join Outputs                 | Merge                           | Joins top news metadata with full content             | Split Out News Data, Remove Certain Content | Website Summarizer           | News Summarizer note                                                                         |
| Website Summarizer           | LangChain Agent                | AI agent that summarizes full news content            | Join Outputs                   | Final Aggregate                | News Summarizer note                                                                         |
| JSON Output Parser 2         | AI Output Parser              | Ensures structured JSON output from Website Summarizer | Website Summarizer            | Website Summarizer (ai_outputParser) | News Summarizer note                                                                         |
| Final Aggregate              | Aggregate                     | Aggregates summarized news for final output           | Website Summarizer             | Final Join Outputs             | News Summarizer note                                                                         |
| Final Join Outputs           | Merge                         | Combines final aggregated summaries                    | Final Aggregate, News Analyzer (second output) | -                             |                                                                                               |
| Sticky Notes (multiple)      | Sticky Note                   | Documentation and guidance at various stages           | -                             | -                             | Contains explanations and advice on filtering, field prep, AI usage, and sub-workflow details |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**  
   - Type: Schedule  
   - Configure to trigger every weekday (Mon-Fri) at 12:00 PM

2. **Add RSS Feed Nodes:**  
   - Create one RSS Feed Read node per source with following URLs:  
     - lobste.rs: https://lobste.rs/rss  
     - ScienceDaily: https://www.sciencedaily.com/rss/all.xml  
     - TheHackersNews*: https://feeds.feedburner.com/TheHackersNews  
     - AlternativeTo: https://feed.alternativeto.net/news/all/  
     - MIT Technology Review: https://www.technologyreview.com/topic/artificial-intelligence/feed/  
     - Silicone Republic Technology: https://www.siliconrepublic.com/category/technology/feed/  
     - Silicone Republic Science (optional, disabled by default): https://www.siliconrepublic.com/category/science/feed/  
   - Set “Continue on Error” to true  
   - Always output data enabled  

3. **Add DuckDuckGo News Search Node:**  
   - Type: n8n-nodes-duckduckgo-search.duckDuckGo  
   - Configure to search news with query “technology”  
   - Enable caching with TTL 300s  
   - Set region to “wt-wt”, max results 100, safe search off  

4. **Connect Schedule output to all feed nodes and DuckDuckGo node**

5. **Create Filter Fields Set Nodes (one per feed):**  
   - For each feed output, create a Set node that extracts and standardizes fields:  
     - title: from feed title  
     - pubDate or date: normalized publication date  
     - link or url: news article URL  
     - content: empty string if missing (to be filled later)  
   - Disabled nodes can be omitted or enabled as needed  

6. **Merge all Filter Fields nodes:**  
   - Use a Merge node with numberInputs equal to the number of feeds (9 total)  
   - Connect all Filter Fields nodes to the Merge inputs  

7. **Add Filter by Datetime node:**  
   - Configure to validate that pubDate is after or equal to today’s date (use expression referencing $today)  

8. **Add Remove Certain Content filter node:**  
   - Set to exclude items with keywords in title, link, or content: opinion, rumour, layoff, porn, marijuana  
   - Also exclude items with empty content  

9. **Add Aggregate node:**  
   - Aggregate all news items into a single array including only title, pubDate, and link fields  

10. **Add News Analyzer (LangChain Agent) node:**  
    - Use Mistral Cloud Chat Model credentials  
    - Configure system message to instruct AI to analyze titles, generate daily news headline and summary, and select top 10 news items with priority rules (technological, scientific, applied math, data science, no duplicates from same website)  
    - Use output parser with JSON schema for structured output  
    - Set max retries to 2  

11. **Add JSON Output Parser node:**  
    - Schema matches the expected AI output with title, summary, top_10_news array of {title, website}  

12. **Add Split Out node:**  
    - Extract top_10_news array from AI output for further processing  

13. **Add Join Outputs node:**  
    - Merge top news metadata with full news content by matching “link” to “website” field  

14. **Add Website Summarizer (LangChain Agent) node:**  
    - Use Mistral Cloud Chat Model  
    - Task: summarize each article’s content in a concise, stylistically varied sentence  
    - Use output parser with JSON schema for title, author, link, summary  
    - Set max retries to 2  

15. **Add JSON Output Parser 2:**  
    - Validate AI summaries against defined JSON schema  

16. **Add Final Aggregate node:**  
    - Aggregate all summarized news into a single array under “output”  

17. **Add Final Join Outputs node:**  
    - Combine all final aggregated summaries and other outputs if needed  

18. **For Content Extraction:**  
    - Add HTTP Request node to fetch full HTML from article links (used for feeds like lobste.rs)  
    - Add Markdown Conversion node to convert HTML to markdown/plain text  
    - Add Rejoin Data node to unify title, pubDate, link, and content  
    - Add Execute Workflow node to call a sub-workflow “Get Webpage Content” with inputs: title, pubDate, link (sub-workflow responsible for extracting content when missing)  
    - Add Remove Elements with Empty Content filter node to remove news items with no content  
    - Connect all these nodes in sequence  
    - Finally, connect back to the Merge node that combines all feeds  

19. **Set all nodes to continue on error as appropriate to avoid workflow halts**

20. **Create Sticky Notes at key points for documentation and user guidance** (optional but recommended)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Use of Mistral Cloud Chat Model is recommended due to free tier, developer-friendliness, and robust API for hobbyists.                                                        | Mistral Cloud API                                                                                    |
| JSON Output Parsers are essential to enforce consistent and structured AI outputs, preventing incomplete, malformed, or gibberish responses.                                   | n8n LangChain Output Parser documentation                                                          |
| The sub-workflow “Get Webpage Content” is required to fetch and parse article content when not provided by the RSS feed, ensuring AI has sufficient text for summarization.     | https://docs.n8n.io/workflows/subworkflow-conversion/                                              |
| Filtering by keywords can be customized to adapt to different content policies or focus areas, e.g., removing gossip, adult content, or unreliable news.                      | See Filtering Step Sticky Note                                                                     |
| The workflow is scheduled to run weekdays at noon, but this can be adjusted in the Schedule node to suit different operational requirements.                                  | Schedule node settings                                                                             |
| The DuckDuckGo search node is a custom node and must be installed separately in n8n to work properly.                                                                           | https://docs.n8n.io/integrations/custom-nodes/                                                     |
| Risk of incomplete data if any RSS feed or API is down; the workflow is configured to continue execution on individual node errors to maximize data collection completeness.    | Node error handling settings                                                                       |

---

**Disclaimer:** The above documentation is generated from an automated n8n workflow export. All data processed is legal and public. The workflow respects all content policies and does not process illegal or protected content.

---