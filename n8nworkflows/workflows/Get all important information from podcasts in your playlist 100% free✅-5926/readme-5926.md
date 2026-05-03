Get all important information from podcasts in your playlist 100% free✅

https://n8nworkflows.xyz/workflows/get-all-important-information-from-podcasts-in-your-playlist-100--free--5926


# Get all important information from podcasts in your playlist 100% free✅

---

# Comprehensive Reference Document for the Workflow  
**Title:** Get all important information from podcasts in your playlist 100% free✅

---

## 1. Workflow Overview

This n8n workflow automates the process of extracting, transcribing, and summarizing podcast episodes from a YouTube playlist, then organizing the output into a Google Sheet. It is designed for users who want to capture detailed, structured summaries of podcasts without manual effort.

**Target Use Cases:**

- Podcast content creators or consumers wanting quick, detailed overviews of episodes.
- Researchers or analysts collecting structured podcast insights.
- Automating content extraction from YouTube playlist podcasts into spreadsheet format for review or sharing.

**Logical Blocks:**

- **1.1 Input Reception and Playlist Extraction:** Trigger and retrieval of all videos from a specified YouTube playlist.
- **1.2 Video Metadata Processing:** Extract video URLs, titles, and IDs for downstream use.
- **1.3 Transcript Retrieval:** Query an external API to get transcripts of each video.
- **1.4 Script Assembly and Preprocessing:** Combine transcript segments and extract initial sentences for summarization.
- **1.5 AI Summarization:** Use language models to generate detailed summaries from transcript snippets.
- **1.6 Data Storage:** Update a Google Sheet with video titles, full transcripts, summaries, and row references.
- **1.7 Batch Processing:** Manage processing of multiple videos in batches to handle API and rate limits efficiently.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Playlist Extraction

**Overview:**  
Starts the workflow manually and fetches all videos from a specified YouTube playlist.

**Nodes Involved:**  
- When clicking 'Execute workflow' (Manual Trigger)  
- YouTube Playlist (YouTube API node)  
- YouTube Playlist Note (Sticky Note)

#### Node Details

- **When clicking 'Execute workflow'**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow on user command.  
  - Configuration: Default manual trigger with no parameters.  
  - Input: None  
  - Output: Starts the chain to fetch playlist data.  
  - Failure: No external dependencies; minimal failure risk.

- **YouTube Playlist**  
  - Type: YouTube API Node  
  - Role: Retrieves all videos from a specified YouTube playlist.  
  - Configuration:  
    - Resource: `playlistItem`  
    - Operation: `getAll`  
    - `playlistId`: A specific playlist ID (redacted)  
    - `part`: Requests multiple parts including contentDetails, snippet, id, and status for comprehensive video info.  
  - Credentials: Uses OAuth2 credentials for YouTube API access.  
  - Input: Manual trigger node  
  - Output: JSON containing video items of the playlist.  
  - Failure Modes: API auth errors, invalid playlist ID, network timeouts.  
  - Sticky Note: Explains how to change the playlist ID and credentials.

---

### 2.2 Video Metadata Processing

**Overview:**  
Extracts video URLs, titles, and video IDs from the playlist data and assigns row IDs for data management.

**Nodes Involved:**  
- titles/URL (Function Node)  
- video ID (Code Node)  
- Split Out (Split Out Node)  
- titles/URL Note (Sticky Note)  
- video ID Note (Sticky Note)  
- Sticky Note11 (Step 2 Label)

#### Node Details

- **titles/URL**  
  - Type: Function Node  
  - Role: Maps playlist video data into structured objects with URL, title, and a numeric ID starting at 2.  
  - Key expressions: Constructs YouTube watch URLs using videoId from snippet.resourceId.  
  - Input: YouTube Playlist output  
  - Output: List of objects with `id`, `url`, and `title`.  
  - Failure Modes: Missing `videoId` or malformed playlist data.

- **video ID**  
  - Type: Code Node  
  - Role: Extracts the video ID from the YouTube URL for each item.  
  - Key expressions: Parses URL string to isolate video ID parameter.  
  - Input: titles/URL output  
  - Output: JSON with `id`, `videoId`, and `title`.  
  - Failure Modes: Invalid URL format, missing `v=` parameter.

- **Split Out**  
  - Type: Split Out Node  
  - Role: Splits the incoming data array based on fields `id, videoId, title` for batch processing.  
  - Input: video ID output  
  - Output: Individual items split for loop processing.  
  - Failure Modes: Empty input or missing fields.

---

### 2.3 Transcript Retrieval

**Overview:**  
For each video, calls the Supadata API to obtain the transcript text.

**Nodes Involved:**  
- supadata (HTTP Request Node)  
- supadata Note (Sticky Note)  
- Sticky Note12 (Step 3 Label)

#### Node Details

- **supadata**  
  - Type: HTTP Request Node  
  - Role: Sends a GET request to Supadata API with video URL to fetch transcript data.  
  - Configuration:  
    - URL dynamically built using `https://api.supadata.ai/v1/transcript?url=https://youtu.be/{{ $json.videoId }}`  
    - HTTP Header includes `x-api-key` for authentication.  
  - Input: Loop Over Items1 output (video-specific data)  
  - Output: Transcript JSON including content segments.  
  - Failure Modes: API key invalidation, rate limiting, network errors, API downtime.

---

### 2.4 Script Assembly and Preprocessing

**Overview:**  
Combines transcript segments into a single continuous script string and extracts the first few sentences for summary input.

**Nodes Involved:**  
- scirpt (Code Node)  
- Code1 (Code Node)  
- scirpt Note (Sticky Note)  
- Code1 Note (Sticky Note)  
- Sticky Note13 (Step 4 Label)

#### Node Details

- **scirpt**  
  - Type: Code Node  
  - Role: Aggregates all `text` fields from transcript segments array into one string representing the full script.  
  - Key logic: Nested loops over items and content arrays to concatenate text with spaces.  
  - Input: supadata output  
  - Output: JSON with combined `script` string.  
  - Failure Modes: Missing or malformed transcript content, empty text arrays.

- **Code1**  
  - Type: Code Node  
  - Role: Extracts the first 5 sentences (adjustable) from the script for summarization input.  
  - Key logic: Uses regex to split text into sentences, slices first 5, joins them back.  
  - Input: scirpt output  
  - Output: JSON with `firstTenSentences` field (actually first five sentences).  
  - Failure Modes: Scripts with no punctuation, very short transcripts.

---

### 2.5 AI Summarization

**Overview:**  
Generates detailed, structured summaries of podcast scripts using a Langchain AI Agent configured with a clear prompt.

**Nodes Involved:**  
- AI Agent1 (Langchain Agent Node)  
- Groq Chat Model (Language Model Node)  
- Sticky Note6 (Step 1 Label)  
- Sticky Note10 (Step 5 Label)  
- Sticky Note13 (AI Model Info)

#### Node Details

- **AI Agent1**  
  - Type: Langchain Agent Node  
  - Role: Uses AI to create a comprehensive summary of podcast scripts based on the first few sentences extracted.  
  - Configuration:  
    - Prompt instructs detailed, sectioned summaries with headings and bullet points.  
    - Injects `{{ $json.firstTenSentences }}` as input text.  
  - Input: Code1 output  
  - Output: JSON with detailed podcast summary.  
  - Failure Modes: AI model downtime, rate limits, prompt interpretation errors.  
  - Connected LLM: Groq Chat Model.

- **Groq Chat Model**  
  - Type: Langchain Language Model Node  
  - Role: Backend large language model providing AI completions.  
  - Configuration:  
    - Model: `llama-3.1-8b-instant`  
  - Credentials: Groq API key.  
  - Input: AI Agent1 LLM input  
  - Output: Summarization text for AI Agent1.

---

### 2.6 Data Storage

**Overview:**  
Updates a Google Sheet with video titles, full scripts, AI-generated summaries, and row identifiers for easy reference.

**Nodes Involved:**  
- Google Sheets3 (Google Sheets Node)  
- Sticky Note10 (Step 5 Label)

#### Node Details

- **Google Sheets3**  
  - Type: Google Sheets Node  
  - Role: Updates existing rows in a specified Google Sheet with new data fields.  
  - Configuration:  
    - Operation: `update`  
    - Sheet Name and Document ID configured (redacted)  
    - Columns updated: `video titles`, `script`, `summery` (summary), and `row_number` as a match key.  
    - Uses expressions to pull data from previous nodes: titles, scripts, summaries.  
  - Credentials: Google OAuth2 credentials.  
  - Input: AI Agent1 output  
  - Output: Confirmation of sheet update.  
  - Failure Modes: Sheet access denied, invalid document ID, incorrect column mapping.

---

### 2.7 Batch Processing

**Overview:**  
Processes videos in batches to efficiently handle large playlists and avoid API rate limits.

**Nodes Involved:**  
- Loop Over Items1 (Split In Batches Node)  
- Connections looping back to supadata and Google Sheets3

#### Node Details

- **Loop Over Items1**  
  - Type: Split In Batches Node  
  - Role: Controls batch processing of video items, sending each to transcript retrieval and then onward.  
  - Configuration: Default batch options (size unspecified)  
  - Input: Split Out output (list of videos)  
  - Output: Iteratively sends single or batch items to supadata and Google Sheets3.  
  - Failure Modes: Batch size too large causing API errors, incomplete batch processing.

---

## 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                                    | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                         |
|----------------------------|-------------------------------|---------------------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger               | Starts workflow execution                          | None                          | YouTube Playlist             |                                                                                                   |
| YouTube Playlist           | YouTube API Node              | Fetches all videos from playlist                   | When clicking 'Execute workflow' | titles/URL                   | 📺 **YouTube Playlist** - How to change playlistId and credentials                                |
| titles/URL                 | Function Node                 | Extracts URLs, titles, assigns row IDs             | YouTube Playlist              | video ID                     | 🔗 **titles/URL** - Extracts video URLs and titles with row IDs                                  |
| video ID                   | Code Node                    | Extracts video ID from URLs                         | titles/URL                    | Split Out                    | 🆔 **video ID** - Extracts video ID from YouTube URLs                                            |
| Split Out                  | Split Out Node               | Splits video items for batch processing            | video ID                     | Loop Over Items1             |                                                                                                   |
| Loop Over Items1           | Split In Batches Node        | Batch processing control                            | Split Out                    | supadata, Google Sheets3     |                                                                                                   |
| supadata                   | HTTP Request Node            | Calls transcript API for each video                 | Loop Over Items1              | scirpt                      | 📝 **supadata** - Calls Supadata API for transcripts                                              |
| scirpt                    | Code Node                    | Combines transcript segments into full script      | supadata                     | Code1                       | 📜 **scirpt** - Combines transcript text segments                                                |
| Code1                      | Code Node                    | Extracts first few sentences for summarization     | scirpt                       | AI Agent1                   | ✂️ **Code1** - Extracts initial sentences for summary                                            |
| AI Agent1                  | Langchain Agent Node         | Generates detailed podcast summary                  | Code1                        | Google Sheets3              |                                                                                                   |
| Groq Chat Model            | Langchain LLM Node           | Provides AI completions for summarization           | AI Agent1 (ai_languageModel)  | AI Agent1                   | 🤖 **Groq Chat Model** - AI language model; can change prompt or model                            |
| Google Sheets3             | Google Sheets Node           | Updates Google Sheet with podcast data              | AI Agent1                    | Loop Over Items1             | 📊 **Google Sheets3** - Updates sheet with titles, scripts, summaries, and row numbers           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start workflow manually.

2. **Add YouTube Playlist Node**  
   - Type: YouTube API  
   - Resource: `playlistItem`  
   - Operation: `getAll`  
   - Parameters:  
     - `playlistId`: Set to your desired YouTube playlist ID.  
     - `part`: `*`, `contentDetails`, `snippet`, `id`, `status`  
   - Credentials: Configure with valid YouTube OAuth2 credentials.  
   - Connect Manual Trigger → YouTube Playlist.

3. **Add Function Node "titles/URL"**  
   - Purpose: Extract video URLs and titles, assign IDs starting at 2.  
   - Code: Map playlist item to `{id, url, title}` format using `snippet.resourceId.videoId`.  
   - Connect YouTube Playlist → titles/URL.

4. **Add Code Node "video ID"**  
   - Purpose: Extract video ID from URL string.  
   - Code: Parse URL string for `v=` parameter to get videoId.  
   - Connect titles/URL → video ID.

5. **Add Split Out Node**  
   - Purpose: Split items on fields `id, videoId, title` for batch processing.  
   - Connect video ID → Split Out.

6. **Add Split In Batches Node "Loop Over Items1"**  
   - Purpose: Iterate over videos in batches (default batch size or as needed).  
   - Connect Split Out → Loop Over Items1.

7. **Add HTTP Request Node "supadata"**  
   - Purpose: Fetch transcript from Supadata API for each video.  
   - URL: `https://api.supadata.ai/v1/transcript?url=https://youtu.be/{{ $json.videoId }}`  
   - Headers: Include `x-api-key` with your Supadata API key.  
   - Connect Loop Over Items1 → supadata.

8. **Add Code Node "scirpt"**  
   - Purpose: Combine transcript content arrays into a single script string.  
   - Logic: Loop through `content` array segments and concatenate `text` fields.  
   - Connect supadata → scirpt.

9. **Add Code Node "Code1"**  
   - Purpose: Extract first 5 sentences from the combined script for summarization input.  
   - Logic: Use regex to split script into sentences and join the first 5.  
   - Connect scirpt → Code1.

10. **Add Langchain Agent Node "AI Agent1"**  
    - Purpose: Generate detailed summaries from initial sentences.  
    - Prompt: Define a detailed summarization prompt, injecting `{{ $json.firstTenSentences }}`.  
    - Connect Code1 → AI Agent1.

11. **Add Langchain LLM Node "Groq Chat Model"**  
    - Purpose: Provide the AI language model backend for the agent.  
    - Model: `llama-3.1-8b-instant` or other configured model.  
    - Credentials: Configure with valid Groq API key.  
    - Connect AI Agent1 (ai_languageModel) → Groq Chat Model.

12. **Add Google Sheets Node "Google Sheets3"**  
    - Purpose: Update rows with video titles, scripts, summaries, and row numbers.  
    - Operation: `update`  
    - Sheet: Configure Google Sheet document ID and sheet name.  
    - Columns: Map `video titles`, `script`, `summery`, `row_number` with relevant fields from previous nodes.  
    - Credentials: Configure with Google OAuth2 credentials.  
    - Connect AI Agent1 → Google Sheets3.

13. **Connect Google Sheets3 back to Loop Over Items1**  
    - Purpose: Continue batch processing until all videos processed.

14. **Add Sticky Notes for Documentation (Optional)**  
    - Add explanatory notes near each block for clarity (e.g., step numbers, node purposes).

---

## 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                        |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| The workflow uses Supadata API for transcript retrieval: ensure API key validity and access.    | Supadata API documentation at https://supadata.ai (verify API changes regularly).      |
| Langchain Agent configured with Groq's `llama-3.1-8b-instant` model; customizable prompt text. | Groq AI platform info: https://groq.com                                                 |
| YouTube OAuth2 credentials require correct scopes for playlist access.                           | YouTube Data API v3 scopes and OAuth2 setup documentation.                             |
| Google Sheets OAuth2 credentials must have edit permissions for target document.                 | Google Sheets API docs for OAuth2 setup and access rights.                             |
| Batch processing via Split In Batches node prevents API rate limits and improves performance.   | Adjust batch size based on API limits and workflow performance needs.                  |
| The "script" node can be customized to preprocess transcripts differently (e.g., filtering).    | Modify code node logic based on transcript format or cleaning requirements.            |
| Summarization prompt includes explicit instructions for detailed, sectioned summaries.          | Prompt engineering impacts summary quality; adjust for target audience needs.          |
| This workflow can be adapted for other video transcript or summarization services by changing API nodes. | Replace Supadata Node with alternative transcript API as needed.                       |

---

**Disclaimer:**  
This document reflects a workflow created entirely with n8n automation tool, respecting all applicable content policies. No illegal, offensive, or protected data is included. All processed data is publicly accessible and legally available.

---