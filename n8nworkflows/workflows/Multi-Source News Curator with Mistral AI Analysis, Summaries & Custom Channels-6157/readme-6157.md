Multi-Source News Curator with Mistral AI Analysis, Summaries & Custom Channels

https://n8nworkflows.xyz/workflows/multi-source-news-curator-with-mistral-ai-analysis--summaries---custom-channels-6157


# Multi-Source News Curator with Mistral AI Analysis, Summaries & Custom Channels

### 1. Workflow Overview

This workflow, titled **"Multi-Source News Curator with Mistral AI Analysis, Summaries & Custom Channels"**, is designed to aggregate news articles and video content from multiple sources, analyze and summarize them using advanced AI (Mistral AI and LangChain agents), and deliver curated content through customizable output channels such as email, Telegram, or WhatsApp. It targets users or organizations needing an automated, flexible news curation pipeline that supports diverse inputs, quality assurance, content translation, and multi-format output.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Scheduling:** Triggers for starting the workflow via schedule, email, or webhook; and configuration of global workflow variables.
- **1.2 News & Video Content Collection:** Multi-source ingestion of news via RSS feeds, SerpAPI searches, and video feeds, including custom feed handling and content retrieval.
- **1.3 Content Filtering and Deduplication:** Filters articles and videos by date, removes duplicates and empty content, and shuffles to ensure freshness.
- **1.4 AI Analysis and Quality Assurance:** Uses Mistral AI and LangChain agents to analyze, QA, and summarize news articles; includes JSON output parsing and error handling.
- **1.5 Content Aggregation and Post-Processing:** Combines summaries, converts results into markdown and text, determines language and tone, and optionally translates content.
- **1.6 Output & Distribution:** Routes curated content to different delivery channels such as email, Telegram, WhatsApp, or webhook responses; includes file writing for archival.
- **1.7 Utility and Control Nodes:** Includes nodes for looping, branching (If, Switch), and error handling to maintain workflow robustness.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

**Overview:**  
This block manages initiating the workflow execution, setting global constants and variables, and providing alternative entry points.

**Nodes Involved:**  
- Schedule  
- Email Trigger (IMAP)  
- Webhook: Call Workflow Activation (disabled)  
- Configure Workflow Args  
- Configure Workflow Args (Alternative)  
- Custom Credentials  
- Configure Workflow Args (Custom Credentials)  
- When clicking ‘Execute workflow’ (Manual Trigger)  

**Node Details:**

- **Schedule**  
  - Type: Schedule Trigger  
  - Config: Default scheduling parameters to trigger the workflow automatically at set intervals.  
  - Input: None (trigger node)  
  - Output: Starts the workflow flow.  
  - Potential failures: None typical, except scheduling misconfiguration.  

- **Email Trigger (IMAP)**  
  - Type: Email Read IMAP  
  - Config: Watches an email inbox for incoming emails to trigger the workflow.  
  - Input: None (trigger node)  
  - Output: Email data to trigger processing.  
  - Potential failures: Authentication errors, connection timeouts.  

- **Webhook: Call Workflow Activation** (Disabled)  
  - Type: Webhook  
  - Config: Provides a REST endpoint to externally trigger the workflow via HTTP POST requests. Disabled by default.  
  - Input: HTTP request  
  - Output: Workflow start  
  - Potential failures: HTTP errors, misconfiguration of URL or firewall restrictions.  

- **Configure Workflow Args**  
  - Type: Global Constants  
  - Config: Stores global variables for easy configuration of workflow variants and templates.  
  - Input: Trigger nodes  
  - Output: Provides configuration variables downstream.  
  - Notes: Helps avoid hardcoding variables directly in nodes; facilitates sharing templates safely.  

- **Configure Workflow Args (Alternative)**  
  - Type: Set  
  - Config: Alternative to global constants node for setting workflow variables if global constants are not preferred.  
  - Input: Trigger nodes  
  - Output: Variables for downstream nodes.  

- **Custom Credentials**  
  - Type: Code  
  - Config: Contains code to manage or set custom credentials for external services.  
  - Input: Workflow args nodes  
  - Output: Provides credentials to nodes needing them.  
  - Potential failures: Code errors, missing credentials.  

- **Configure Workflow Args (Custom Credentials)**  
  - Type: Global Constants  
  - Config: Stores custom credential variables for external integrations.  
  - Input: Custom Credentials node  
  - Output: Credential data.  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Config: Allows manual triggering of the workflow from the n8n UI.  
  - Input: None  
  - Output: Starts workflow process.  

---

#### 2.2 News & Video Content Collection

**Overview:**  
This block collects news and video items from multiple sources including custom RSS feeds, SerpAPI searches, and video RSS/XML feeds. It supports nested workflows for content retrieval where needed.

**Nodes Involved:**  
- Custom RSS Feed List (Code)  
- Call Loop Over Custom RSS Feed News List (Execute Workflow)  
- Custom Feed (RSS Feed Read)  
- Call Get Webpage Content to Data (Execute Workflow)  
- Remove Elements with Empty Content  
- Google_news search (SerpAPI)  
- Call Search Queries with SerpAPI (Execute Workflow)  
- Split Out News Results  
- Remove Elements with Empty Links  
- Filter Fields for Loop (Set)  
- Loop Over Items (Search)  
- Loop Over Items  
- Custom Video Feed List (Code)  
- HTTP Request to XML Feed  
- XML  
- Split Out Videos  
- Edit Video Fields  
- Video Filter by Datetime  
- Remove Video Info with Empty Content  
- Process Video Feed Searches (Set)  
- Call Video News - Get the Descriptions of Video Found through Feeds and Theme Search (Execute Workflow)  

**Node Details:**

- **Custom RSS Feed List**  
  - Type: Code  
  - Config: Defines a list of user-customized RSS feeds; indicates if sub-workflows are needed for content retrieval.  
  - Output: List of feeds to process.  

- **Call Loop Over Custom RSS Feed News List**  
  - Type: Execute Workflow  
  - Config: Calls a sub-workflow that iterates over custom RSS feeds to fetch news.  
  - Input: Custom RSS Feed List  
  - Output: Collected news items.  

- **Custom Feed**  
  - Type: RSS Feed Read  
  - Config: Reads RSS feed data for custom feeds; configured to always output data (even on errors).  
  - Output: RSS feed items.  
  - Failures: Feed unavailable, parsing errors.  

- **Call Get Webpage Content to Data**  
  - Type: Execute Workflow  
  - Config: Calls sub-workflow to scrape or fetch full webpage content for news items.  
  - Input: URLs from feeds or searches.  
  - Output: Enhanced content data.  

- **Remove Elements with Empty Content**  
  - Type: Filter  
  - Config: Filters out items with missing or empty content fields to ensure relevance.  

- **Google_news search**  
  - Type: SerpAPI  
  - Config: Uses SerpAPI to perform Google News queries.  
  - Output: News search results.  
  - Potential failures: API key limits, network issues.  

- **Call Search Queries with SerpAPI**  
  - Type: Execute Workflow  
  - Config: Calls sub-workflow to handle multiple SerpAPI news searches iteratively.  

- **Split Out News Results**  
  - Type: Split Out  
  - Config: Splits array of news results to individual items for processing.  

- **Remove Elements with Empty Links**  
  - Type: Filter  
  - Config: Removes news items missing URLs. Common issue with SerpAPI partial results.  

- **Filter Fields for Loop**  
  - Type: Set  
  - Config: Sets or filters fields to prepare items for looping or further processing.  

- **Loop Over Items (Search)**  
  - Type: Split In Batches  
  - Config: Processes items in batches to avoid overload or API limits.  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Config: General batch processing node for iterating over feed items.  

- **Custom Video Feed List**  
  - Type: Code  
  - Config: Lists custom video RSS/XML feeds; indicates if content search is needed.  

- **HTTP Request to XML Feed**  
  - Type: HTTP Request  
  - Config: Fetches XML feeds for videos.  

- **XML**  
  - Type: XML  
  - Config: Parses XML feed data into structured format.  

- **Split Out Videos**  
  - Type: Split Out  
  - Config: Splits video feed items to individual records.  

- **Edit Video Fields**  
  - Type: Set  
  - Config: Adjusts or adds fields relevant to video data.  

- **Video Filter by Datetime**  
  - Type: Filter  
  - Config: Filters videos based on publication datetime to keep recent content.  

- **Remove Video Info with Empty Content**  
  - Type: Filter  
  - Config: Removes video entries missing essential content.  

- **Process Video Feed Searches**  
  - Type: Set  
  - Config: Prepares data for video feed processing or search.  

- **Call Video News - Get the Descriptions of Video Found through Feeds and Theme Search**  
  - Type: Execute Workflow  
  - Config: Calls sub-workflow to enrich video news with descriptions and metadata.  

---

#### 2.3 Content Filtering and Deduplication

**Overview:**  
This block filters articles by datetime, removes duplicates and empty content, and shuffles the news items to maintain freshness and relevance.

**Nodes Involved:**  
- Remove Empty  
- Remove Certain Content  
- Filter by Datetime  
- Remove Duplicates  
- Shuffle News  
- Limit Number of Articles  

**Node Details:**

- **Remove Empty**  
  - Type: Filter  
  - Config: Removes news items with empty key fields (e.g., title, content).  

- **Remove Certain Content**  
  - Type: Filter  
  - Config: Filters out unwanted or blacklisted content based on rules or keywords.  

- **Filter by Datetime**  
  - Type: Filter  
  - Config: Only keeps articles published within a specific timeframe (e.g., last 24 hours).  

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Config: Eliminates duplicate news items based on unique attributes such as link or title.  

- **Shuffle News**  
  - Type: Sort  
  - Config: Randomizes news order to avoid bias in presentation.  

- **Limit Number of Articles**  
  - Type: Limit  
  - Config: Restricts total articles processed downstream to a configured maximum.  

---

#### 2.4 AI Analysis and Quality Assurance

**Overview:**  
This block applies AI-powered analysis, quality checks, summarization, and structured parsing of news content using Mistral Cloud Chat Model and LangChain agents.

**Nodes Involved:**  
- Aggregate  
- News Analyzer (LangChain agent)  
- JSON Output Parser  
- If Not Empty  
- Join Outputs  
- Website Summarizer (LangChain agent)  
- JSON Output Parser 2  
- Quality Assurance (LangChain agent)  
- JSON Output Parser (QA)  
- Aggregate for QA  
- Loop for QA  
- Split Out QA Results  
- Prep QA Results  
- If QA isn't Needed  
- If1 (conditional)  

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Config: Combines multiple items into a batch for AI analysis.  

- **News Analyzer**  
  - Type: LangChain Agent  
  - Config: Uses Mistral AI to analyze news items, extracting insights, relevance, or metadata.  
  - Error Handling: Retries up to 4 times; continues on error.  

- **JSON Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Config: Parses structured JSON output from AI analysis for downstream use.  

- **If Not Empty**  
  - Type: If  
  - Config: Checks if parsed JSON is valid and not empty to prevent workflow failure.  

- **Join Outputs**  
  - Type: Merge  
  - Config: Joins processed news items after AI analysis for further summarization.  

- **Website Summarizer**  
  - Type: LangChain Agent  
  - Config: Produces summaries of news websites or aggregated content using AI.  
  - Retries twice; continues on error.  

- **JSON Output Parser 2**  
  - Type: LangChain Output Parser Structured  
  - Config: Parses JSON from Website Summarizer agent.  

- **Quality Assurance**  
  - Type: LangChain Agent  
  - Config: Performs QA on analyzed content to improve quality and correctness.  

- **JSON Output Parser (QA)**  
  - Type: LangChain Output Parser Structured  
  - Config: Parses QA agent output.  

- **Aggregate for QA**  
  - Type: Aggregate  
  - Config: Groups items for QA processing.  

- **Loop for QA**  
  - Type: Split In Batches  
  - Config: Processes QA items in batches.  

- **Split Out QA Results**  
  - Type: Split Out  
  - Config: Splits QA results for further processing.  

- **Prep QA Results**  
  - Type: Set  
  - Config: Prepares QA results for aggregation, removes pubDate field post-filtering.  

- **If QA isn't Needed**  
  - Type: If  
  - Config: Determines whether QA step should be skipped or executed.  

- **If1**  
  - Type: If  
  - Config: Checks additional conditionals mainly before HTTP Request for content fetching.  

---

#### 2.5 Content Aggregation and Post-Processing

**Overview:**  
Aggregates AI results, converts content into markdown and text, detects language and tone, and applies translation if needed.

**Nodes Involved:**  
- Combine General Summary with Specific Summaries (Merge)  
- Results to Text (Code)  
- Final Markdown Conversion (Markdown)  
- Determine Language & Tone (Code)  
- If Not English (If)  
- If Tone isn't Empty (If)  
- Content Translator (LangChain Agent)  
- Final Aggregate (Aggregate)  
- Final Edit Fields (Set)  

**Node Details:**

- **Combine General Summary with Specific Summaries**  
  - Type: Merge  
  - Config: Combines broad summary with specific article summaries into one dataset.  

- **Results to Text**  
  - Type: Code  
  - Config: Converts aggregated results into plain text format for output or delivery.  

- **Final Markdown Conversion**  
  - Type: Markdown  
  - Config: Converts text content into markdown format for richer presentation.  

- **Determine Language & Tone**  
  - Type: Code  
  - Config: Analyzes content for language detection and tone assessment to guide translation and formatting.  

- **If Not English**  
  - Type: If  
  - Config: Branches workflow to decide if content translation is needed based on detected language.  

- **If Tone isn't Empty**  
  - Type: If  
  - Config: Checks if a tone was detected to decide on further content processing or translation.  

- **Content Translator**  
  - Type: LangChain Agent  
  - Config: Uses AI to translate content to target language or adjust tone if needed.  

- **Final Aggregate**  
  - Type: Aggregate  
  - Config: Final aggregation before sending content out or final edits.  

- **Final Edit Fields**  
  - Type: Set  
  - Config: Final adjustments to fields before output delivery, e.g., formatting or adding metadata.  

---

#### 2.6 Output & Distribution

**Overview:**  
Handles routing of curated news content to various output channels, including email, Telegram, WhatsApp, webhook responses, and file writing for archiving.

**Nodes Involved:**  
- Sending Step Prep (Set)  
- Switch  
- Send email (Email Send)  
- Telegram  
- WhatsApp message (Disabled)  
- Respond to Webhook: JSON with Result (Disabled)  
- Create a binary data for the Latest News (Function)  
- Write contents to disk (Read/Write File)  
- Final Join Outputs (Merge)  

**Node Details:**

- **Sending Step Prep**  
  - Type: Set  
  - Config: Prepares message body, subject, attachments, and other output parameters.  

- **Switch**  
  - Type: Switch  
  - Config: Routes content to one of several output methods based on configuration or user choice.  

- **Send email**  
  - Type: Email Send  
  - Config: Sends the curated news content via email; supports OAuth2 or other credentials.  

- **Telegram**  
  - Type: Telegram  
  - Config: Sends curated content as Telegram messages; requires Telegram bot credentials.  

- **WhatsApp message** (Disabled)  
  - Type: WhatsApp  
  - Config: Disabled node for sending messages via WhatsApp API.  

- **Respond to Webhook: JSON with Result** (Disabled)  
  - Type: Respond to Webhook  
  - Config: Disabled response node for returning JSON results to webhook callers.  

- **Create a binary data for the Latest News**  
  - Type: Function  
  - Config: Converts latest news content into binary data for file writing or further processing.  

- **Write contents to disk**  
  - Type: Read/Write File  
  - Config: Saves curated news content as files on disk for archival or further use.  

- **Final Join Outputs**  
  - Type: Merge  
  - Config: Joins final content streams before output distribution.  

---

#### 2.7 Utility and Control Nodes

**Overview:**  
Nodes that provide flow control, error handling, and conditional execution.

**Nodes Involved:**  
- If (multiple)  
- No Operation, do nothing (NoOp)  
- Stop and Error  
- Confirmation (Set)  
- Filter Fields for Custom Feed (Set)  
- Filter Fields for Loop (Set)  
- Remove Elements with Empty Content (Custom Feed)  
- Remove Elements with Empty Content  
- Remove Duplicates  
- Remove Elements with Empty Links  
- Remove Empty  
- Remove Certain Content  
- Shuffle News  
- Limit Number of Articles  
- Loop Over Items  
- Loop Over Items (Search)  
- Split Out  
- Split Out News Data  
- Split Out QA Results  
- Split Out Videos  
- Aggregate  
- Aggregate for QA  
- Join Outputs  
- Final Aggregate  
- Final Join Outputs  

**Node Details:**  
These nodes manage iteration, batching, filtering, branching, and error flow. For example, `If` nodes gate workflow paths based on conditions such as content presence or language; `Stop and Error` halts the workflow with error signaling; `No Operation` allows graceful handling when video search is disabled.

---

### 3. Summary Table

| Node Name                                    | Node Type                           | Functional Role                                     | Input Node(s)                                        | Output Node(s)                                       | Sticky Note                                                                 |
|----------------------------------------------|-----------------------------------|----------------------------------------------------|-----------------------------------------------------|-----------------------------------------------------|-----------------------------------------------------------------------------|
| Schedule                                     | Schedule Trigger                  | Workflow automatic trigger                          | None                                                | Configure Workflow Args                              |                                                                             |
| Email Trigger (IMAP)                         | Email Read IMAP                  | Email-based trigger                                | None                                                | Configure Workflow Args                              |                                                                             |
| Webhook: Call Workflow Activation            | Webhook                         | External HTTP trigger (disabled)                   | None                                                | Configure Workflow Args                              | To execute a POST request using cURL...                                     |
| Configure Workflow Args                      | Global Constants                | Sets global workflow variables                      | Schedule, Email Trigger, Webhook                     | If Video Search is Enabled, Call Search Queries...  | Useful for quick variable setting and template sharing                      |
| Configure Workflow Args (Alternative)       | Set                             | Alternative global variables setup                  | Schedule                                            | If Video Search is Enabled                           |                                                                             |
| Custom Credentials                           | Code                            | Manages custom credentials                          | Configure Workflow Args                             | Configure Workflow Args (Custom Credentials)         |                                                                             |
| Configure Workflow Args (Custom Credentials)| Global Constants                | Custom credential variables                          | Custom Credentials                                  |                                                     |                                                                             |
| When clicking ‘Execute workflow’             | Manual Trigger                  | Manual workflow start                               | None                                                | Configure Workflow Args                              |                                                                             |
| Custom RSS Feed List                         | Code                            | Defines custom RSS feeds list                        | Configure Workflow Args                             | Call Loop Over Custom RSS Feed News List             |                                                                             |
| Call Loop Over Custom RSS Feed News List     | Execute Workflow                | Calls sub-workflow to process custom RSS feeds      | Custom RSS Feed List                                | Merge                                               |                                                                             |
| Custom Feed                                 | RSS Feed Read                   | Reads custom RSS feed                               | Loop Over Items                                    | Filter Fields for Custom Feed                        |                                                                             |
| Call Get Webpage Content to Data             | Execute Workflow                | Retrieves full webpage content                       | If, Call Search Queries with SerpAPI                | Remove Elements with Empty Content                   |                                                                             |
| Remove Elements with Empty Content           | Filter                         | Removes items with empty content                     | Call Get Webpage Content to Data                    | Merge                                               |                                                                             |
| Google_news search                          | SerpAPI                        | Performs Google News search                          | Loop Over Items (Search)                            | Split Out News Results                              |                                                                             |
| Call Search Queries with SerpAPI             | Execute Workflow                | Executes batch news searches via SerpAPI            | Configure Workflow Args                             | Call Get Webpage Content to Data                     |                                                                             |
| Split Out News Results                       | Split Out                      | Splits news results array into items                | Google_news search                                 | Remove Elements with Empty Links                     |                                                                             |
| Remove Elements with Empty Links             | Filter                         | Removes news without valid URLs                      | Split Out News Results                              | Filter Fields for Loop                              | Note: SerpAPI returns empty links sometimes                                |
| Filter Fields for Loop                       | Set                            | Prepares fields for looping                           | Remove Elements with Empty Links                    | Loop Over Items (Search)                            |                                                                             |
| Loop Over Items (Search)                      | Split In Batches               | Processes search results in batches                   | Filter Fields for Loop                              | Google_news search                                  |                                                                             |
| Loop Over Items                              | Split In Batches               | Batch processing for feed items                       | Remove Elements with Empty Content (Custom Feed)   | Custom Feed                                        |                                                                             |
| Custom Video Feed List                       | Code                           | Lists custom video RSS/XML feeds                      | Configure Workflow Args                             | HTTP Request to XML Feed                            |                                                                             |
| HTTP Request to XML Feed                     | HTTP Request                  | Fetches video XML feeds                               | Custom Video Feed List                             | XML                                                |                                                                             |
| XML                                         | XML                            | Parses XML video feed                                 | HTTP Request to XML Feed                           | Split Out Videos                                   |                                                                             |
| Split Out Videos                            | Split Out                     | Splits video feed array into items                    | XML                                                | Edit Video Fields                                  |                                                                             |
| Edit Video Fields                           | Set                            | Edits video item fields                                | Split Out Videos                                   | Video Filter by Datetime                           |                                                                             |
| Video Filter by Datetime                    | Filter                         | Filters videos by date                                 | Edit Video Fields                                  | Remove Video Info with Empty Content               |                                                                             |
| Remove Video Info with Empty Content         | Filter                         | Filters out videos missing content                     | Video Filter by Datetime                           | Confirmation                                       |                                                                             |
| Process Video Feed Searches                  | Set                            | Prepares video feed search data                        | Custom Video Feed List                             | Custom Video Feed List                             |                                                                             |
| Call Video News - Get the Descriptions...   | Execute Workflow               | Enriches video news with descriptions                  | If Video Search is Enabled                         | Merge                                             |                                                                             |
| Remove Empty                                | Filter                         | Removes news items with empty fields                   | Merge                                             | Remove Duplicates                                 |                                                                             |
| Remove Certain Content                      | Filter                         | Filters out unwanted content                            | Filter by Datetime                                | Shuffle News                                      |                                                                             |
| Filter by Datetime                          | Filter                         | Filters articles by publication date                   | Remove Duplicates                                 | Remove Certain Content                            |                                                                             |
| Remove Duplicates                           | Remove Duplicates             | Removes duplicate news items                            | Remove Empty                                      | Filter by Datetime                                |                                                                             |
| Shuffle News                               | Sort                           | Randomizes news order                                   | Remove Certain Content                            | Limit Number of Articles                         |                                                                             |
| Limit Number of Articles                    | Limit                          | Limits total articles                                  | Shuffle News                                      | Join Outputs, If QA isn't Needed                 |                                                                             |
| Aggregate                                  | Aggregate                     | Groups items for AI analysis                            | Prep QA Results, If QA isn't Needed, Limit Number of Articles | News Analyzer, Quality Assurance, Website Summarizer |                                                                             |
| News Analyzer                              | LangChain Agent              | AI analysis of news content                             | Aggregate                                         | If Not Empty                                     | Retries 4 times; continues on error                                       |
| JSON Output Parser                         | LangChain Output Parser       | Parses AI JSON output                                   | News Analyzer                                     | News Analyzer                                    |                                                                             |
| If Not Empty                              | If                             | Checks if AI output is valid                            | News Analyzer                                     | Split Out News Data, Combine General Summary... |                                                                             |
| Split Out News Data                        | Split Out                     | Splits news data array                                  | If Not Empty                                      | Join Outputs                                     |                                                                             |
| Combine General Summary with Specific Summaries | Merge                       | Combines general and article summaries                  | Split Out News Data                               | Results to Text                                  |                                                                             |
| Results to Text                           | Code                           | Converts summaries to text                              | Combine General Summary                           | Final Markdown Conversion, Determine Language & Tone |                                                                             |
| Final Markdown Conversion                  | Markdown                     | Converts text to markdown format                         | Results to Text                                   | Final Join Outputs                              |                                                                             |
| Determine Language & Tone                   | Code                          | Detects language and tone                                | Results to Text                                   | Final Join Outputs                              |                                                                             |
| Final Join Outputs                        | Merge                         | Joins final processed content                            | Final Markdown Conversion, Determine Language & Tone | If Not English                                  |                                                                             |
| If Not English                            | If                            | Determines if content translation is needed             | Final Join Outputs                               | If Tone isn't Empty, Content Translator          |                                                                             |
| If Tone isn't Empty                       | If                            | Checks for tone presence                                 | If Not English                                  | Final Edit Fields, Content Translator             |                                                                             |
| Content Translator                        | LangChain Agent              | Translates or tone-adjusts content                       | If Not English                                  | Sending Step Prep                                |                                                                             |
| Final Edit Fields                        | Set                           | Final field edits before output                          | If Tone isn't Empty                             | Sending Step Prep                                |                                                                             |
| Sending Step Prep                        | Set                           | Prepares content for delivery                            | Final Edit Fields, Content Translator            | Switch                                           |                                                                             |
| Switch                                   | Switch                        | Routes content to output channels                         | Sending Step Prep                                | Send email, Telegram, WhatsApp message, Respond... |                                                                             |
| Send email                              | Email Send                   | Sends curated news by email                               | Switch                                           | None                                             |                                                                             |
| Telegram                                | Telegram                     | Sends curated news via Telegram                           | Switch                                           | None                                             |                                                                             |
| WhatsApp message                        | WhatsApp                     | Sends curated news via WhatsApp (disabled)               | Switch                                           | None                                             |                                                                             |
| Respond to Webhook: JSON with Result    | Respond to Webhook           | Responds to webhook with JSON (disabled)                  | Switch                                           | None                                             |                                                                             |
| Create a binary data for the Latest News | Function                     | Creates binary content for file writing                    | Switch                                           | Write contents to disk                           |                                                                             |
| Write contents to disk                   | Read/Write File              | Saves content to disk                                     | Create a binary data for the Latest News         | None                                             |                                                                             |
| Confirmation                           | Set                           | Confirmation step linking to merge                        | No Operation, do nothing                         | Merge                                            |                                                                             |
| No Operation, do nothing                | NoOp                         | Placeholder for disabled video search branch              | If Video Search is Enabled                       | Confirmation                                     |                                                                             |
| Stop and Error                         | Stop and Error               | Stops workflow on error                                    | If Not Empty (false branch)                      | None                                             |                                                                             |
| Various Sticky Notes                   | Sticky Note                  | Comments and notes across the workflow                    | —                                                 | —                                                | See sticky notes section below for details                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Set Up Triggers & Global Variables:**  
   - Create a **Schedule Trigger** node for automatic runs.  
   - Add an **Email Trigger (IMAP)** node for email-based activation.  
   - Optionally add a **Webhook** trigger node (disabled by default).  
   - Create a **Global Constants** node named "Configure Workflow Args" to define global variables for feeds, API keys, time filters, and output settings.  
   - Add a **Set** node as alternative global variables setup if global constants are not available.  
   - Create a **Code** node to manage custom credentials, then a **Global Constants** node to expose these credentials downstream.  
   - Add a **Manual Trigger** node for manual execution.

2. **News & Video Feed Collection:**  
   - Add a **Code** node ("Custom RSS Feed List") to provide user-defined RSS feed URLs and flags for sub-workflow needs.  
   - Create an **Execute Workflow** node calling a sub-workflow that loops over these feeds ("Call Loop Over Custom RSS Feed News List").  
   - Add an **RSS Feed Read** node ("Custom Feed") that reads each RSS feed with error handling enabled.  
   - Create another **Execute Workflow** node ("Call Get Webpage Content to Data") to fetch full article content from URLs.  
   - Insert a **Filter** node to remove items with empty content.  
   - Add a **SerpAPI** node ("Google_news search") for Google News search queries.  
   - Use an **Execute Workflow** node to loop and handle SerpAPI results ("Call Search Queries with SerpAPI").  
   - Add **Split Out** nodes to process arrays into individual items.  
   - Insert **Filter** nodes to remove news without valid URLs or empty fields.  
   - Add **Set** nodes to prepare data fields for looping.  
   - Use **Split In Batches** nodes to process items in chunks and avoid API rate limits.  
   - For videos, add a **Code** node ("Custom Video Feed List") listing video RSS/XML feeds.  
   - Create an **HTTP Request** node to fetch XML feeds, followed by an **XML** node to parse them.  
   - Use **Split Out**, **Set**, and **Filter** nodes to process video items, filter by date, and remove empty content.  
   - Add an **Execute Workflow** node to enrich video news descriptions.  
   - Include error handling and conditional branches to skip video processing if disabled.

3. **Filtering and Deduplication:**  
   - Add multiple **Filter** nodes to remove empty or undesired content.  
   - Use a **Remove Duplicates** node to avoid repeated articles.  
   - Add a **Sort** node ("Shuffle News") to randomize item order.  
   - Use a **Limit** node to restrict the number of articles processed downstream.

4. **AI Analysis & QA:**  
   - Aggregate news items using **Aggregate** nodes.  
   - Add **LangChain Agent** nodes for "News Analyzer", "Website Summarizer", and "Quality Assurance" using Mistral AI. Configure retry logic and error handling.  
   - Add **LangChain Output Parser Structured** nodes to parse JSON AI responses.  
   - Use **If** nodes to conditionally handle empty or failed outputs.  
   - Use **Split Out** and **Split In Batches** nodes to process QA results.  
   - Prepare QA results with **Set** nodes and aggregate again as needed.  
   - Use conditional branching to skip QA if not needed.

5. **Aggregation & Post-Processing:**  
   - Merge general summaries and specific article summaries with **Merge** nodes.  
   - Convert results to text via **Code** nodes, then to markdown with **Markdown** nodes.  
   - Detect language and tone using **Code** nodes.  
   - Add conditional **If** nodes to check if translation is needed.  
   - Use a **LangChain Agent** node for content translation.  
   - Finalize content with **Set** nodes for any last adjustments.  
   - Aggregate final outputs.

6. **Output and Distribution:**  
   - Prepare sending parameters in a **Set** node.  
   - Use a **Switch** node to route output to chosen channels:  
     - **Email Send** node for email delivery (configure credentials and parameters).  
     - **Telegram** node for Telegram messages (configure bot credentials).  
     - **WhatsApp** node (disabled by default).  
     - **Respond to Webhook** node (disabled by default).  
     - **Function** node to create binary data for saving.  
     - **Read/Write File** node to save content locally.  

7. **Utilities & Controls:**  
   - Add **If**, **NoOp**, **Stop and Error**, and **Set** nodes throughout for flow control, error handling, and confirmation steps.  
   - Configure retries and error continuation where appropriate to prevent workflow interruption.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                |
|------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| "To execute a POST request using cURL, you can use: curl -X POST [URL] -d ..." | Webhook: Call Workflow Activation node notes for external triggering examples. |
| "If you would like, you can provide a list of custom RSS feeds. Please indicate if they will require a sub-workflow to retrieve content as well." | Custom RSS Feed List node description for user customization.                 |
| "This node is used to test new RSS Feeds before adding them."                | Custom Feed Test node notes for RSS feed validation before deployment.        |
| "- Hint - Use: sudo find / -name \"filename\" If you cannot find the file ..." | Write contents to disk node notes for troubleshooting file location issues.   |
| "It is necessary because the JSON parser can fail while still returning a success status for some reason." | If Not Empty node notes explaining error handling rationale.                  |
| "It is a necessary node because sites like lobste.rs do not provide the actual content of the links." | HTTP Request node notes describing content retrieval necessity.               |

---

**Disclaimer:** The text above is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.