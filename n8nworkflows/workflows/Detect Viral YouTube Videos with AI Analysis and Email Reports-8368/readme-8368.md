Detect Viral YouTube Videos with AI Analysis and Email Reports

https://n8nworkflows.xyz/workflows/detect-viral-youtube-videos-with-ai-analysis-and-email-reports-8368


# Detect Viral YouTube Videos with AI Analysis and Email Reports

### 1. Workflow Overview

This workflow automates the discovery, analysis, and reporting of potentially viral YouTube videos based on a user-defined keyword or niche. It leverages YouTube Data API to search recent videos, collects detailed video and channel statistics, computes a custom "Algorithmic Lift Score" to estimate viral potential, and then uses an AI agent to generate an insightful summary report. Finally, the report is emailed to a designated recipient.

Logical blocks grouped by functionality and dependencies:

- **1.1 Input Reception & Initialization**  
  Receives manual or scheduled triggers and sets up initial parameters such as search query, API keys, timeframe, and report recipient.

- **1.2 YouTube Data Retrieval**  
  Searches for videos with the specified query and timeframe, then fetches detailed video and channel statistics.

- **1.3 Data Processing & Metric Calculation**  
  Processes raw data, calculates viral metrics including views per subscriber, engagement rates, and the composite Algorithmic Lift Score.

- **1.4 Data Aggregation & Sorting**  
  Removes duplicates, sorts videos by viral score, limits the selection to top entries, and aggregates data for AI analysis.

- **1.5 AI Analysis & Summary Generation**  
  Passes aggregated data to an AI agent (configured for OpenAI GPT-4o-mini) which produces a concise, insightful summary of viral trends and key findings.

- **1.6 Email Report Dispatch**  
  Converts AI output from markdown to HTML and emails the report to the configured recipient.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Initialization

**Overview:**  
Initiates workflow execution either manually or by schedule, and configures key parameters controlling the search and reporting behavior.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  
- Setup

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows user to start workflow on demand  
  - Config: Default (no parameters)  
  - Inputs: None  
  - Outputs: Connects to Setup node  
  - Failures: User must trigger manually to test  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow daily at 13:00 (1 PM)  
  - Config: Set to trigger at hour 13 every day  
  - Inputs: None  
  - Outputs: Connects to Setup node  
  - Failures: None expected, but workflow won’t run if node is disabled or time misconfigured  

- **Setup**  
  - Type: Set node  
  - Role: Defines workflow parameters such as search keyword, API key, days to look back, max results, and recipient email  
  - Configurations:  
    - `query`: Search keyword (default "AIvideo")  
    - `GoogleAPIkey`: API key for YouTube Data API v3 (empty by default, must be set)  
    - `daysback`: Number of days back to search (default 3)  
    - `maxResult`: Number of videos to retrieve (default 20, recommended max 50 due to rate limits)  
    - `email`: Recipient email address for report (empty by default)  
  - Inputs: Receives trigger from manual or schedule trigger  
  - Outputs: Connects to SearchVideos node  
  - Edge Cases: Missing API key or email will cause downstream failures or no report sent  

---

#### 1.2 YouTube Data Retrieval

**Overview:**  
Searches YouTube for videos matching the query, then extracts detailed video and channel statistics via multiple API calls.

**Nodes Involved:**  
- SearchVideos  
- Split Out1  
- Split Out2  
- GetChannelInfo  
- GetVidStats  
- ChannelInfo  
- VidStats  
- Merge

**Node Details:**

- **SearchVideos**  
  - Type: HTTP Request  
  - Role: Calls YouTube Search API to find videos based on query, date range, and order by view count  
  - Key Config:  
    - Endpoint: `youtube/v3/search`  
    - Query parameters: API key, `part=snippet`, order by viewCount, type=video, publishedAfter calculated from `daysback`, maxResults from setup  
  - Inputs: Setup node output  
  - Outputs: Connects to Split Out1  
  - Failures: API quota exceeded, invalid API key, network issues  

- **Split Out1**  
  - Type: Split Out  
  - Role: Extracts specific fields (`videoId`, `channelId`, `publishedAt`) from search results for processing  
  - Inputs: SearchVideos output  
  - Outputs: Connects to Split Out2  

- **Split Out2**  
  - Type: Split Out  
  - Role: Further splits channelId and videoId fields for separate API calls  
  - Inputs: Split Out1 output  
  - Outputs: Connects to GetChannelInfo and GetVidStats nodes  

- **GetChannelInfo**  
  - Type: HTTP Request  
  - Role: Fetches detailed channel data including snippet, statistics, and topic details using channelId  
  - Config: Calls YouTube Channels API with parts `snippet,statistics,topicDetails`  
  - Inputs: Split Out2 data  
  - Outputs: Connects to ChannelInfo code node  
  - Failures: Invalid channelId, API quota exceeded, malformed requests  

- **GetVidStats**  
  - Type: HTTP Request  
  - Role: Fetches detailed video statistics and snippet data for each videoId  
  - Config: Calls YouTube Videos API with parts `snippet,statistics`  
  - Inputs: Split Out2 data  
  - Outputs: Connects to VidStats code node  
  - Failures: Invalid videoId, API quota exceeded  

- **ChannelInfo**  
  - Type: Code  
  - Role: Extracts and formats channel information from API response for later merging  
  - Logic: Parses first channel item, retrieves name, ID, open date, total views, subscriber count  
  - Inputs: GetChannelInfo output  
  - Outputs: Connects to Merge node as input #0  

- **VidStats**  
  - Type: Code  
  - Role: Extracts and formats video data from API response, adds human-readable category names  
  - Logic: Maps YouTube category IDs to names, extracts stats like viewCount, likeCount, commentCount, publishedAt, videoId  
  - Inputs: GetVidStats output  
  - Outputs: Connects to Merge node as input #1  

- **Merge**  
  - Type: Merge  
  - Role: Combines channel info and video stats on matching `channelID`  
  - Config: Mode: "combine" with field matching on `channelID`  
  - Inputs: ChannelInfo and VidStats outputs  
  - Outputs: Connects to Remove_Duplicates node  
  - Failures: Mismatch in keys may cause incomplete merges  

---

#### 1.3 Data Processing & Metric Calculation

**Overview:**  
Removes duplicate videos, calculates viral metrics including views per subscriber, engagement rates, and a composite Algorithmic Lift Score for ranking.

**Nodes Involved:**  
- Remove_Duplicates  
- CalculateMetrics  
- AlgorithmicLiftScore

**Node Details:**

- **Remove_Duplicates**  
  - Type: Remove Duplicates  
  - Role: Ensures no duplicate video entries based on default criteria  
  - Inputs: Merge output  
  - Outputs: CalculateMetrics node  
  - Edge Cases: Overly aggressive deduplication might remove distinct videos if fields overlap  

- **CalculateMetrics**  
  - Type: Code  
  - Role: Calculates base metrics per video: views, likes, comments, published date, like rate, comment rate, views per hour, plus passes through channel and category info  
  - Logic highlights:  
    - Converts counts to numbers with fallbacks  
    - Calculates hours since published with date parsing  
    - Computes like and comment rates as percentages of views  
    - Calculates views per hour since published  
  - Inputs: Remove_Duplicates output  
  - Outputs: AlgorithmicLiftScore node  
  - Edge Cases: Invalid or missing publishedAt dates handled by fallback to 0 or defaults  

- **AlgorithmicLiftScore**  
  - Type: Code  
  - Role: Computes composite viral score per video based on:  
    - Views per subscriber (weighted 60%)  
    - Relative engagement rate (likes + comments per view, weighted 30%)  
    - Logarithm of views per hour (weighted 10%)  
  - Logic highlights:  
    - Handles zero or missing subscriber counts gracefully  
    - Enforces minimum 1 hour since publish to avoid division by zero  
    - Outputs calculated metrics alongside original data  
  - Inputs: CalculateMetrics output  
  - Outputs: Sort node  
  - Edge Cases: Invalid dates, zero subscribers guarded against; division by zero avoided  

---

#### 1.4 Data Aggregation & Sorting

**Overview:**  
Sorts videos by the calculated Algorithmic Lift Score, limits selection to top 5, aggregates data for AI processing.

**Nodes Involved:**  
- Sort  
- Limit  
- Aggregate

**Node Details:**

- **Sort**  
  - Type: Sort  
  - Role: Orders videos descending by `algorithmicLiftScore`  
  - Inputs: AlgorithmicLiftScore output  
  - Outputs: Limit node  
  - Edge Cases: Missing or NaN scores could cause unexpected ordering  

- **Limit**  
  - Type: Limit  
  - Role: Restricts results to top 5 videos for analysis and reporting  
  - Inputs: Sort output  
  - Outputs: Aggregate node  
  - Edge Cases: If fewer than 5 videos available, passes all  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all top video items into a single JSON array for AI input  
  - Inputs: Limit output  
  - Outputs: AI Agent node  
  - Edge Cases: Empty input results in empty aggregation, potentially causing AI node failure  

---

#### 1.5 AI Analysis & Summary Generation

**Overview:**  
Uses an AI agent to analyze aggregated video data and generate a concise report highlighting viral trends, top videos, key statistics, and notable observations.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Think  
- Markdown

**Node Details:**

- **AI Agent**  
  - Type: Langchain Agent Node  
  - Role: Sends JSON array of video data to AI for analysis using a defined system prompt and instructions  
  - Configuration:  
    - Input text set as JSON string from Aggregate node output  
    - System message instructs AI to analyze 20 videos with metrics like Algorithmic Lift Score, views per subscriber, engagement rate  
    - Output requirements specify summary structure including top viral videos and statistical insights, max 300 words  
  - Inputs: Aggregate node output (JSON array)  
  - Outputs: Markdown node  
  - Edge Cases: Large or malformed JSON may cause AI errors; AI model availability (OpenAI quota, connectivity) impact  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Underlying language model used by AI Agent (GPT-4o-mini)  
  - Credentials: Requires valid OpenAI API key  
  - Inputs: From AI Agent (ai_languageModel input)  
  - Outputs: AI Agent (ai_tool output)  
  - Edge Cases: API key invalid or quota exhausted; network issues  

- **Think**  
  - Type: Langchain Tool Think (AI helper)  
  - Role: Auxiliary AI step to assist reasoning (connected but not detailed in workflow usage)  

- **Markdown**  
  - Type: Markdown Node  
  - Role: Converts AI agent's markdown summary to HTML format for email  
  - Inputs: AI Agent output  
  - Outputs: Send_Report node  

---

#### 1.6 Email Report Dispatch

**Overview:**  
Sends the AI-generated analysis summary as an HTML email to the configured recipient using Gmail.

**Nodes Involved:**  
- Send_Report

**Node Details:**

- **Send_Report**  
  - Type: Gmail Node  
  - Role: Sends email containing the viral video analysis report  
  - Configuration:  
    - Recipient email taken from Setup node parameter  
    - Subject includes dynamic keyword and daysback info  
    - Message body uses AI-generated HTML content from Markdown node  
    - Gmail OAuth2 credentials required (user must authenticate)  
  - Inputs: Markdown output  
  - Outputs: Terminal node (no further output)  
  - Edge Cases: Invalid recipient email, Gmail credential issues, rate limits, permission errors  

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                                   | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                             |
|-----------------------------|--------------------------------------|-------------------------------------------------|--------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                       | Manual start trigger                             | None                           | Setup                        | See Main Overview sticky note for workflow intro                                                      |
| Schedule Trigger             | Schedule Trigger                     | Scheduled daily trigger                          | None                           | Setup                        | See Main Overview sticky note for workflow intro                                                      |
| Setup                       | Set                                 | Defines search parameters and credentials       | When clicking ‘Test workflow’, Schedule Trigger | SearchVideos                 | See Sticky Note1 for detailed setup instructions including API key and email configuration            |
| SearchVideos                | HTTP Request                        | Searches YouTube for videos matching criteria   | Setup                         | Split Out1                   |                                                                                                       |
| Split Out1                 | Split Out                           | Extracts video and channel IDs                   | SearchVideos                  | Split Out2                   |                                                                                                       |
| Split Out2                 | Split Out                           | Further extracts IDs for API requests            | Split Out1                   | GetChannelInfo, GetVidStats  |                                                                                                       |
| GetChannelInfo             | HTTP Request                        | Retrieves channel info via YouTube API           | Split Out2                   | ChannelInfo                  |                                                                                                       |
| GetVidStats                | HTTP Request                        | Retrieves detailed video stats                    | Split Out2                   | VidStats                     |                                                                                                       |
| ChannelInfo                | Code                               | Parses and formats channel data                   | GetChannelInfo               | Merge                       |                                                                                                       |
| VidStats                   | Code                               | Parses and formats video data with category names| GetVidStats                  | Merge                       |                                                                                                       |
| Merge                      | Merge                              | Merges channel and video data on channelID       | ChannelInfo, VidStats        | Remove_Duplicates            |                                                                                                       |
| Remove_Duplicates           | Remove Duplicates                   | Deduplicates merged video data                    | Merge                        | CalculateMetrics             |                                                                                                       |
| CalculateMetrics           | Code                               | Calculates base video metrics                      | Remove_Duplicates            | AlgorithmicLiftScore         |                                                                                                       |
| AlgorithmicLiftScore       | Code                               | Calculates composite viral score and engagement rates | CalculateMetrics            | Sort                        |                                                                                                       |
| Sort                       | Sort                               | Sorts videos descending by Algorithmic Lift Score| AlgorithmicLiftScore         | Limit                       |                                                                                                       |
| Limit                      | Limit                              | Limits to top 5 videos for analysis               | Sort                        | Aggregate                   |                                                                                                       |
| Aggregate                  | Aggregate                         | Aggregates top videos into JSON array for AI      | Limit                       | AI Agent                   |                                                                                                       |
| AI Agent                   | Langchain Agent                   | Generates AI summary report                        | Aggregate                   | Markdown                   | See Sticky Note2 for AI model setup instructions                                                      |
| OpenAI Chat Model          | Langchain OpenAI Chat Model       | OpenAI GPT-4o-mini model used by AI Agent         | AI Agent (ai_languageModel)  | AI Agent (ai_tool)          | See Sticky Note2 for AI model setup instructions                                                      |
| Think                      | Langchain Tool Think              | AI reasoning helper (internal use)                | AI Agent (ai_tool)           | AI Agent                   |                                                                                                       |
| Markdown                   | Markdown                          | Converts AI markdown to HTML                        | AI Agent                    | Send_Report                 |                                                                                                       |
| Send_Report                | Gmail                            | Sends email report with analysis                   | Markdown                    | None                       | See Sticky Note3 for Gmail setup and sending instructions                                            |
| Sticky Note                | Sticky Note                      | Provides main workflow overview                     | None                       | None                       | Contains main overview and workflow introduction                                                      |
| Sticky Note1               | Sticky Note                      | Setup node configuration instructions              | None                       | None                       | Explains search parameters, API key setup, and email configuration                                   |
| Sticky Note2               | Sticky Note                      | AI model connection instructions                    | None                       | None                       | Instructions for configuring OpenAI or alternative AI models                                        |
| Sticky Note3               | Sticky Note                      | Gmail account and email sending setup instructions | None                       | None                       | Details Gmail OAuth2 credential setup and email recipient configuration                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add **Manual Trigger** node named "When clicking ‘Test workflow’". No configuration needed.  
   - Add **Schedule Trigger** node named "Schedule Trigger" set to trigger daily at 13:00 (1 PM).

2. **Add Setup Node:**  
   - Add a **Set** node named "Setup".  
   - Define parameters:  
     - `query` (string): default "AIvideo"  
     - `GoogleAPIkey` (string): leave blank, fill with your YouTube Data API key  
     - `daysback` (number): default 3  
     - `maxResult` (number): default 20 (recommend ≤50)  
     - `email` (string): leave blank, fill with your email address  

3. **Connect Triggers to Setup:**  
   - Connect **When clicking ‘Test workflow’** and **Schedule Trigger** outputs to Setup node input.

4. **Add SearchVideos Node (HTTP Request):**  
   - Name: "SearchVideos"  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Query Parameters:  
     - `key`: `={{ $json.GoogleAPIkey }}` (from Setup)  
     - `part`: `snippet`  
     - `order`: `viewCount`  
     - `type`: `video`  
     - `publishedAfter`: `={{ $now.minus({ days: $json.daysback }).toUTC().toISO() }}`  
     - `q`: `={{ $json.query }}`  
     - `maxResults`: `={{ $json.maxResult }}`  
   - Connect Setup output to SearchVideos input.

5. **Add Split Out Nodes:**  
   - "Split Out1" split on `items` field, include fields: `videoId`, `channelId`, `publishedAt`.  
   - Connect SearchVideos output to Split Out1.  
   - "Split Out2" split on `items.snippet.channelId`, include fields: `items.snippet.channelId`, `items.snippet.publishedAt`, `items.id.videoId`.  
   - Connect Split Out1 output to Split Out2.

6. **Add GetChannelInfo Node (HTTP Request):**  
   - Name: "GetChannelInfo"  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/channels`  
   - Query Parameters:  
     - `key`: `={{ $json.GoogleAPIkey }}`  
     - `part`: `snippet,statistics,topicDetails`  
     - `id`: `={{ $json['items.snippet.channelId'] }}`  
   - Connect Split Out2 output to GetChannelInfo.

7. **Add GetVidStats Node (HTTP Request):**  
   - Name: "GetVidStats"  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/videos`  
   - Query Parameters:  
     - `key`: `={{ $json.GoogleAPIkey }}`  
     - `part`: `snippet,statistics`  
     - `id`: `={{ $json['items.id.videoId'] }}`  
   - Connect Split Out2 output also to GetVidStats.

8. **Add ChannelInfo Node (Code):**  
   - Name: "ChannelInfo"  
   - Code: Parses first item from GetChannelInfo response, extracts channelName, channelID, channelOpenDate, channelTotalViewCount, subscriberCount.  
   - Connect GetChannelInfo output to ChannelInfo.

9. **Add VidStats Node (Code):**  
   - Name: "VidStats"  
   - Code: Parses first item from GetVidStats response, maps category ID to name, extracts video stats and metadata.  
   - Connect GetVidStats output to VidStats.

10. **Add Merge Node:**  
    - Name: "Merge"  
    - Mode: "combine" with field match on `channelID`  
    - Connect ChannelInfo output to Merge input #0.  
    - Connect VidStats output to Merge input #1.

11. **Add Remove_Duplicates Node:**  
    - Name: "Remove_Duplicates"  
    - Connect Merge output to Remove_Duplicates input.

12. **Add CalculateMetrics Node (Code):**  
    - Name: "CalculateMetrics"  
    - Code: Calculates views, likes, comments, likeRate, commentRate, viewsPerHour, and extracts other metadata.  
    - Connect Remove_Duplicates output to CalculateMetrics input.

13. **Add AlgorithmicLiftScore Node (Code):**  
    - Name: "AlgorithmicLiftScore"  
    - Code: Calculates composite viral score using weighted formula of views per subscriber, engagement rate, and views per hour.  
    - Connect CalculateMetrics output to AlgorithmicLiftScore input.

14. **Add Sort Node:**  
    - Name: "Sort"  
    - Sort by `algorithmicLiftScore` descending.  
    - Connect AlgorithmicLiftScore output to Sort input.

15. **Add Limit Node:**  
    - Name: "Limit"  
    - Limit to 5 items.  
    - Connect Sort output to Limit input.

16. **Add Aggregate Node:**  
    - Name: "Aggregate"  
    - Aggregate all item data into a single JSON array.  
    - Connect Limit output to Aggregate input.

17. **Add AI Agent Node:**  
    - Name: "AI Agent"  
    - Parameters:  
      - Text input: `=json: {{$('Aggregate').item.json.data.toJsonString() }}`  
      - System message: Detailed prompt instructing AI to analyze viral video metrics and generate a concise summary as described in the original prompt.  
    - Connect Aggregate output to AI Agent input.

18. **Add OpenAI Chat Model Node:**  
    - Name: "OpenAI Chat Model"  
    - Model: GPT-4o-mini or preferred OpenAI model  
    - Connect AI Agent node’s `ai_languageModel` input to this node.  
    - Add valid OpenAI API credentials.

19. **Add Markdown Node:**  
    - Name: "Markdown"  
    - Mode: Convert markdown to HTML.  
    - Connect AI Agent output to Markdown input.

20. **Add Send_Report Node (Gmail):**  
    - Name: "Send_Report"  
    - Configure Gmail OAuth2 credentials.  
    - Set email recipient using expression `={{$('Setup').item.json.email}}`.  
    - Subject: `Virality analysis of the last {{ $('Setup').item.json.daysback }} days for keyword: "{{ $('Setup').item.json.query }}"`  
    - Message: Use HTML output from Markdown node.  
    - Connect Markdown output to Send_Report input.

21. **Connect all nodes as per the described connections:**

    - Setup → SearchVideos  
    - SearchVideos → Split Out1 → Split Out2 → GetChannelInfo & GetVidStats  
    - GetChannelInfo → ChannelInfo → Merge (#0)  
    - GetVidStats → VidStats → Merge (#1)  
    - Merge → Remove_Duplicates → CalculateMetrics → AlgorithmicLiftScore → Sort → Limit → Aggregate → AI Agent → Markdown → Send_Report  
    - OpenAI Chat Model connected as AI Agent’s language model node input  

22. **Save and Activate:**  
    - Fill in your Google API key and email in Setup node.  
    - Ensure OpenAI credentials are configured.  
    - Ensure Gmail OAuth2 is connected with permission to send emails.  
    - Test with manual trigger or wait for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Main workflow overview explaining the purpose, steps, and author credits.                                                                                                                                                         | Sticky Note "Main Overview" in workflow                                                            |
| Detailed instructions for Setup node configuration, including how to obtain and configure Google API key for YouTube Data API v3.                                                                                              | Sticky Note1; [Google Cloud Console](https://console.cloud.google.com/)                            |
| Instructions for connecting OpenAI API credentials and note on alternative AI providers like Google Generative AI (Gemini).                                                                                                     | Sticky Note2                                                                                      |
| Gmail OAuth2 setup instructions to configure email sending and permissions.                                                                                                                                                      | Sticky Note3                                                                                      |
| YouTube category ID to name mapping is embedded in code node "VidStats" with awareness of possible category changes or overlaps by YouTube.                                                                                    | VidStats node                                                                                      |
| The workflow is designed to respect YouTube API rate limits by default maxResult capped at 20 (recommended max 50). Users should monitor API quotas and adjust as necessary.                                                       | Setup node parameter explanation and Sticky Note1                                                 |
| AI summary output is limited to approximately 300 words with bullet points for readability. AI prompt instructs to highlight key viral metrics and trends.                                                                        | AI Agent system message                                                                           |
| Requires n8n version supporting Langchain integration and nodes: HTTP Request v4.2+, Code v2, Gmail v2.1, and Markdown.                                                                                                         | Workflow node versions                                                                             |
| For best results, ensure all credentials (Google API, OpenAI, Gmail OAuth2) are valid and have sufficient quota and permissions.                                                                                                | General recommendation                                                                            |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.