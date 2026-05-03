🎦🚀 YouTube Video Comment Analysis Agent

https://n8nworkflows.xyz/workflows/-----youtube-video-comment-analysis-agent-2965


# 🎦🚀 YouTube Video Comment Analysis Agent

### 1. Workflow Overview

This workflow, titled **🎦🚀 YouTube Video Comment Analysis Agent**, is designed to assist YouTube content creators by automatically fetching video details and comments, analyzing them, and generating a comprehensive, actionable report. The report provides insights into video performance, audience sentiment, recurring themes in comments, engagement drivers, and content opportunities. It also suggests keywords for discoverability and potential collaboration opportunities.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**: Setting up workflow variables and triggering the workflow.
- **1.2 YouTube API URL Construction**: Dynamically building API URLs for fetching video details and comments.
- **1.3 Data Retrieval**: Fetching video details and paginated comments from YouTube Data API.
- **1.4 Data Processing and Aggregation**: Combining video details and comments into a single JSON object.
- **1.5 AI-Powered Analysis and Reporting**: Using an AI agent to analyze the data and generate a detailed report.
- **1.6 Report Formatting and Distribution**: Converting the report to HTML, emailing it via Gmail, and saving it to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initializes the workflow by defining key variables such as the Google API key and the YouTube video ID. It also includes manual and external triggers to start the workflow.

- **Nodes Involved:**  
  - Workflow Variables  
  - When Executed by Another Workflow (disabled)  
  - When clicking ‘Test workflow’ (manual trigger)  
  - Sticky Note12 (instructions on variables)

- **Node Details:**

  - **Workflow Variables**  
    - Type: Set node  
    - Role: Defines `GOOGLE_API_KEY` and `VIDEO_ID` as workflow variables.  
    - Configuration:  
      - `GOOGLE_API_KEY`: Placeholder string `[YOUR_GOOGLE_API_KEY_GOES_HERE]` (must be replaced by user).  
      - `VIDEO_ID`: Default example value `c5dw_jsGNBk` (should be changed per use case).  
    - Inputs: Trigger nodes  
    - Outputs: Passes variables downstream  
    - Edge Cases: Missing or incorrect API key or video ID will cause API request failures.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows this workflow to be triggered by another workflow with JSON input containing `videoId`.  
    - Disabled: Yes (not active by default).  
    - Inputs: External workflow trigger  
    - Outputs: Passes input JSON downstream  
    - Edge Cases: Disabled; if enabled, must ensure input JSON contains valid `videoId`.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing purposes.  
    - Inputs: User manual trigger  
    - Outputs: Starts workflow execution  
    - Edge Cases: None

  - **Sticky Note12**  
    - Type: Sticky Note  
    - Role: Provides instructions on setting workflow variables and Google API key setup.  
    - Content:  
      ```
      ## 💡 Workflow Variables
      https://cloud.google.com/docs/get-started/access-apis

      - GOOGLE_API_KEY
      - VIDEO_ID - 🖐️CHANGE THIS!!!
      ```

---

#### 2.2 YouTube API URL Construction

- **Overview:**  
  Constructs the YouTube Data API URL dynamically using the provided video ID and Google API key for fetching video details.

- **Nodes Involved:**  
  - Create YouTube API URL  
  - Sticky Note14 (YouTube API documentation links)

- **Node Details:**

  - **Create YouTube API URL**  
    - Type: Code node (JavaScript)  
    - Role: Builds the URL for the YouTube Videos API endpoint with required parts (`snippet`, `contentDetails`, `status`, `statistics`, `player`, `topicDetails`).  
    - Configuration:  
      - Reads `VIDEO_ID` and `GOOGLE_API_KEY` from input JSON.  
      - Throws errors if either is missing.  
      - Constructs URL with dynamic parameters.  
    - Inputs: Workflow Variables node output  
    - Outputs: JSON object with `youtubeUrl` property  
    - Edge Cases: Missing or empty `VIDEO_ID` or `GOOGLE_API_KEY` causes error and halts workflow.

  - **Sticky Note14**  
    - Type: Sticky Note  
    - Role: Provides reference links to YouTube Data API documentation.  
    - Content:  
      ```
      ## YouTube Video Details
      https://developers.google.com/youtube/v3/docs
      https://www.googleapis.com/youtube/v3/videos
      ```

---

#### 2.3 Data Retrieval

- **Overview:**  
  Fetches video details and all comments (with pagination) from YouTube Data API.

- **Nodes Involved:**  
  - Get YouTube Video Details  
  - Get Video Comments with Pagination  
  - Split Out Comments  
  - Sticky Note15 (YouTube comments API docs)  
  - Sticky Note2 (alternate comments retrieval note)

- **Node Details:**

  - **Get YouTube Video Details**  
    - Type: HTTP Request  
    - Role: Sends GET request to the constructed YouTube video details URL.  
    - Configuration:  
      - URL set dynamically from `youtubeUrl` field.  
      - No special options enabled.  
    - Inputs: Create YouTube API URL node output  
    - Outputs: Video details JSON response  
    - Edge Cases: API quota exceeded, invalid API key, invalid video ID, network errors.

  - **Get Video Comments with Pagination**  
    - Type: Code node (JavaScript)  
    - Role: Retrieves all top-level comments for the video by iterating through paginated responses.  
    - Configuration:  
      - Uses YouTube CommentThreads API endpoint.  
      - Loops until no `nextPageToken` is returned.  
      - Collects comment text into an array.  
    - Inputs: Workflow Variables node output  
    - Outputs: JSON object with `comments` array  
    - Edge Cases: API quota limits, missing API key, video with no comments, network errors.

  - **Split Out Comments**  
    - Type: Split Out node  
    - Role: Splits the array of comments into individual items for further processing.  
    - Configuration:  
      - Field to split: `comments`  
    - Inputs: Get Video Comments with Pagination output  
    - Outputs: Individual comment items  
    - Edge Cases: Empty comments array results in no output items.

  - **Sticky Note15**  
    - Type: Sticky Note  
    - Role: Provides links to YouTube comments API documentation.  
    - Content:  
      ```
      ## YouTube Video Comments
      https://developers.google.com/youtube/v3/docs
      https://www.googleapis.com/youtube/v3/commentThreads
      ```

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Notes an alternate method to get the latest 100 comments without pagination (disabled in workflow).  
    - Content:  
      ```
      ## YouTube Video Comments (Alternate)
      Get latest 100 comments without pagination
      ```

---

#### 2.4 Data Processing and Aggregation

- **Overview:**  
  Combines video details and comments into a single JSON object to prepare for AI analysis.

- **Nodes Involved:**  
  - Combine Comments  
  - Merge YouTube Details & Transcript  
  - Create One JSON Object  
  - Sticky Note10 (general processing tool note)

- **Node Details:**

  - **Combine Comments**  
    - Type: Summarize node  
    - Role: Concatenates all comment texts into a single string field named `comments`.  
    - Configuration:  
      - Field to summarize: `comments`  
      - Aggregation method: Concatenate  
    - Inputs: Split Out Comments output  
    - Outputs: Single item with concatenated comments string  
    - Edge Cases: Empty input results in empty string.

  - **Merge YouTube Details & Transcript**  
    - Type: Merge node  
    - Role: Combines video details and concatenated comments into one data structure by position.  
    - Configuration:  
      - Mode: Combine  
      - Combine by: Position  
    - Inputs:  
      - Get YouTube Video Details output  
      - Combine Comments output  
    - Outputs: Merged data object with video details and comments  
    - Edge Cases: Mismatched input lengths could cause missing data.

  - **Create One JSON Object**  
    - Type: Aggregate node  
    - Role: Aggregates all incoming data into a single JSON object for AI consumption.  
    - Configuration:  
      - Aggregate all item data into one object  
    - Inputs: Merge YouTube Details & Transcript output  
    - Outputs: Single aggregated JSON object  
    - Edge Cases: None

  - **Sticky Note10**  
    - Type: Sticky Note  
    - Role: Describes this block as the YouTube Video Details & Comments Processing Tool.  
    - Content:  
      ```
      ## 🛠️YouTube Video Details & Comments Processing Tool
      ```

---

#### 2.5 AI-Powered Analysis and Reporting

- **Overview:**  
  Uses an AI agent (LangChain with OpenAI GPT-4o-mini) to analyze the combined video data and comments, generating a detailed, structured report with insights and recommendations.

- **Nodes Involved:**  
  - YouTube Video Report Agent  
  - gpt-4o-mini (language model)  
  - Sticky Note4 (workflow description)  
  - Sticky Note1 (encouragement note)

- **Node Details:**

  - **YouTube Video Report Agent**  
    - Type: LangChain Agent node  
    - Role: Processes the aggregated JSON data with a conversational AI agent to produce a comprehensive report.  
    - Configuration:  
      - Input text template includes the full JSON data string.  
      - System message instructs the AI to analyze video details and comments, producing a multi-section report covering video overview, sentiment analysis, themes, engagement, content opportunities, audience profile, recommendations, keywords, collaborations, and suggestions for similar content.  
      - Output is a Markdown-formatted report string.  
    - Inputs: Create One JSON Object output  
    - Outputs: JSON with `output` field containing the report text  
    - Edge Cases: AI API quota limits, malformed input data, or prompt failures.

  - **gpt-4o-mini**  
    - Type: LangChain Language Model node  
    - Role: Provides the GPT-4o-mini model for the agent to use.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Credentials: OpenAI API key  
    - Inputs: Connected internally to the agent node  
    - Outputs: AI-generated text  
    - Edge Cases: API key invalid, rate limits, network errors.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Describes the overall workflow purpose and key features.  
    - Content:  
      ```
      # YouTube Video Comment Analysis Agent

      This agent is designed to analyze YouTube video details and comments to generate a **comprehensive and actionable report** for content creators. The report provides insights into:

      - **Video performance**: Metrics such as views, likes, and comments.
      - **Audience engagement**: Identifying what resonates with viewers.
      - **Viewer feedback**: Highlighting trends, interests, and areas for improvement.

      ### Key Features:
      1. **Sentiment Analysis**: Evaluates the tone of comments (positive, negative, neutral) to understand audience sentiment.
      2. **Recurring Themes**: Identifies common topics or questions in comments.
      3. **Engagement Drivers**: Highlights video elements that sparked high engagement.
      4. **Actionable Recommendations**: Offers specific strategies for improving content and addressing viewer needs.
      5. **Keyword Suggestions**: Extracts frequently mentioned terms for better discoverability.
      6. **Collaboration Opportunities**: Suggests potential partnerships based on viewer feedback or related channels.
      7. **Audience Profiling**: Infers audience characteristics such as expertise level and interests.

      ### Objective:
      The goal is to empower YouTube creators with **data-driven insights** to create engaging content that resonates with their audience while addressing viewer needs and preferences.
      ```

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Encouragement note for users to try the workflow.  
    - Content:  
      ```
      ## 👍 Try Me!
      ```

---

#### 2.6 Report Formatting and Distribution

- **Overview:**  
  Converts the AI-generated Markdown report into HTML, then sends it via Gmail and saves it to Google Drive.

- **Nodes Involved:**  
  - Markdown to HTML  
  - Gmail Report  
  - Save Report to Google Drive  
  - Sticky Note (YouTube Video Comment Reporting Agent title)

- **Node Details:**

  - **Markdown to HTML**  
    - Type: Markdown node  
    - Role: Converts the AI-generated Markdown report (`output` field) into HTML format.  
    - Configuration:  
      - Mode: markdownToHtml  
      - Input: `output` field from AI agent node  
      - Output key: `html`  
    - Inputs: YouTube Video Report Agent output  
    - Outputs: HTML formatted report  
    - Edge Cases: Malformed Markdown could cause formatting issues.

  - **Gmail Report**  
    - Type: Gmail node  
    - Role: Sends the HTML report via email.  
    - Configuration:  
      - Recipient: `joe@example.com` (placeholder, should be replaced)  
      - Subject: "YouTube Video Report"  
      - Message body: HTML content from Markdown to HTML node  
      - Option: Attribution disabled  
      - Credentials: Gmail OAuth2 account configured  
    - Inputs: Markdown to HTML output  
    - Outputs: Email sent confirmation  
    - Edge Cases: Invalid email address, Gmail API quota, OAuth token expiration.

  - **Save Report to Google Drive**  
    - Type: Google Drive node  
    - Role: Saves the report as a text file in Google Drive.  
    - Configuration:  
      - File name: "YouTube Video Report - [Video Title]" (dynamic from video details)  
      - Content: AI report Markdown text  
      - Drive: "My Drive"  
      - Folder: Root folder  
      - Operation: Create file from text  
      - Credentials: Google Drive OAuth2 account configured  
    - Inputs: Markdown to HTML output (uses Markdown text)  
    - Outputs: File creation confirmation  
    - Edge Cases: Insufficient permissions, quota exceeded, invalid credentials.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Title note for the reporting agent section.  
    - Content:  
      ```
      ## 📽️ YouTube Video Comment Reporting Agent
      ```

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                               | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                      |
|--------------------------------|----------------------------------|-----------------------------------------------|--------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | External trigger with JSON input               | -                              | Workflow Variables                    |                                                                                                |
| Workflow Variables             | Set                              | Defines GOOGLE_API_KEY and VIDEO_ID            | When Executed by Another Workflow, When clicking ‘Test workflow’ | Create YouTube API URL, Get Video Comments with Pagination | Sticky Note12: Instructions on workflow variables and API key setup                             |
| When clicking ‘Test workflow’  | Manual Trigger                   | Manual start of workflow                        | -                              | Workflow Variables                    |                                                                                                |
| Create YouTube API URL         | Code                             | Builds YouTube API URL for video details       | Workflow Variables             | Get YouTube Video Details             | Sticky Note14: YouTube API documentation links                                                 |
| Get YouTube Video Details      | HTTP Request                    | Fetches video details from YouTube API         | Create YouTube API URL          | Merge YouTube Details & Transcript    |                                                                                                |
| Get Video Comments with Pagination | Code                         | Fetches all video comments with pagination     | Workflow Variables             | Split Out Comments                    | Sticky Note15: YouTube comments API docs; Sticky Note2: Alternate comments retrieval note       |
| Split Out Comments             | Split Out                       | Splits comments array into individual items    | Get Video Comments with Pagination | Combine Comments                    |                                                                                                |
| Combine Comments              | Summarize                       | Concatenates all comments into one string      | Split Out Comments             | Merge YouTube Details & Transcript    |                                                                                                |
| Merge YouTube Details & Transcript | Merge                       | Combines video details and comments            | Get YouTube Video Details, Combine Comments | Create One JSON Object            | Sticky Note10: YouTube Video Details & Comments Processing Tool                                |
| Create One JSON Object         | Aggregate                      | Aggregates all data into one JSON object       | Merge YouTube Details & Transcript | YouTube Video Report Agent          |                                                                                                |
| YouTube Video Report Agent     | LangChain Agent                | AI analysis and report generation               | Create One JSON Object         | Markdown to HTML                      | Sticky Note4: Workflow overview and key features                                               |
| gpt-4o-mini                   | LangChain Language Model       | Provides GPT-4o-mini model for AI agent        | Internal to YouTube Video Report Agent | YouTube Video Report Agent          |                                                                                                |
| Markdown to HTML               | Markdown                      | Converts Markdown report to HTML                | YouTube Video Report Agent     | Gmail Report, Save Report to Google Drive |                                                                                                |
| Gmail Report                  | Gmail                         | Sends the HTML report via email                 | Markdown to HTML               | -                                   |                                                                                                |
| Save Report to Google Drive    | Google Drive                  | Saves the report as a text file in Google Drive | Markdown to HTML               | -                                   |                                                                                                |
| Sticky Note12                 | Sticky Note                   | Instructions on workflow variables              | -                              | -                                   | See above                                                                                      |
| Sticky Note14                 | Sticky Note                   | YouTube API documentation links                 | -                              | -                                   | See above                                                                                      |
| Sticky Note15                 | Sticky Note                   | YouTube comments API documentation links        | -                              | -                                   | See above                                                                                      |
| Sticky Note10                 | Sticky Note                   | YouTube Video Details & Comments Processing Tool | -                              | -                                   | See above                                                                                      |
| Sticky Note4                  | Sticky Note                   | Workflow overview and key features               | -                              | -                                   | See above                                                                                      |
| Sticky Note2                  | Sticky Note                   | Alternate comments retrieval note                | -                              | -                                   | See above                                                                                      |
| Sticky Note                  | Sticky Note                   | Title for reporting agent section                 | -                              | -                                   | See above                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Create a Set Node for Workflow Variables**  
   - Name: `Workflow Variables`  
   - Add two string fields:  
     - `GOOGLE_API_KEY` with placeholder `[YOUR_GOOGLE_API_KEY_GOES_HERE]` (replace with your actual key).  
     - `VIDEO_ID` with the target YouTube video ID (default example: `c5dw_jsGNBk`).  
   - Connect the Manual Trigger node output to this node.

3. **Create a Code Node to Build YouTube Video Details API URL**  
   - Name: `Create YouTube API URL`  
   - Paste JavaScript code that:  
     - Reads `VIDEO_ID` and `GOOGLE_API_KEY` from input.  
     - Throws error if missing.  
     - Constructs URL for YouTube Videos API with parts: `snippet,contentDetails,status,statistics,player,topicDetails`.  
   - Connect `Workflow Variables` node output to this node.

4. **Create an HTTP Request Node to Fetch Video Details**  
   - Name: `Get YouTube Video Details`  
   - Method: GET  
   - URL: Use expression to get `youtubeUrl` from previous node output.  
   - Connect `Create YouTube API URL` node output to this node.

5. **Create a Code Node to Fetch All Comments with Pagination**  
   - Name: `Get Video Comments with Pagination`  
   - Paste JavaScript code that:  
     - Uses YouTube CommentThreads API.  
     - Loops through pages using `nextPageToken`.  
     - Collects all top-level comment texts into an array.  
   - Connect `Workflow Variables` node output to this node.

6. **Create a Split Out Node to Split Comments Array**  
   - Name: `Split Out Comments`  
   - Field to split: `comments`  
   - Connect `Get Video Comments with Pagination` node output to this node.

7. **Create a Summarize Node to Concatenate Comments**  
   - Name: `Combine Comments`  
   - Field to summarize: `comments`  
   - Aggregation: Concatenate  
   - Connect `Split Out Comments` node output to this node.

8. **Create a Merge Node to Combine Video Details and Comments**  
   - Name: `Merge YouTube Details & Transcript`  
   - Mode: Combine  
   - Combine by: Position  
   - Connect `Get YouTube Video Details` and `Combine Comments` node outputs to this node.

9. **Create an Aggregate Node to Create One JSON Object**  
   - Name: `Create One JSON Object`  
   - Aggregate all item data into one JSON object.  
   - Connect `Merge YouTube Details & Transcript` node output to this node.

10. **Create a LangChain Agent Node for AI Analysis**  
    - Name: `YouTube Video Report Agent`  
    - Configure with:  
      - Input text: Template including the JSON stringified data from previous node.  
      - System message: Detailed instructions for AI to generate a comprehensive report with sections on video overview, comment sentiment, themes, engagement, content opportunities, audience profile, recommendations, keywords, collaborations, and suggestions.  
      - Use `gpt-4o-mini` model (create a LangChain Language Model node with this model and connect it).  
    - Connect `Create One JSON Object` node output to this node.

11. **Create a Markdown Node to Convert AI Output to HTML**  
    - Name: `Markdown to HTML`  
    - Mode: markdownToHtml  
    - Input: AI agent node’s `output` field.  
    - Output key: `html`  
    - Connect `YouTube Video Report Agent` node output to this node.

12. **Create a Gmail Node to Send Report via Email**  
    - Name: `Gmail Report`  
    - Recipient: Replace with actual email address (default: `joe@example.com`).  
    - Subject: "YouTube Video Report"  
    - Message: Use `html` field from Markdown node.  
    - Disable attribution.  
    - Configure Gmail OAuth2 credentials.  
    - Connect `Markdown to HTML` node output to this node.

13. **Create a Google Drive Node to Save Report**  
    - Name: `Save Report to Google Drive`  
    - Operation: Create file from text  
    - File name: Use expression to include video title from merged data (e.g., `YouTube Video Report - {{ $('Merge YouTube Details & Transcript').item.json.items.first().snippet.title }}`)  
    - Content: Use AI report Markdown text (`output` field)  
    - Drive: My Drive  
    - Folder: Root  
    - Configure Google Drive OAuth2 credentials.  
    - Connect `Markdown to HTML` node output to this node.

14. **Add Sticky Notes for Documentation and Instructions**  
    - Add notes near relevant nodes with instructions, API documentation links, and usage tips as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow designed to empower YouTube creators with data-driven insights for content improvement and audience engagement.               | Workflow overview and objective                                                                    |
| Google API key must have access to YouTube Data API. Setup guide: https://cloud.google.com/docs/get-started/access-apis                | Workflow Variables Sticky Note12                                                                   |
| YouTube Data API documentation: https://developers.google.com/youtube/v3/docs                                                           | Sticky Note14 and Sticky Note15                                                                    |
| YouTube CommentThreads API documentation: https://developers.google.com/youtube/v3/docs/commentThreads                                  | Sticky Note15                                                                                       |
| AI agent uses LangChain with OpenAI GPT-4o-mini model for natural language analysis and report generation.                              | YouTube Video Report Agent node                                                                     |
| Gmail OAuth2 and Google Drive OAuth2 credentials must be configured for email sending and report saving functionality.                   | Gmail Report and Save Report to Google Drive nodes                                                  |
| Alternate method to fetch latest 100 comments without pagination is available but disabled by default.                                  | Sticky Note2                                                                                        |
| Example video ID and placeholder API key must be replaced before running the workflow.                                                   | Workflow Variables node                                                                             |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the **🎦🚀 YouTube Video Comment Analysis Agent** workflow in n8n. It covers all nodes, their configurations, logical grouping, and operational context to support both human users and automation agents.