YouTube Video Summary to Discord with GPT-4o, Slack Approval, and Google Sheets

https://n8nworkflows.xyz/workflows/youtube-video-summary-to-discord-with-gpt-4o--slack-approval--and-google-sheets-4584


# YouTube Video Summary to Discord with GPT-4o, Slack Approval, and Google Sheets

---

## 1. Workflow Overview

This n8n workflow automates the process of monitoring a specific YouTube channel for new videos, summarizing the video descriptions using GPT-4o, managing human approval via Slack, and finally posting approved summaries to a Discord channel while logging data in Google Sheets. It is designed for content teams or social media managers who want to streamline video summarization and distribution efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Metadata Extraction:** Detects new YouTube videos via RSS feed and extracts the video ID for further data retrieval.
- **1.2 Video Metadata Retrieval:** Fetches detailed metadata about the video from the YouTube Data API.
- **1.3 AI Summary Generation:** Uses an OpenAI GPT agent to generate a concise summary of the video description.
- **1.4 Storage & Human Approval:** Stores the summary and metadata in Google Sheets and sends the summary to Slack for human approval.
- **1.5 Final Publishing:** Posts the approved summary to a Discord channel.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Metadata Extraction

**Overview:**  
This block listens for new videos published on a specified YouTube channel using its RSS feed. It extracts the video ID from the video URL for use in subsequent API calls.

**Nodes Involved:**  
- YouTube RSS Trigger  
- Extract Channel ID (Function Node)

#### Node: YouTube RSS Trigger

- **Type & Role:** RSS Feed Read Trigger node; triggers the workflow when a new video appears in the YouTube channel’s RSS feed.
- **Configuration:**  
  - Feed URL set to the specified YouTube channel's RSS feed URL (`https://www.youtube.com/feeds/videos.xml?channel_id=UCf3dc-Y3k5vBTCVpPCAfG6g`).
  - Polls continuously, triggering on new feed items.
- **Expressions/Variables:** Outputs feed items including video URL (`link`), title, and `isoDate`.
- **Input/Output:** No input; outputs new RSS feed item JSON.
- **Edge Cases:**  
  - If the RSS feed is unavailable or returns errors, the trigger will fail or pause.
  - Rate limits or network issues may cause delays.
- **Version Specifics:** n8n v1 compatible.

#### Node: Extract Channel ID

- **Type & Role:** Code (Function) node; extracts the YouTube video ID from the feed's video URL.
- **Configuration:**  
  - JavaScript code parses the `link` field to extract the video ID parameter from URLs like `https://www.youtube.com/watch?v=VIDEO_ID`.
- **Expressions/Variables:** Uses `item.json.link` to derive `videoId`.
- **Input/Output:** Input from RSS trigger; outputs item with `videoId` property.
- **Edge Cases:**  
  - URLs that do not follow the expected structure may cause errors.
  - Missing or malformed links will result in undefined `videoId`.
- **Version Specifics:** Uses n8n v2 Code node syntax.

---

### 2.2 Video Metadata Retrieval

**Overview:**  
This block fetches detailed metadata about the detected video from the YouTube Data API using the extracted video ID.

**Nodes Involved:**  
- Fetch Video Details (HTTP Request)

#### Node: Fetch Video Details

- **Type & Role:** HTTP Request node; queries YouTube Data API v3 for video details.
- **Configuration:**  
  - URL: `https://www.googleapis.com/youtube/v3/videos`
  - Query parameters include:
    - `part=snippet` to get video title, description, and publish date.
    - `id={{ $json.videoId }}` dynamically set from previous node.
    - `key=YOUR-API_KEY` placeholder for YouTube API key.
- **Expressions/Variables:** Uses `{{$json.videoId}}` from previous node.
- **Input/Output:** Input from Extract Channel ID; outputs API response containing video metadata.
- **Edge Cases:**  
  - Invalid or missing API key will cause authorization errors.
  - Invalid video ID or deleted videos will return empty or error responses.
  - API rate limits may cause throttling.
- **Version Specifics:** n8n v4+ HTTP Request with query parameter support.

---

### 2.3 AI Summary Generation

**Overview:**  
Generates a concise summary of the video description using an OpenAI GPT agent configured for summarization tasks.

**Nodes Involved:**  
- Summarize Agent (Langchain Agent node)  
- OpenAI GPT Summary Model (Language Model node)

#### Node: Summarize Agent

- **Type & Role:** Langchain Agent node; orchestrates AI summarization using an OpenAI model.
- **Configuration:**  
  - Input text combines video title and description:  
    ```
    Title: {{ $json.items[0].snippet.title }}
    Description: {{ $json.items[0].snippet.description }}
    ```
  - System message prompts the AI to summarize the video description or state if not available.
  - Uses a defined prompt type.
- **Expressions/Variables:** Accesses video metadata from `Fetch Video Details` output.
- **Input/Output:** Input from Fetch Video Details; output is the summarized text.
- **Edge Cases:**  
  - Empty or missing descriptions lead to fallback messages.
  - GPT model failure or API errors may cause timeouts or empty outputs.
- **Version Specifics:** Uses Langchain integration version 1.9.

#### Node: OpenAI GPT Summary Model

- **Type & Role:** Language Model node; OpenAI GPT-4o-mini model used as the backend for the agent.
- **Configuration:**  
  - Model set to `gpt-4o-mini`.
  - No additional options specified.
- **Input/Output:** Provides AI model responses to the Summarize Agent.
- **Edge Cases:**  
  - API quota or credential errors.
  - Model unavailability or latency.
- **Version Specifics:** n8n Langchain v1.2.

---

### 2.4 Storage & Human Approval

**Overview:**  
Stores the video summary and metadata in Google Sheets and sends the summary to a Slack channel, waiting for user approval or rejection.

**Nodes Involved:**  
- Store results to Google Sheet  
- Send Summary for Approval (Slack)

#### Node: Store results to Google Sheet

- **Type & Role:** Google Sheets node; appends a new row with video title, summary, URL, and published date.
- **Configuration:**  
  - Document ID and sheet name specified (Google Sheets document URL and sheet gid=0).
  - Columns mapped explicitly:
    - Title from `Fetch Video Details` (`items[0].snippet.title`)
    - Summary from AI output (`$json.output`)
    - Video URL from RSS Trigger (`link`)
    - Video Published Date from RSS Trigger (`isoDate`)
- **Input/Output:** Input from Summarize Agent; output passes to Slack node.
- **Edge Cases:**  
  - Google Sheets OAuth token expiration or permission errors.
  - Spreadsheet limits or quota exceeded.
  - Missing or malformed data causing append failure.
- **Version Specifics:** n8n v4.5 with OAuth2 credentials.

#### Node: Send Summary for Approval (Slack)

- **Type & Role:** Slack node with `sendAndWait` operation; sends the message to a Slack channel and waits for user interaction.
- **Configuration:**  
  - Sends message to Slack channel ID `C08TTV0CC3E` (all-nathing).
  - Message includes video title and summary.
  - Appends no attribution.
  - Waits for approval or rejection response.
- **Input/Output:** Input from Google Sheets node; output triggers Discord node upon approval.
- **Edge Cases:**  
  - Slack API rate limits or auth failures.
  - User does not respond, leading to potential workflow timeout.
  - Message formatting issues with Slack markdown.
- **Version Specifics:** Slack node version 2.3 supporting interactive message waiting.

---

### 2.5 Final Publishing

**Overview:**  
Upon Slack approval, the workflow posts the approved video summary to a specified Discord channel.

**Nodes Involved:**  
- Post Approved Summary (Discord)

#### Node: Post Approved Summary

- **Type & Role:** Discord node; posts a message to the configured Discord channel.
- **Configuration:**  
  - Message content includes video title and summary from Google Sheets stored data.
  - `guildId` and `channelId` fields are placeholders (empty strings), requiring configuration.
- **Input/Output:** Input from Slack approval node.
- **Edge Cases:**  
  - Discord webhook misconfiguration or permission issues.
  - Missing or empty channel/guild IDs prevent posting.
  - Discord API rate limits.
- **Version Specifics:** Discord node version 2.

---

## 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                          | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                   |
|---------------------------|----------------------------------|----------------------------------------|---------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| YouTube RSS Trigger       | RSS Feed Read Trigger             | Detects new YouTube videos via RSS     | —                         | Extract Channel ID          | See Sticky Note: Section 1: Trigger & Metadata Extraction                                    |
| Extract Channel ID        | Code (Function)                  | Extracts video ID from RSS video URL   | YouTube RSS Trigger       | Fetch Video Details         | See Sticky Note: Section 1: Trigger & Metadata Extraction                                    |
| Fetch Video Details       | HTTP Request                    | Fetches video metadata from YouTube API | Extract Channel ID        | Summarize Agent            | See Sticky Note: Section 2: Video Metadata Retrieval                                         |
| Summarize Agent           | Langchain Agent (OpenAI GPT)     | Summarizes video description via GPT   | Fetch Video Details        | Store results to Google Sheet | See Sticky Note: Section 3: AI Summary Generation                                            |
| OpenAI GPT Summary Model  | Langchain Language Model         | Provides GPT-4o-mini model for summarization | Summarize Agent (AI model) | Summarize Agent (AI output) |                                                                                               |
| Store results to Google Sheet | Google Sheets                   | Appends video summary and metadata     | Summarize Agent           | Send Summary for Approval   | See Sticky Note: Section 4: Storage & Human Approval                                        |
| Send Summary for Approval | Slack (sendAndWait)              | Sends summary to Slack for approval     | Store results to Google Sheet | Post Approved Summary       | See Sticky Note: Section 4: Storage & Human Approval                                        |
| Post Approved Summary     | Discord                         | Posts approved summary to Discord      | Send Summary for Approval  | —                          | See Sticky Note: Section 5: Final Publishing                                                |
| Sticky Note               | Sticky Note                     | Documentation and section notes         | —                         | —                          | Provides explanations for workflow sections                                                 |
| Sticky Note9              | Sticky Note                     | Workflow assistance contact info        | —                         | —                          | Assistance contacts and links                                                              |
| Sticky Note4              | Sticky Note                     | Full workflow overview and documentation | —                         | —                          | Comprehensive workflow documentation                                                       |
| Sticky Note5              | Sticky Note                     | Section 2 notes on video metadata retrieval | —                         | —                          | Section 2: Video Metadata Retrieval                                                        |
| Sticky Note6              | Sticky Note                     | Section 3 notes on AI summary generation | —                         | —                          | Section 3: AI Summary Generation                                                           |
| Sticky Note7              | Sticky Note                     | Section 4 notes on storage and approval | —                         | —                          | Section 4: Storage & Human Approval                                                        |
| Sticky Note8              | Sticky Note                     | Section 5 notes on final publishing      | —                         | —                          | Section 5: Final Publishing                                                                |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “YouTube RSS Trigger” node (RSS Feed Read Trigger):**  
   - Set feed URL to `https://www.youtube.com/feeds/videos.xml?channel_id=UCf3dc-Y3k5vBTCVpPCAfG6g`.  
   - Configure polling to trigger on new videos.

2. **Add “Extract Channel ID” node (Code node):**  
   - JavaScript code to parse the video link from RSS and extract the video ID parameter:  
     ```js
     return items.map(item => {
       const link = item.json.link;
       const videoId = link.split("v=")[1].split("&")[0];
       return { json: { videoId } };
     });
     ```  
   - Connect input from YouTube RSS Trigger.

3. **Add “Fetch Video Details” node (HTTP Request):**  
   - URL: `https://www.googleapis.com/youtube/v3/videos`  
   - Method: GET  
   - Query Parameters:  
     - `part` = `snippet`  
     - `id` = `={{ $json.videoId }}` (dynamic from previous node)  
     - `key` = `YOUR-API_KEY` (replace with valid YouTube Data API key)  
   - Connect input from Extract Channel ID node.

4. **Add “OpenAI GPT Summary Model” node (Langchain Language Model):**  
   - Set model to `gpt-4o-mini` (or substitute with your preferred OpenAI GPT model).  
   - No additional options required.  
   - No direct input connections (used by agent node).

5. **Add “Summarize Agent” node (Langchain Agent):**  
   - Set input text:  
     ```
     Title: {{ $json.items[0].snippet.title }}
     Description: {{ $json.items[0].snippet.description }}
     ```  
   - Set system message:  
     `"You are a YouTube video summarizer. If the description is not available just tell that it is not available. Summarize the following video description"`  
   - Link the agent to use the OpenAI GPT Summary Model node as its language model.  
   - Connect input from Fetch Video Details node.

6. **Add “Store results to Google Sheet” node:**  
   - Connect input from Summarize Agent output.  
   - Configure Google Sheets credentials with OAuth2.  
   - Set Spreadsheet ID to your target Google Sheet.  
   - Set Sheet Name (gid=0 or your target sheet).  
   - Map columns as follows:  
     - `Title`: `={{ $('Fetch Video Details').item.json.items[0].snippet.title }}`  
     - `Summary`: `={{ $json.output }}` (agent output)  
     - `Video URL`: `={{ $('YouTube RSS Trigger').item.json.link }}`  
     - `Video Published Date`: `={{ $('YouTube RSS Trigger').item.json.isoDate }}`  
   - Operation: Append.

7. **Add “Send Summary for Approval” node (Slack):**  
   - Connect input from Google Sheets node.  
   - Configure Slack OAuth credentials.  
   - Set channel ID to your approval Slack channel (example: `C08TTV0CC3E`).  
   - Message:  
     ```
     The summary of a video titled "{{ $json.Title }}" is given below:

     {{ $json.Summary }}
     ```  
   - Operation: sendAndWait to wait for user interaction.

8. **Add “Post Approved Summary” node (Discord):**  
   - Connect input from Slack node (on approval).  
   - Configure Discord webhook or OAuth credentials.  
   - Specify Discord Guild ID and Channel ID where summary should be posted.  
   - Message content:  
     ```
     New Video Summary
     Title: {{ $('Store results to Google Sheet').item.json.Title }}
     Summary: {{ $('Store results to Google Sheet').item.json.Summary }}
     ```  
   - Resource: message.

9. **Connect nodes as follows:**  
   - YouTube RSS Trigger → Extract Channel ID → Fetch Video Details → Summarize Agent → Store results to Google Sheet → Send Summary for Approval → Post Approved Summary.

10. **Test each step carefully:**  
    - Validate RSS feed triggers correctly.  
    - Verify API key permissions for YouTube Data API.  
    - Confirm GPT summarization outputs expected summaries.  
    - Ensure Google Sheets write access is working.  
    - Slack message sends and waits for interaction.  
    - Discord message posts only upon approval.

---

## 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Workflow automates YouTube video monitoring, summarization, human approval, and multi-platform posting.                            | Workflow overview per Sticky Note4                                                                                      |
| Contact for assistance: Yaron@nofluff.online                                                                                       | Sticky Note9                                                                                                            |
| Video tips and tutorials available: https://www.youtube.com/@YaronBeen/videos and LinkedIn https://www.linkedin.com/in/yaronbeen/ | Sticky Note9                                                                                                            |
| Workflow sections clearly documented with sticky notes for maintainability and clarity.                                            | Multiple sticky notes (Sticky Note, Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8)                              |
| Requires valid YouTube API key with access to YouTube Data API v3.                                                                 | Mentioned in Fetch Video Details node                                                                                   |
| Requires configured OAuth2 credentials for Google Sheets and Slack with appropriate scopes.                                        | Mentioned in respective nodes                                                                                           |
| Discord node requires proper webhook or bot token with permissions to post messages in the target channel/guild.                   | Mentioned in Post Approved Summary node                                                                                  |

---

*Disclaimer:* The provided text is derived solely from an automated n8n workflow. It fully complies with content policies and contains no illegal or protected elements. All data processed is legal and public.

---