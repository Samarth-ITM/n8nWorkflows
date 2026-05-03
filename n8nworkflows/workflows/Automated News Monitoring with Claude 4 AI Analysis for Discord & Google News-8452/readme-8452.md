Automated News Monitoring with Claude 4 AI Analysis for Discord & Google News

https://n8nworkflows.xyz/workflows/automated-news-monitoring-with-claude-4-ai-analysis-for-discord---google-news-8452


# Automated News Monitoring with Claude 4 AI Analysis for Discord & Google News

### 1. Workflow Overview

This workflow automates a weekly news monitoring and analysis cycle focused on customized topics, leveraging Claude 4 AI for deep content analysis and delivering structured intelligence reports to a Discord channel. It integrates scheduled triggers, Google Sheets for query management, SerpAPI for Google News search, Firecrawl for article scraping, and Discord for team communication.

The workflow is logically divided into seven main functional blocks:

- **1.1 Scheduled Activation:** Triggers the workflow weekly to initiate the monitoring cycle.
- **1.2 Query Management:** Retrieves and iterates over user-defined search queries from Google Sheets.
- **1.3 News Discovery:** Performs Google News searches per query and extracts top relevant articles.
- **1.4 Article Scraping:** Scrapes full article content for the top three news results per query.
- **1.5 AI Content Analysis:** Uses Claude 4 AI to analyze and synthesize scraped article content into structured summaries.
- **1.6 Discord Message Preparation:** Aggregates analyses, segments lengthy texts to comply with Discord limits, and prepares messages.
- **1.7 Discord Delivery:** Sends introduction, segmented intelligence reports, and conclusion messages to a Discord channel using a branded bot persona.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Activation

- **Overview:**  
  Initiates the workflow execution automatically every Monday at 9 AM, enabling consistent weekly news monitoring.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow weekly based on cron expression `0 9 * * 1` (Monday 9:00 AM).  
    - Configuration: Cron expression set for weekly activation.  
    - Inputs: None (external trigger)  
    - Outputs: Triggers "Get query" node  
    - Edge Cases: Cron misconfiguration may cause missed triggers; ensure server time zone alignment.

  - **Sticky Note2 & Sticky Note3**  
    - Type: Sticky Note  
    - Role: Documentation and user instructions for scheduling setup and workflow purpose.  
    - Notes: Describes user responsibilities and system results related to scheduling.

#### 1.2 Query Management

- **Overview:**  
  Fetches search queries from a Google Sheets document, then iterates over each query for processing.

- **Nodes Involved:**  
  - Get query  
  - Loop Over Items  
  - Sticky Note4  
  - Sticky Note5

- **Node Details:**  

  - **Get query**  
    - Type: Google Sheets  
    - Role: Retrieves all search queries from the "Query" sheet in a specified Google Sheets document.  
    - Configuration: Document ID linked via OAuth2 credentials; sheet named "Query".  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Query rows to "Loop Over Items"  
    - Edge Cases: Authentication failures, empty or malformed sheet data, API rate limits.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each query individually to manage API rate limits and sequential execution.  
    - Configuration: Default batch size (processes one item at a time).  
    - Inputs: Rows from "Get query"  
    - Outputs: Passes individual queries to "Search GNews" and "Compilation données veilles".  
    - Edge Cases: Batch processing errors; ensure batch size suits API constraints.

  - **Sticky Note4 & Sticky Note5**  
    - Type: Sticky Note  
    - Role: User instructions for managing search queries and system capabilities.

#### 1.3 News Discovery

- **Overview:**  
  Executes Google News searches for each query using SerpAPI and selects the top three relevant articles.

- **Nodes Involved:**  
  - Search GNews  
  - Return URL only  
  - Sticky Note6  
  - Sticky Note7

- **Node Details:**  

  - **Search GNews**  
    - Type: SerpAPI  
    - Role: Queries Google News with the current search term, retrieving news results.  
    - Configuration: Search query set dynamically from current item’s `Query` field; uses SerpAPI credentials.  
    - Inputs: Individual query from "Loop Over Items"  
    - Outputs: Raw news search results to "Return URL only"  
    - Edge Cases: API rate limits, network errors, invalid query strings.

  - **Return URL only**  
    - Type: Code (JavaScript)  
    - Role: Processes SerpAPI results to sort articles by position and extract the top 3 with structured metadata (link, title, snippet, source, date).  
    - Configuration: Custom JS code sorts and slices news results, constructs output object with three article entries.  
    - Inputs: Raw news data from "Search GNews"  
    - Outputs: Structured top 3 articles to scraping nodes  
    - Edge Cases: Less than 3 results available, missing data fields, malformed input.

  - **Sticky Note6 & Sticky Note7**  
    - Type: Sticky Note  
    - Role: Explains system’s news retrieval details and features including relevance scoring and retry logic.

#### 1.4 Article Scraping

- **Overview:**  
  Scrapes the full text content of the top three news articles per topic using Firecrawl, preparing clean markdown content for AI analysis.

- **Nodes Involved:**  
  - Scrape article 1  
  - Scrape article 2  
  - Scrape article 3  
  - Sticky Note8  
  - Sticky Note9

- **Node Details:**  

  - **Scrape article 1 / 2 / 3**  
    - Type: Firecrawl Scraper  
    - Role: Scrapes full article content from URLs extracted previously.  
    - Configuration: Each node takes URL from respective article slot (`article1.link`, `article2.link`, `article3.link`); retry enabled with max 5 tries; error handling continues on failure to avoid halting the workflow.  
    - Inputs: Article URLs from "Return URL only"  
    - Outputs: Scraped article content in markdown format to "Rédaction veille"  
    - Edge Cases: Inaccessible URLs, paywalls, website structure changes, scraping errors.

  - **Sticky Note8 & Sticky Note9**  
    - Type: Sticky Note  
    - Role: Details about full-text scraping, formatting preservation, error handling, and multi-website compatibility.

#### 1.5 AI Content Analysis

- **Overview:**  
  Uses Claude 4 AI to analyze the scraped articles and generate structured, professional intelligence summaries in French.

- **Nodes Involved:**  
  - Rédaction veille  
  - Anthropic Chat Model6  
  - Sticky Note10  
  - Sticky Note11

- **Node Details:**  

  - **Rédaction veille**  
    - Type: Langchain Chain LLM  
    - Role: Prepares and sends a formatted prompt including the three articles’ markdown content to the Claude 4 model for synthesis.  
    - Configuration: Prompts the AI to produce a structured summary in French with specific formatting instructions (numbered articles, media/source/date/link/resume, max 3 sentences summary per article).  
    - Inputs: Scraped article markdowns and metadata  
    - Outputs: Synthesized text for aggregation  
    - Edge Cases: Missing article content leads to partial summaries; prompt failure or AI API errors.

  - **Anthropic Chat Model6**  
    - Type: Anthropic Chat LLM (Claude 4 Sonnet)  
    - Role: Executes the actual AI generation based on the prompt from "Rédaction veille".  
    - Configuration: Uses Claude 4 Sonnet model with max tokens set to 20,000; Anthropic API credentials configured.  
    - Inputs: Prompt from "Rédaction veille"  
    - Outputs: AI-generated summaries to "Rédaction veille"  
    - Edge Cases: API timeouts, token limit exceeded, auth errors.

  - **Sticky Note10 & Sticky Note11**  
    - Type: Sticky Note  
    - Role: Describes AI processing capabilities including advanced comprehension, formatting consistency, and professional tone.

#### 1.6 Discord Message Preparation

- **Overview:**  
  Aggregates all AI-generated summaries, segments the text to respect Discord's 2000-character message limit, and prepares messages for sequential delivery.

- **Nodes Involved:**  
  - Compilation données veilles  
  - Découpage message discord  
  - Loop Over Items 2  
  - Sticky Note12  
  - Sticky Note13

- **Node Details:**  

  - **Compilation données veilles**  
    - Type: Aggregate  
    - Role: Collects and combines all individual AI summaries into a single aggregated data set.  
    - Configuration: Aggregates on the "text" field of all items.  
    - Inputs: Synthesized summaries from "Loop Over Items"  
    - Outputs: Combined text to "Découpage message discord" and "Intro"  
    - Edge Cases: Large data volumes; ensure aggregation does not exceed memory limits.

  - **Découpage message discord**  
    - Type: Code (JavaScript)  
    - Role: Splits aggregated text into multiple messages respecting Discord’s 2000-character limit with a safety margin; intelligently splits by paragraphs and sentences to maintain readability.  
    - Inputs: Aggregated text from "Compilation données veilles"  
    - Outputs: Array of segmented messages to "Loop Over Items 2"  
    - Edge Cases: Very long sentences exceeding limits, malformed text inputs.

  - **Loop Over Items 2**  
    - Type: SplitInBatches  
    - Role: Iterates over the segmented messages to send them sequentially to Discord.  
    - Inputs: Messages from "Découpage message discord"  
    - Outputs: Sends individual message items downstream for Discord posting.  
    - Edge Cases: Batch size and ordering errors.

  - **Sticky Note12 & Sticky Note13**  
    - Type: Sticky Note  
    - Role: Explains text segmentation logic, Discord message constraints, and formatting preservation.

#### 1.7 Discord Delivery

- **Overview:**  
  Sends an introductory greeting, followed by segmented intelligence reports, and concludes with a closing message to a Discord channel, using a branded bot persona.

- **Nodes Involved:**  
  - Intro  
  - Veille  
  - Conclusion  
  - Sticky Note14  
  - Sticky Note15

- **Node Details:**  

  - **Intro**  
    - Type: Discord Node  
    - Role: Sends a fixed, branded introduction message to the target Discord channel to announce the start of the weekly intelligence report.  
    - Configuration: Uses Discord bot credentials; posts to specific guild and channel IDs; static message content with branding and excitement.  
    - Inputs: Triggered after "Compilation données veilles" and "Découpage message discord" (parallel with first segment).  
    - Outputs: None downstream except starts message flow.  
    - Edge Cases: Discord API rate limits, permission errors.

  - **Veille**  
    - Type: Discord Node  
    - Role: Posts each segmented intelligence message to Discord channel, suppressing embeds to maintain plain text appearance.  
    - Configuration: Dynamic content from segmented messages; uses same Discord bot and channel as "Intro".  
    - Inputs: Messages from "Loop Over Items 2"  
    - Outputs: None downstream (feeds into Conclusion node)  
    - Edge Cases: Message delivery failures, API limits.

  - **Conclusion**  
    - Type: Discord Node  
    - Role: Sends a closing message to indicate completion of the monitoring cycle, maintaining branded voice.  
    - Configuration: Static content; executes once per workflow run; uses same Discord bot and channel.  
    - Inputs: Triggered after all segmented messages sent ("Loop Over Items 2")  
    - Outputs: None  
    - Edge Cases: Timing issues if messages are delayed or fail to send.

  - **Sticky Note14 & Sticky Note15**  
    - Type: Sticky Note  
    - Role: Describes branded, engaging communication style, sequential delivery, and team engagement goals.

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                                  | Input Node(s)          | Output Node(s)                    | Sticky Note                                                             |
|-----------------------|-----------------------------------|-------------------------------------------------|------------------------|----------------------------------|-------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                  | Weekly workflow initiation                      | None                   | Get query                       | # Phase 1: Weekly Schedule Activation                                   |
| Sticky Note2           | Sticky Note                      | User guidance on scheduling                      | None                   | None                            | # Phase 1: Weekly Schedule Activation                                   |
| Sticky Note3           | Sticky Note                      | Detailed schedule and system overview            | None                   | None                            | Config instructions and system capabilities for scheduling             |
| Get query              | Google Sheets                    | Fetches monitoring queries from Google Sheets   | Schedule Trigger       | Loop Over Items                 | # Phase 2: Search Query Management and Topic Processing                 |
| Loop Over Items        | SplitInBatches                  | Iterates over each query                          | Get query              | Search GNews, Compilation données veilles | # Phase 2: Search Query Management and Topic Processing                 |
| Sticky Note4           | Sticky Note                      | Query management phase title                      | None                   | None                            | # Phase 2: Search Query Management and Topic Processing                 |
| Sticky Note5           | Sticky Note                      | Query management instructions                     | None                   | None                            | Instructions for managing queries                                      |
| Search GNews           | SerpAPI                         | Executes Google News searches per query          | Loop Over Items        | Return URL only                 | # Phase 3: Comprehensive News Discovery and Collection                 |
| Return URL only        | Code                            | Extracts and selects top 3 news articles         | Search GNews           | Scrape article 1               | # Phase 3: Comprehensive News Discovery and Collection                 |
| Sticky Note6           | Sticky Note                      | News discovery phase title                        | None                   | None                            | # Phase 3: Comprehensive News Discovery and Collection                 |
| Sticky Note7           | Sticky Note                      | News discovery features and details               | None                   | None                            | Details on news search and ranking                                     |
| Scrape article 1       | Firecrawl                       | Scrapes full content of first article             | Return URL only        | Scrape article 2               | # Phase 4: Deep Content Analysis and Article Scraping                  |
| Scrape article 2       | Firecrawl                       | Scrapes full content of second article            | Scrape article 1       | Scrape article 3               | # Phase 4: Deep Content Analysis and Article Scraping                  |
| Scrape article 3       | Firecrawl                       | Scrapes full content of third article             | Scrape article 2       | Rédaction veille               | # Phase 4: Deep Content Analysis and Article Scraping                  |
| Sticky Note8           | Sticky Note                      | Article scraping phase title                      | None                   | None                            | # Phase 4: Deep Content Analysis and Article Scraping                  |
| Sticky Note9           | Sticky Note                      | Article scraping details and error handling       | None                   | None                            | Scraping capabilities and error strategies                             |
| Rédaction veille       | Langchain Chain LLM             | Prepares and sends prompt for AI content analysis | Scrape article 3       | Loop Over Items                | # Phase 5: AI-Powered Content Analysis and Synthesis                   |
| Anthropic Chat Model6  | Anthropic Chat Model            | Executes Claude 4 AI analysis                      | Rédaction veille       | Rédaction veille               | # Phase 5: AI-Powered Content Analysis and Synthesis                   |
| Sticky Note10          | Sticky Note                      | AI analysis phase title                           | None                   | None                            | # Phase 5: AI-Powered Content Analysis and Synthesis                   |
| Sticky Note11          | Sticky Note                      | AI analysis instructions and features             | None                   | None                            | AI capabilities and formatting standards                              |
| Compilation données veilles | Aggregate                    | Aggregates AI summaries for message preparation  | Loop Over Items        | Découpage message discord, Intro | # Phase 6: Content Optimization for Discord Delivery                  |
| Découpage message discord | Code                         | Segments aggregated text to comply with Discord limits | Compilation données veilles | Loop Over Items 2             | # Phase 6: Content Optimization for Discord Delivery                  |
| Loop Over Items 2      | SplitInBatches                  | Iterates over segmented messages                   | Découpage message discord | Veille, Conclusion           | # Phase 6: Content Optimization for Discord Delivery                  |
| Sticky Note12          | Sticky Note                      | Discord message preparation phase title           | None                   | None                            | # Phase 6: Content Optimization for Discord Delivery                  |
| Sticky Note13          | Sticky Note                      | Discord message segmentation details               | None                   | None                            | Text splitting and message flow explanations                         |
| Intro                  | Discord                         | Sends introduction message to Discord channel     | Compilation données veilles, Découpage message discord | None             | # Phase 7: Automated Team Communication and Report Delivery           |
| Veille                 | Discord                         | Sends intelligence messages in sequence            | Loop Over Items 2      | Loop Over Items 2 (loops back) | # Phase 7: Automated Team Communication and Report Delivery           |
| Conclusion             | Discord                         | Sends concluding message to Discord channel       | Loop Over Items 2      | None                            | # Phase 7: Automated Team Communication and Report Delivery           |
| Sticky Note14          | Sticky Note                      | Discord delivery phase title                       | None                   | None                            | # Phase 7: Automated Team Communication and Report Delivery           |
| Sticky Note15          | Sticky Note                      | Discord delivery instructions and features         | None                   | None                            | Branded communication and team engagement details                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 9 * * 1` (Monday 9:00 AM)  
   - No credentials needed  

2. **Add Google Sheets node ("Get query"):**  
   - Connect from Schedule Trigger  
   - Configure with Google Sheets OAuth2 credentials  
   - Set Document ID to your Google Sheets URL containing search queries  
   - Set Sheet Name to `"Query"`  
   - Retrieve all rows from this sheet  

3. **Add SplitInBatches node ("Loop Over Items"):**  
   - Connect from "Get query"  
   - Use default batch size (1) to process queries sequentially  

4. **Add SerpAPI node ("Search GNews"):**  
   - Connect from "Loop Over Items"  
   - Configure with SerpAPI credentials  
   - Set operation to `google_news`  
   - Set query parameter to current batch item’s `Query` field (`={{ $json.Query }}`)  
   - Enable retry on failure  

5. **Add Code node ("Return URL only"):**  
   - Connect from "Search GNews"  
   - Insert JavaScript to extract and sort top 3 news articles by position, including link, title, snippet, source, and date  
   - Return structured object with articles info  

6. **Add three Firecrawl nodes ("Scrape article 1", "Scrape article 2", "Scrape article 3"):**  
   - Connect sequentially: "Return URL only" → "Scrape article 1" → "Scrape article 2" → "Scrape article 3"  
   - Configure each with Firecrawl credentials  
   - Set each URL parameter dynamically from respective article link in "Return URL only" output  
   - Enable max retries 5, continue on error  
   - Output markdown content from scraped articles  

7. **Add Langchain Chain LLM node ("Rédaction veille"):**  
   - Connect from "Scrape article 3"  
   - Configure prompt to include URLs, dates, and markdown content of all three articles  
   - Include instructions to analyze and produce structured summaries in French with specified formatting  
   - Use OpenAI or Anthropic API credentials as appropriate (here Anthropic Claude 4 Sonnet)  

8. **Add Anthropic Chat Model node ("Anthropic Chat Model6"):**  
   - Connect as AI language model source for "Rédaction veille"  
   - Set model to `claude-sonnet-4-20250514` with max tokens 20,000  
   - Attach Anthropic API credentials  

9. **Connect "Rédaction veille" output back to "Loop Over Items" node:**  
   - This enables batching over all queries for synthesis  

10. **Add Aggregate node ("Compilation données veilles"):**  
    - Connect from "Loop Over Items"  
    - Aggregate all AI-generated summaries into a single text field  

11. **Add Code node ("Découpage message discord"):**  
    - Connect from Aggregate node  
    - Implement JavaScript to split long text into Discord message-size chunks with a 50-character safety margin  
    - Split by paragraphs and sentences to maintain readability  

12. **Add SplitInBatches node ("Loop Over Items 2"):**  
    - Connect from "Découpage message discord"  
    - Process each message chunk sequentially for Discord posting  

13. **Add Discord node ("Intro"):**  
    - Connect from Aggregate node (parallel to "Découpage message discord")  
    - Configure Discord bot credentials (OAuth2 or bot token)  
    - Set guild and channel IDs  
    - Use static branded introduction message  

14. **Add Discord node ("Veille"):**  
    - Connect from "Loop Over Items 2"  
    - Configure to post segmented intelligence report messages dynamically from each batch item’s `message` field  
    - Set option to suppress embeds for plain text presentation  

15. **Add Discord node ("Conclusion"):**  
    - Connect from "Loop Over Items 2" (trigger after all messages sent)  
    - Configure with same Discord credentials, guild and channel  
    - Use static branded conclusion message  
    - Set to execute once per workflow run  

16. **Add Sticky Notes at appropriate nodes:**  
    - For user guidance and documentation, add sticky notes summarizing each phase, configuration reminders, and operational notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow uses a branded bot persona "Claptrap" for engaging and consistent team communication in Discord.                      | Discord message nodes Intro, Veille, Conclusion                                                                |
| Google Sheets document must have a sheet named "Query" with monitoring topics/keywords listed for search.                           | "Get query" node configuration                                                                                  |
| SerpAPI account is required for Google News scraping with rate limits and API keys management.                                      | "Search GNews" node configuration                                                                                |
| Firecrawl scraper handles various website formats with retry logic to ensure content extraction reliability.                       | Article scraping nodes ("Scrape article 1", etc.)                                                                |
| Claude 4 Sonnet model from Anthropic is used for advanced AI content synthesis and requires appropriate API credentials.           | "Anthropic Chat Model6" node configuration                                                                       |
| Discord API rate limits and message length restrictions (2000 characters) are handled via custom code to split long messages.      | "Découpage message discord" node                                                                                  |
| Workflow is designed for extensibility: new queries can be added in Google Sheets without modifying the workflow.                   | Query management block                                                                                             |
| The workflow respects error handling best practices: retries on API calls, continues on scraping errors, and batch processing.     | Various nodes with retry and error handling configuration                                                         |
| For best operation, ensure all OAuth and API credentials are valid and have necessary permissions (Google Sheets, SerpAPI, Firecrawl, Discord). | Credentials management                                                                                            |
| Additional details and user instructions are embedded as sticky notes throughout the workflow for operational clarity.             | Sticky notes nodes at each phase                                                                                  |

---

**Disclaimer:** The provided description and analysis are based exclusively on an automated workflow implemented in n8n. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.