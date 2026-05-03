Automatic Weekly Digital PR Stories Suggestions with Reddit and Anthropic

https://n8nworkflows.xyz/workflows/automatic-weekly-digital-pr-stories-suggestions-with-reddit-and-anthropic-3155


# Automatic Weekly Digital PR Stories Suggestions with Reddit and Anthropic

### 1. Workflow Overview

The **Automatic Weekly Digital PR Stories Suggestions with Reddit and Anthropic** workflow is an advanced automation designed to support digital PR professionals by identifying trending news stories on Reddit, analyzing public sentiment from comments, extracting article content, and generating strategic PR story ideas. It operates on a weekly schedule and automates the entire process from data gathering to report distribution.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Topic Preparation**: Receives user-defined topics and prepares them for processing.
- **1.2 Reddit Data Retrieval and Filtering**: Searches Reddit for posts related to topics, filters posts by engagement and content type, and removes duplicates.
- **1.3 Comment Retrieval and Sentiment Analysis**: Retrieves top comments for each post, formats them, and uses Anthropic AI to analyze sentiment and audience insights.
- **1.4 News Content Extraction and Analysis**: Extracts article content from linked URLs using Jina AI and analyzes it with Anthropic AI.
- **1.5 PR Strategy Report Generation**: Combines analyses to generate comprehensive PR story reports using Anthropic AI.
- **1.6 Report Compilation, Storage, and Distribution**: Converts reports to text files, compresses them, uploads to Google Drive, and notifies the team via Mattermost.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Topic Preparation

- **Overview**: This block initializes the workflow by setting the topics of interest and splitting them into individual items for processing.
- **Nodes Involved**: `Schedule Trigger`, `Set Data`, `Split Topics into Items`

##### Nodes:

- **Schedule Trigger**
  - Type: Trigger node
  - Role: Initiates workflow execution every Monday at 6 AM.
  - Configuration: Weekly interval, trigger at Monday 6:00 AM.
  - Inputs: None (trigger)
  - Outputs: Triggers `Set Data`.
  - Edge cases: Misconfigured schedule may cause missed runs.

- **Set Data**
  - Type: Set node
  - Role: Defines topics (e.g., "Donald Trump", "Politics") and stores the Jina API key.
  - Configuration: Topics as multiline string; Jina API key placeholder.
  - Inputs: Trigger from `Schedule Trigger`.
  - Outputs: Passes data to `Split Topics into Items`.
  - Edge cases: Missing or invalid API key; empty topics list.

- **Split Topics into Items**
  - Type: Code node
  - Role: Splits the multiline topics string into individual topic items for parallel processing.
  - Configuration: JavaScript code splits by newline, trims whitespace, outputs array of topic objects.
  - Inputs: Receives topics string from `Set Data`.
  - Outputs: Array of items, each with a single topic.
  - Edge cases: Empty or malformed topics string; trimming errors.

---

#### 2.2 Reddit Data Retrieval and Filtering

- **Overview**: Searches Reddit for posts matching each topic, filters posts by upvotes and content type, and removes duplicates to focus on high-quality trending content.
- **Nodes Involved**: `Search Posts`, `Upvotes Requirement Filtering`, `Set Reddit Posts`, `Remove Duplicates`

##### Nodes:

- **Search Posts**
  - Type: Reddit node
  - Role: Searches Reddit for posts matching the current topic.
  - Configuration: Keyword set dynamically from topic (parameter "meta" in JSON but should be dynamic), location "allReddit", sorted by "hot", returns all results.
  - Inputs: Topic items from `Split Topics into Items`.
  - Outputs: Reddit posts matching the topic.
  - Edge cases: API rate limits, authentication errors, empty results.

- **Upvotes Requirement Filtering**
  - Type: If node
  - Role: Filters posts to include only those with more than 100 upvotes, post type "link", and excludes URLs containing "bsky.app".
  - Configuration: Conditions:
    - `ups` > 100
    - `post_hint` equals "link"
    - `url` does not contain "bsky.app"
  - Inputs: Posts from `Search Posts`.
  - Outputs: Posts passing filter to `Set Reddit Posts`.
  - Edge cases: Posts missing expected fields; posts with unexpected `post_hint`.

- **Set Reddit Posts**
  - Type: Set node
  - Role: Maps and formats Reddit post data into a consistent structure with fields like Title, Subreddit, Upvotes, Comments, URLs, Date, Post ID.
  - Configuration: Assigns fields from Reddit JSON to named properties; converts UNIX timestamp to ISO date.
  - Inputs: Filtered posts from `Upvotes Requirement Filtering`.
  - Outputs: Formatted posts to `Remove Duplicates`.
  - Edge cases: Missing or malformed fields; date conversion errors.

- **Remove Duplicates**
  - Type: Code node
  - Role: Removes duplicate posts based on URL, keeping the one with the highest upvotes; excludes URLs containing "redd.it".
  - Configuration: JavaScript code uses a Map keyed by URL, compares upvotes, excludes "redd.it" URLs.
  - Inputs: Formatted posts from `Set Reddit Posts`.
  - Outputs: Unique posts to `Loop Over Items`.
  - Edge cases: Posts without URLs; non-numeric upvotes; all posts filtered out.

---

#### 2.3 Comment Retrieval and Sentiment Analysis

- **Overview**: For each unique Reddit post, retrieves all comments, extracts the top 30 comments by score (including replies), formats them into Markdown, and analyzes sentiment and audience insights using Anthropic AI.
- **Nodes Involved**: `Loop Over Items`, `Set for Loop`, `Get Comments`, `Extract Top Comments`, `Format Comments`, `Anthropic Chat Model` (Comments Analysis)

##### Nodes:

- **Loop Over Items**
  - Type: SplitInBatches node
  - Role: Processes posts one by one to manage API calls and processing load.
  - Configuration: Default batch size (1).
  - Inputs: Unique posts from `Remove Duplicates`.
  - Outputs: Single post item to `Set for Loop`.
  - Edge cases: Large number of posts may slow workflow.

- **Set for Loop**
  - Type: Set node
  - Role: Prepares and passes post metadata for downstream nodes.
  - Configuration: Copies fields like Title, Subreddit, Upvotes, Comments, URLs, Date, Post ID.
  - Inputs: Single post from `Loop Over Items`.
  - Outputs: To `Get Comments`.
  - Edge cases: Missing fields.

- **Get Comments**
  - Type: Reddit node
  - Role: Retrieves all comments for the given post ID and subreddit.
  - Configuration: Operation "getAll" for post comments.
  - Inputs: Post metadata from `Set for Loop`.
  - Outputs: Raw comments to `Extract Top Comments`.
  - Edge cases: API limits, deleted or removed comments, empty comment threads.

- **Extract Top Comments**
  - Type: Code node
  - Role: Filters and flattens comment tree, excludes deleted comments, selects top 30 comments by score, preserves reply hierarchy.
  - Configuration: JavaScript code recursively processes comments, excludes deleted, sorts by score, rebuilds hierarchy.
  - Inputs: Raw comments from `Get Comments`.
  - Outputs: Filtered comment tree to `Format Comments`.
  - Edge cases: Complex nested replies, missing fields, deleted comments.

- **Format Comments**
  - Type: Code node
  - Role: Converts filtered comment tree into Markdown format with indentation to represent hierarchy.
  - Configuration: JavaScript code recursively formats comments and replies, excludes deleted comments.
  - Inputs: Filtered comments from `Extract Top Comments`.
  - Outputs: Markdown string to `Comments Analysis`.
  - Edge cases: Markdown formatting issues with special characters.

- **Anthropic Chat Model (Comments Analysis)**
  - Type: Anthropic AI node (Langchain)
  - Role: Analyzes Reddit post and comments Markdown to extract sentiment, narratives, audience insights, and PR implications.
  - Configuration:
    - Model: `claude-3-7-sonnet-20250219`
    - Temperature: 0.5
    - Max tokens: 8096
    - Prompt includes Reddit post info and comments in Markdown.
  - Inputs: Markdown comments from `Format Comments`, post metadata from `Set for Loop`.
  - Outputs: Sentiment analysis text to `Get News Content`.
  - Edge cases: API errors, prompt length limits, incomplete analysis.

---

#### 2.4 News Content Extraction and Analysis

- **Overview**: Extracts the content of the news article linked in the Reddit post using Jina AI, then analyzes the content combined with Reddit sentiment data to identify story elements and viral potential.
- **Nodes Involved**: `Get News Content`, `Keep Last`, `Anthropic Chat Model1` (News Analysis)

##### Nodes:

- **Get News Content**
  - Type: HTTP Request node
  - Role: Calls Jina AI API to extract article content from the URL.
  - Configuration:
    - URL: `https://r.jina.ai/{{ URL from Set for Loop }}`
    - Headers: Accept `text/event-stream`, Authorization with Jina API key, custom headers to control response.
    - Retries: 5 attempts with 5 seconds wait.
  - Inputs: Post URL from `Set for Loop`.
  - Outputs: Raw streamed content to `Keep Last`.
  - Edge cases: API failures, invalid URLs, rate limits.

- **Keep Last**
  - Type: Code node
  - Role: Parses streamed response, extracts the last valid JSON entry containing article title and content.
  - Configuration: JavaScript code filters lines starting with `data: {`, parses last JSON.
  - Inputs: Raw response from `Get News Content`.
  - Outputs: Parsed article content to `News Analysis`.
  - Edge cases: Malformed JSON, empty response, parsing errors.

- **Anthropic Chat Model1 (News Analysis)**
  - Type: Anthropic AI node (Langchain)
  - Role: Analyzes extracted news content combined with Reddit metrics and sentiment analysis to evaluate story potential and narrative opportunities.
  - Configuration:
    - Model: `claude-3-7-sonnet-20250219`
    - Temperature: 0.5
    - Max tokens: 8096
    - Prompt includes news content, Reddit metrics, and sentiment analysis.
  - Inputs: Article content from `Keep Last`, sentiment analysis text from `Comments Analysis`.
  - Outputs: News analysis text to `Stories Report`.
  - Edge cases: API errors, prompt length limits.

---

#### 2.5 PR Strategy Report Generation

- **Overview**: Generates a comprehensive PR strategy report combining Reddit data, sentiment analysis, and news content analysis to identify story opportunities, priorities, and execution plans.
- **Nodes Involved**: `Anthropic Chat Model2` (Stories Report), `Set Final Report`

##### Nodes:

- **Anthropic Chat Model2 (Stories Report)**
  - Type: Anthropic AI node (Langchain)
  - Role: Crafts a detailed PR strategy report with story ideas, priority rankings, and strategic recommendations.
  - Configuration:
    - Model: `claude-3-7-sonnet-20250219`
    - Temperature: 0.5
    - Max tokens: 8096
    - Prompt includes news analysis, Reddit metrics, and sentiment analysis.
  - Inputs: News analysis from `News Analysis`, sentiment analysis from `Comments Analysis`.
  - Outputs: PR story report text to `Set Final Report`.
  - Edge cases: API errors, prompt length limits.

- **Set Final Report**
  - Type: Set node
  - Role: Combines Reddit metrics and AI-generated report sections into a final formatted report string.
  - Configuration: Uses expressions to concatenate Reddit post info and extract report sections from AI outputs using regex.
  - Inputs: PR story report text from `Stories Report`, news analysis, and comments analysis.
  - Outputs: Final report text to `Convert to File`.
  - Edge cases: Missing or malformed AI output; regex extraction failures.

---

#### 2.6 Report Compilation, Storage, and Distribution

- **Overview**: Converts final reports to text files, batches and compresses them into a ZIP archive, uploads to Google Drive, sets sharing permissions, and sends a notification with the download link to a Mattermost channel.
- **Nodes Involved**: `Convert to File`, `Loop Over Items`, `Aggregate`, `Merge Binary Files`, `Compress files`, `Google Drive6`, `Google Drive7`, `Send files to Mattermost3`

##### Nodes:

- **Convert to File**
  - Type: ConvertToFile node
  - Role: Converts the final report text into a UTF-8 encoded text file, naming it based on the report headline.
  - Configuration: Filename extracted via regex from report headline.
  - Inputs: Final report text from `Set Final Report`.
  - Outputs: Binary file to `Loop Over Items`.
  - Edge cases: Missing headline; invalid filename characters.

- **Loop Over Items** (second use)
  - Type: SplitInBatches node
  - Role: Processes each file individually for aggregation.
  - Inputs: Files from `Convert to File`.
  - Outputs: Files to `Aggregate`.
  - Edge cases: Large number of files may slow processing.

- **Aggregate**
  - Type: Aggregate node
  - Role: Aggregates all binary files into a single item for compression.
  - Configuration: Includes binaries.
  - Inputs: Files from `Loop Over Items`.
  - Outputs: Aggregated binary files to `Merge Binary Files`.
  - Edge cases: Large file sizes.

- **Merge Binary Files**
  - Type: Code node
  - Role: Collects all binary keys from aggregated item to prepare for compression.
  - Configuration: JavaScript code extracts binary keys.
  - Inputs: Aggregated files from `Aggregate`.
  - Outputs: Binary keys metadata to `Compress files`.
  - Edge cases: Missing binaries.

- **Compress files**
  - Type: Compression node
  - Role: Compresses all report files into a ZIP archive with a timestamped filename.
  - Configuration: Output format ZIP, filename includes date and random suffix.
  - Inputs: Binary files from `Merge Binary Files`.
  - Outputs: ZIP archive to `Google Drive6`.
  - Edge cases: Compression failures; large file sizes.

- **Google Drive6**
  - Type: Google Drive node
  - Role: Uploads the ZIP archive to a specified Google Drive folder.
  - Configuration: Folder ID specified; uses OAuth2 credentials.
  - Inputs: ZIP archive from `Compress files`.
  - Outputs: File metadata to `Google Drive7`.
  - Edge cases: Authentication errors; insufficient permissions.

- **Google Drive7**
  - Type: Google Drive node
  - Role: Sets sharing permissions on the uploaded ZIP file to "anyone with the link can read".
  - Configuration: Role "reader", type "anyone".
  - Inputs: File ID from `Google Drive6`.
  - Outputs: Confirmation to `Send files to Mattermost3`.
  - Edge cases: Permission errors.

- **Send files to Mattermost3**
  - Type: HTTP Request node
  - Role: Sends a notification message to a Mattermost channel with a link to the uploaded ZIP archive.
  - Configuration:
    - URL: Mattermost incoming webhook URL (user must replace placeholders).
    - JSON body includes channel, username, icon URL, and message with download link.
  - Inputs: File ID from `Google Drive7`.
  - Outputs: None.
  - Edge cases: Invalid webhook URL; network errors.

---

### 3. Summary Table

| Node Name                | Node Type                      | Functional Role                                    | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                         |
|--------------------------|--------------------------------|---------------------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Trigger                        | Starts workflow weekly at Monday 6 AM             | None                        | Set Data                    | ## Automatic Weekly Digital PR Stories Suggestions ... Setup instructions included in sticky note content.                        |
| Set Data                 | Set                            | Defines topics and Jina API key                    | Schedule Trigger            | Split Topics into Items      | See above                                                                                                                          |
| Split Topics into Items   | Code                           | Splits topics string into individual topic items  | Set Data                    | Search Posts                | See above                                                                                                                          |
| Search Posts             | Reddit                         | Searches Reddit for posts matching topics          | Split Topics into Items     | Upvotes Requirement Filtering | See above                                                                                                                          |
| Upvotes Requirement Filtering | If                        | Filters posts by upvotes, post type, and URL       | Search Posts                | Set Reddit Posts            | See above                                                                                                                          |
| Set Reddit Posts         | Set                            | Formats Reddit post data into structured fields    | Upvotes Requirement Filtering | Remove Duplicates          | See above                                                                                                                          |
| Remove Duplicates        | Code                           | Removes duplicate posts by URL, keeps highest upvotes | Set Reddit Posts          | Loop Over Items             | See above                                                                                                                          |
| Loop Over Items          | SplitInBatches                 | Processes posts one by one                          | Remove Duplicates           | Set for Loop, Aggregate     | See above                                                                                                                          |
| Set for Loop             | Set                            | Prepares post metadata for downstream nodes       | Loop Over Items             | Get Comments                | See above                                                                                                                          |
| Get Comments             | Reddit                         | Retrieves all comments for a post                   | Set for Loop                | Extract Top Comments        | See above                                                                                                                          |
| Extract Top Comments     | Code                           | Filters top 30 comments by score, preserves replies | Get Comments               | Format Comments             | See above                                                                                                                          |
| Format Comments          | Code                           | Converts comments to Markdown with hierarchy       | Extract Top Comments        | Comments Analysis           | See above                                                                                                                          |
| Anthropic Chat Model (Comments Analysis) | Langchain Anthropic AI | Analyzes Reddit comments for sentiment and insights | Format Comments            | Get News Content            | See above                                                                                                                          |
| Get News Content         | HTTP Request                   | Extracts article content from URL via Jina AI      | Comments Analysis           | Keep Last                   | See above                                                                                                                          |
| Keep Last                | Code                           | Parses streamed response, extracts last JSON entry | Get News Content            | News Analysis               | See above                                                                                                                          |
| Anthropic Chat Model1 (News Analysis) | Langchain Anthropic AI | Analyzes news content with Reddit sentiment data   | Keep Last, Comments Analysis | Stories Report             | See above                                                                                                                          |
| Anthropic Chat Model2 (Stories Report) | Langchain Anthropic AI | Generates PR strategy report from combined analyses | News Analysis, Comments Analysis | Set Final Report         | See above                                                                                                                          |
| Set Final Report         | Set                            | Combines all analysis into final formatted report  | Stories Report              | Convert to File             | See above                                                                                                                          |
| Convert to File          | ConvertToFile                  | Converts report text to UTF-8 text file             | Set Final Report            | Loop Over Items (files)     | See above                                                                                                                          |
| Loop Over Items (files)  | SplitInBatches                 | Processes report files for aggregation              | Convert to File             | Aggregate                   | See above                                                                                                                          |
| Aggregate                | Aggregate                     | Aggregates all report files into one item           | Loop Over Items (files)     | Merge Binary Files          | See above                                                                                                                          |
| Merge Binary Files       | Code                           | Collects binary keys for compression                 | Aggregate                   | Compress files              | See above                                                                                                                          |
| Compress files           | Compression                   | Compresses report files into ZIP archive             | Merge Binary Files          | Google Drive6               | See above                                                                                                                          |
| Google Drive6            | Google Drive                  | Uploads ZIP archive to Google Drive folder           | Compress files             | Google Drive7               | See above                                                                                                                          |
| Google Drive7            | Google Drive                  | Sets sharing permissions on uploaded ZIP             | Google Drive6              | Send files to Mattermost3   | See above                                                                                                                          |
| Send files to Mattermost3 | HTTP Request                 | Sends notification with download link to Mattermost | Google Drive7              | None                       | See above                                                                                                                          |
| Sticky Note              | Sticky Note                   | Workflow description and setup instructions          | None                       | None                       | Contains detailed workflow overview, setup instructions, and customization tips with links to credential guides and APIs.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**
   - Type: Schedule Trigger
   - Configure to run weekly on Monday at 6:00 AM.

2. **Create Set Data Node**
   - Type: Set
   - Add two fields:
     - `Topics` (string, multiline): e.g., "Donald Trump\nPolitics"
     - `Jina API Key` (string): Your Jina API key.
   - Connect Schedule Trigger → Set Data.

3. **Create Split Topics into Items Node**
   - Type: Code
   - JavaScript to split `Topics` string by newline, trim, and output array of topic items.
   - Connect Set Data → Split Topics into Items.

4. **Create Search Posts Node**
   - Type: Reddit
   - Operation: Search
   - Keyword: Use expression to get current topic from input item.
   - Location: allReddit
   - Sort: hot
   - Return all results.
   - Connect Split Topics into Items → Search Posts.
   - Assign Reddit OAuth2 credentials.

5. **Create Upvotes Requirement Filtering Node**
   - Type: If
   - Conditions:
     - `ups` > 100
     - `post_hint` equals "link"
     - `url` does not contain "bsky.app"
   - Connect Search Posts → Upvotes Requirement Filtering.

6. **Create Set Reddit Posts Node**
   - Type: Set
   - Map Reddit post fields to:
     - Title, Subreddit, Upvotes, Comments, Reddit URL, URL, Is_URL, Date (ISO), Post ID.
   - Connect Upvotes Requirement Filtering (true) → Set Reddit Posts.

7. **Create Remove Duplicates Node**
   - Type: Code
   - JavaScript to remove duplicate posts by URL, keep highest upvotes, exclude URLs containing "redd.it".
   - Connect Set Reddit Posts → Remove Duplicates.

8. **Create Loop Over Items Node**
   - Type: SplitInBatches
   - Default batch size 1.
   - Connect Remove Duplicates → Loop Over Items.

9. **Create Set for Loop Node**
   - Type: Set
   - Copy post metadata fields from input.
   - Connect Loop Over Items → Set for Loop.

10. **Create Get Comments Node**
    - Type: Reddit
    - Operation: getAll comments for postId and subreddit from Set for Loop.
    - Connect Set for Loop → Get Comments.
    - Assign Reddit OAuth2 credentials.

11. **Create Extract Top Comments Node**
    - Type: Code
    - JavaScript to flatten comment tree, exclude deleted, select top 30 by score, preserve replies.
    - Connect Get Comments → Extract Top Comments.

12. **Create Format Comments Node**
    - Type: Code
    - JavaScript to convert comment tree to Markdown with indentation.
    - Connect Extract Top Comments → Format Comments.

13. **Create Anthropic Chat Model Node (Comments Analysis)**
    - Type: Langchain Anthropic AI
    - Model: `claude-3-7-sonnet-20250219`
    - Temperature: 0.5
    - Max tokens: 8096
    - Prompt: Analyze Reddit post info and comments Markdown for sentiment and PR insights.
    - Connect Format Comments → Anthropic Chat Model.
    - Assign Anthropic credentials.

14. **Create Get News Content Node**
    - Type: HTTP Request
    - URL: `https://r.jina.ai/{{ URL from Set for Loop }}`
    - Headers: Accept `text/event-stream`, Authorization with Jina API key, plus custom headers.
    - Retries: 5 with 5s delay.
    - Connect Anthropic Chat Model → Get News Content.

15. **Create Keep Last Node**
    - Type: Code
    - Parse streamed response, extract last JSON entry with article title and content.
    - Connect Get News Content → Keep Last.

16. **Create Anthropic Chat Model1 Node (News Analysis)**
    - Type: Langchain Anthropic AI
    - Model: `claude-3-7-sonnet-20250219`
    - Temperature: 0.5
    - Max tokens: 8096
    - Prompt: Analyze news content with Reddit metrics and sentiment analysis.
    - Connect Keep Last → Anthropic Chat Model1.
    - Assign Anthropic credentials.

17. **Create Anthropic Chat Model2 Node (Stories Report)**
    - Type: Langchain Anthropic AI
    - Model: `claude-3-7-sonnet-20250219`
    - Temperature: 0.5
    - Max tokens: 8096
    - Prompt: Generate PR strategy report combining news analysis and sentiment.
    - Connect Anthropic Chat Model1 → Anthropic Chat Model2.
    - Assign Anthropic credentials.

18. **Create Set Final Report Node**
    - Type: Set
    - Compose final report string combining Reddit metrics and AI-generated report sections using expressions and regex extraction.
    - Connect Anthropic Chat Model2 → Set Final Report.

19. **Create Convert to File Node**
    - Type: ConvertToFile
    - Operation: toText
    - Encoding: UTF-8
    - Filename: Extract headline from report for filename.
    - Connect Set Final Report → Convert to File.

20. **Create Loop Over Items Node (for files)**
    - Type: SplitInBatches
    - Connect Convert to File → Loop Over Items.

21. **Create Aggregate Node**
    - Type: Aggregate
    - Aggregate all binary files into one item.
    - Include binaries.
    - Connect Loop Over Items → Aggregate.

22. **Create Merge Binary Files Node**
    - Type: Code
    - Extract binary keys from aggregated item.
    - Connect Aggregate → Merge Binary Files.

23. **Create Compress Files Node**
    - Type: Compression
    - Operation: compress to ZIP
    - Filename: `Trending_Stories_YYYY_MM_DD_random.zip`
    - Input binary keys from Merge Binary Files.
    - Connect Merge Binary Files → Compress files.

24. **Create Google Drive6 Node**
    - Type: Google Drive
    - Operation: Upload file
    - Folder ID: Set target folder
    - Connect Compress files → Google Drive6.
    - Assign Google Drive OAuth2 credentials.

25. **Create Google Drive7 Node**
    - Type: Google Drive
    - Operation: Share file
    - Permissions: Role "reader", Type "anyone"
    - Connect Google Drive6 → Google Drive7.

26. **Create Send files to Mattermost3 Node**
    - Type: HTTP Request
    - Method: POST
    - URL: Mattermost incoming webhook URL (replace placeholders)
    - JSON Body: Notify channel with download link to Google Drive file.
    - Connect Google Drive7 → Send files to Mattermost3.

27. **Create Sticky Note**
    - Type: Sticky Note
    - Content: Workflow overview and setup instructions with links.
    - Position: Top-left for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow automates weekly identification of trending Reddit stories, sentiment analysis, content extraction, and PR strategy report generation and distribution.                                                             | Workflow purpose                                                                                             |
| Reddit OAuth2 API credential setup guide: https://docs.n8n.io/integrations/builtin/credentials/reddit/                                                                                                                        | Credential setup                                                                                             |
| Anthropic Account credential setup guide: https://docs.n8n.io/integrations/builtin/credentials/anthropic/                                                                                                                     | Credential setup                                                                                             |
| Google Drive OAuth2 API credential setup guide: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/                                                                                            | Credential setup                                                                                             |
| Jina API key can be obtained here: https://jina.ai/api-dashboard/key-manager                                                                                                                                                   | API key source                                                                                               |
| Mattermost incoming webhook setup guide: https://developers.mattermost.com/integrate/webhooks/incoming/                                                                                                                       | Notification setup                                                                                           |
| Workflow runs every Monday at 6 AM by default; schedule can be modified in the Schedule Trigger node.                                                                                                                         | Scheduling                                                                                                   |
| The workflow is designed for PR professionals, content marketers, and communications teams to automate social media trend monitoring and PR opportunity identification.                                                      | Target users                                                                                                 |
| Customization options include topic selection, filtering criteria, AI prompt tuning, report formatting, distribution channels, and schedule frequency.                                                                       | Customization                                                                                               |
| Potential failure points include API rate limits, authentication errors, malformed data, and network issues; error handling is minimal and should be enhanced for production use.                                            | Error considerations                                                                                         |
| The workflow uses Anthropic Claude AI models for natural language understanding and generation, leveraging Langchain integration in n8n.                                                                                      | AI integration                                                                                               |
| The workflow uses Jina AI for article content extraction via HTTP streaming API.                                                                                                                                               | Content extraction                                                                                           |
| The workflow sends final report notifications to Mattermost channels via incoming webhooks for team collaboration.                                                                                                          | Collaboration                                                                                               |

---

This documentation provides a complete, structured understanding of the workflow, enabling advanced users and AI agents to reproduce, modify, and anticipate operational considerations effectively.