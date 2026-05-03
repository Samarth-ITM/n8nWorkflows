Digest new YouTube videos to Slack with Google Sheets, RapidAPI & GPT-4o-mini

https://n8nworkflows.xyz/workflows/digest-new-youtube-videos-to-slack-with-google-sheets--rapidapi---gpt-4o-mini-4801


# Digest new YouTube videos to Slack with Google Sheets, RapidAPI & GPT-4o-mini

### 1. Workflow Overview

This workflow automates the process of monitoring multiple YouTube RSS feeds, identifying newly published videos, extracting their metadata and subtitles, generating AI-powered video summaries, and posting digest messages to a Slack channel. It integrates Google Sheets for managing feed URLs and tracking processed videos, RapidAPI for fetching subtitles, and OpenAI's GPT-4o-mini model for content summarization.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Source Management:** Periodic trigger and Slack event trigger to start the workflow; fetching and managing YouTube RSS feed URLs from Google Sheets.
- **1.2 RSS Feed Processing & Filtering:** Downloading RSS feeds, parsing XML to JSON, filtering only newly published videos, and checking for duplicates.
- **1.3 Video Metadata & Subtitles Retrieval:** Fetching video details and subtitles using YouTube API and RapidAPI.
- **1.4 Transcript Processing & AI Summarization:** Parsing subtitles and video chapters, preparing transcript text, and generating AI summaries with GPT-4o-mini.
- **1.5 Logging & Notification:** Logging processed videos in Google Sheets and posting formatted summaries to Slack.
- **1.6 Slack Command Interface & Feed Management:** Listening to Slack mentions to add or remove RSS feed URLs dynamically via an AI agent.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Source Management

**Overview:**  
This block initiates the workflow on a schedule to check feeds regularly and also listens to Slack mentions for interactive feed management. It retrieves the list of YouTube RSS feed URLs from a Google Sheet.

**Nodes Involved:**  
- Schedule Trigger  
- Slack Trigger  
- Get RSS Links  
- Batch RSS Items  
- RSS Feed URL  
- AI Agent (with Google Sheets Tool nodes: Get Rows, Append Row, Delete Row)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* scheduleTrigger  
  - *Config:* Runs every 10 minutes to start the automated polling of feeds.  
  - *Connections:* Outputs to "Get RSS Links".  
  - *Edge Cases:* Workflow pauses if n8n instance is offline or does not run at scheduled times.

- **Slack Trigger**  
  - *Type:* slackTrigger  
  - *Config:* Triggers on "app_mention" events in a specific Slack channel.  
  - *Connections:* Outputs to "Get Channel ID".  
  - *Edge Cases:* Requires proper Slack OAuth2 setup and bot invited to channel. Network or auth failures possible.

- **Get RSS Links**  
  - *Type:* googleSheets  
  - *Config:* Reads RSS feed URLs from a specific Google Sheet and tab, using Google Sheets OAuth2 credentials.  
  - *Connections:* Outputs to "Batch RSS Items".  
  - *Edge Cases:* API quota limits, permission errors, or invalid sheet ID.

- **Batch RSS Items**  
  - *Type:* splitInBatches  
  - *Config:* Processes one feed URL at a time to manage memory and retries easily.  
  - *Connections:* Outputs to "Fetch RSS Feed" for each batch.  
  - *Edge Cases:* Large feed lists could slow processing.

- **RSS Feed URL**  
  - *Type:* set  
  - *Config:* Constructs the RSS URL from the YouTube channel ID received from the "Get Channel ID" node.  
  - *Connections:* Outputs to "AI Agent".  
  - *Edge Cases:* Requires valid channel ID; malformed URLs cause errors.

- **AI Agent (with Get Rows, Append Row, Delete Row)**  
  - *Type:* langchain.agent + googleSheetsTool nodes  
  - *Role:* Interpret Slack message to determine if user wants to add or remove a feed URL.  
  - *Config:* Checks Google Sheet for existence of RSS URL, then either appends or deletes rows accordingly.  
  - *Connections:* Triggered from Slack commands, communicates with Google Sheets nodes.  
  - *Edge Cases:* Ambiguous Slack messages, API permission errors, or concurrency issues when multiple users modify feeds simultaneously.

---

#### 1.2 RSS Feed Processing & Filtering

**Overview:**  
This block downloads each RSS feed XML, converts it to JSON, and filters out videos that are not newly published within a defined freshness window. It ensures only fresh videos proceed further.

**Nodes Involved:**  
- Fetch RSS Feed (HTTP Request)  
- Parse RSS XML (XML node)  
- If newly published (Code)  
- Check for new videos (If)  
- Filter for New Video (Google Sheets)  
- Check if the Video Exists (If)  
- Link is Processed (Set)  
- Batch RSS Items (for retry of feeds)

**Node Details:**

- **Fetch RSS Feed**  
  - *Type:* httpRequest  
  - *Config:* Uses RSS URL from batch input to download raw XML feed content.  
  - *Connections:* Outputs to "Parse RSS XML".  
  - *Edge Cases:* Network failures, invalid URLs, or rate limiting.

- **Parse RSS XML**  
  - *Type:* xml  
  - *Config:* Parses RSS XML to structured JSON for downstream filtering.  
  - *Connections:* Outputs to "If newly published".  
  - *Edge Cases:* Malformed XML or unexpected RSS schema.

- **If newly published**  
  - *Type:* code  
  - *Config:* Compares each video’s published timestamp with the current time minus the freshness window (e.g., last 1000 minutes) to mark new videos.  
  - *Connections:* Outputs the first new video to "Check for new videos".  
  - *Edge Cases:* Timezone issues or missing published date fields.

- **Check for new videos**  
  - *Type:* if  
  - *Config:* Checks if the video is flagged as new.  
  - *Connections:* True branch to "Filter for New Video", False branch to "Batch RSS Items" (skip).  
  - *Edge Cases:* Empty input or logic errors.

- **Filter for New Video**  
  - *Type:* googleSheets  
  - *Config:* Checks Google Sheets "Video Links" tab to see if the video link already exists, preventing duplicates.  
  - *Connections:* Outputs to "Check if the Video Exists".  
  - *Edge Cases:* API latency or rate limits.

- **Check if the Video Exists**  
  - *Type:* if  
  - *Config:* If Google Sheets returns no rows, video is new; otherwise, it has been processed.  
  - *Connections:* True to "Log New Videos" (new video), False to "Link is Processed" (skip).  
  - *Edge Cases:* Google Sheets API errors.

- **Link is Processed**  
  - *Type:* set  
  - *Config:* Sets a message "Link is already processed" to prune or log duplicates.  
  - *Connections:* Loops back to "Batch RSS Items" for next feed.  
  - *Edge Cases:* None significant.

---

#### 1.3 Video Metadata & Subtitles Retrieval

**Overview:**  
This block fetches detailed video metadata from YouTube API and retrieves subtitles via RapidAPI for richer context and transcript extraction.

**Nodes Involved:**  
- Log New Videos (Google Sheets append)  
- Fetch Video Details (YouTube API)  
- Fetch Subtitles URL (RapidAPI)  
- Format Response (Code)  
- Get Subtitles (HTTP Request)  
- Get Timestamps (Code)  
- Check for Timestamps (If)  
- Formatted Chapter Text (Code)  
- Formatted Transcript w/o Chapters (Code)

**Node Details:**

- **Log New Videos**  
  - *Type:* googleSheets  
  - *Config:* Appends new video link to the "Video Links" sheet to mark it processed.  
  - *Connections:* Outputs to "Fetch Video Details".  
  - *Edge Cases:* API write permission issues.

- **Fetch Video Details**  
  - *Type:* youTube (YouTube API node)  
  - *Config:* Retrieves video snippet (title, description) by video ID extracted from the feed link.  
  - *Connections:* Outputs to "Fetch Subtitles URL".  
  - *Credentials:* Authenticated YouTube OAuth2 required.  
  - *Edge Cases:* API quota limits, invalid video ID.

- **Fetch Subtitles URL**  
  - *Type:* httpRequest (RapidAPI)  
  - *Config:* Calls RapidAPI `/subtitles` endpoint with video ID to get available subtitles.  
  - *Credentials:* RapidAPI HTTP Header Auth with X-RapidAPI-Key.  
  - *Connections:* Outputs to "Format Response".  
  - *Edge Cases:* API quota exhaustion or network errors.

- **Format Response**  
  - *Type:* code  
  - *Config:* Selects best available English subtitle URL, ensures URL format includes `fmt=srv1` for transcript XML.  
  - *Connections:* Outputs to "Get Subtitles".  
  - *Edge Cases:* Missing English subtitles fallback handled.

- **Get Subtitles**  
  - *Type:* httpRequest  
  - *Config:* Downloads subtitle XML transcript from the formatted URL.  
  - *Connections:* Outputs to "Get Timestamps".  
  - *Edge Cases:* Network failures or invalid URLs.

- **Get Timestamps**  
  - *Type:* code  
  - *Config:* Extracts chapter timestamps from the video description using regex pattern `(\\d{1,2}:\\d{2})\\s+(.*)`. Prepares chapter start time, title, and end time.  
  - *Connections:* Outputs to "Check for Timestamps".  
  - *Edge Cases:* Videos without timestamped chapters.

- **Check for Timestamps**  
  - *Type:* if  
  - *Config:* Checks if chapters are present to decide between segmented or full transcript processing.  
  - *Connections:* True to "Formatted Chapter Text", False to "Formatted Transcript w/o Chapters".  
  - *Edge Cases:* Empty or malformed chapter data.

- **Formatted Chapter Text**  
  - *Type:* code  
  - *Config:* Maps subtitle transcript XML segments to chapter timestamps, merging text within each chapter time range. Outputs array of chapters with their text for AI input.  
  - *Connections:* Outputs to "Build AI Payload".  
  - *Edge Cases:* Overlapping timestamps, missing chapters.

- **Formatted Transcript w/o Chapters**  
  - *Type:* code  
  - *Config:* Merges all subtitle text into a single string without chapter segmentation, preparing a simpler transcript for AI input.  
  - *Connections:* Outputs to "Generate Summary".  
  - *Edge Cases:* Very long transcripts may exceed token limits.

---

#### 1.4 Transcript Processing & AI Summarization

**Overview:**  
This block builds the AI input payload from transcript and video metadata, sends it to OpenAI GPT-4o-mini to generate a detailed summary and article, then formats the AI response into Slack-compatible message blocks.

**Nodes Involved:**  
- Build AI Payload (Code)  
- Generate Summary (OpenAI GPT-4o-mini)  
- Generate Slack Blocks - Detailed Summary (Code)

**Node Details:**

- **Build AI Payload**  
  - *Type:* code  
  - *Config:* Combines chapter texts or transcript into structured JSON including video title and description for AI consumption.  
  - *Connections:* Outputs to "Generate Summary".  
  - *Edge Cases:* Missing chapter texts or metadata.

- **Generate Summary**  
  - *Type:* langchain.openAi (OpenAI GPT-4o-mini)  
  - *Config:* Sends detailed prompt instructing the model to produce two JSON outputs: a detailed quick rundown summary (bullet points) and a detailed article expanding on each chapter. Uses video title, description, and transcript text as context.  
  - *Credentials:* OpenAI API key required.  
  - *Connections:* Outputs to "Generate Slack Blocks - Detailed Summary".  
  - *Edge Cases:* API rate limits, token length limits.

- **Generate Slack Blocks - Detailed Summary**  
  - *Type:* code  
  - *Config:* Parses AI JSON response, extracts summary and article content, formats the text into Slack message blocks and fallback text for posting. Handles JSON parse errors.  
  - *Connections:* Outputs to "Post to Slack".  
  - *Edge Cases:* AI response malformation or JSON parse failure.

---

#### 1.5 Logging & Notification

**Overview:**  
This block posts the final video summary digest to a Slack channel and manages the batch processing lifecycle.

**Nodes Involved:**  
- Post to Slack  
- Batch RSS Items (loop back)

**Node Details:**

- **Post to Slack**  
  - *Type:* slack  
  - *Config:* Posts the formatted Slack message blocks and fallback text to a selected Slack channel using OAuth2 credentials. Includes a link to the full video.  
  - *Credentials:* Slack OAuth2 with `chat:write` scope.  
  - *Edge Cases:* Slack API rate limits, auth token expiry.

- **Batch RSS Items**  
  - *Role:* Continues processing next RSS feed after posting.  
  - *Edge Cases:* None specific here.

---

#### 1.6 Slack Command Interface & Feed Management

**Overview:**  
This block listens for Slack bot mentions to allow users to dynamically add or remove YouTube RSS feed URLs from the source Google Sheet via conversational commands, mediated by an AI agent.

**Nodes Involved:**  
- Slack Trigger  
- Get Channel ID (HTTP Request)  
- RSS Feed URL (Set)  
- AI Agent  
- Get Rows (Google Sheets Tool)  
- Append Row (Google Sheets Tool)  
- Delete Row (Google Sheets Tool)

**Node Details:**

- **Slack Trigger**  
  - *Type:* slackTrigger  
  - *Config:* Triggers on "app_mention" events to receive user commands in Slack.  
  - *Connections:* Outputs to "Get Channel ID".  
  - *Edge Cases:* Requires Slack setup with event subscriptions.

- **Get Channel ID**  
  - *Type:* httpRequest (RapidAPI)  
  - *Config:* Retrieves YouTube channel details from the mentioned URL to extract channel ID needed for RSS URL construction.  
  - *Credentials:* RapidAPI header auth.  
  - *Edge Cases:* Invalid or malformed YouTube URLs.

- **RSS Feed URL**  
  - *Type:* set  
  - *Config:* Constructs the RSS feed URL from the extracted channel ID for use by the AI Agent.  
  - *Connections:* Outputs to "AI Agent".  
  - *Edge Cases:* Channel ID extraction failure.

- **AI Agent**  
  - *Type:* langchain.agent  
  - *Config:* Decodes Slack message intent to "add" or "remove" feeds and triggers appropriate Google Sheets Tool nodes to update the RSS feed list.  
  - *Connections:* Communicates with "Get Rows", "Append Row", "Delete Row".  
  - *Edge Cases:* Ambiguous user commands or API errors.

- **Get Rows, Append Row, Delete Row**  
  - *Type:* googleSheetsTool  
  - *Config:* Perform read, add, or delete operations on the Google Sheet managing RSS feed URLs.  
  - *Edge Cases:* Google API permission or rate limits.

---

### 3. Summary Table

| Node Name                           | Node Type               | Functional Role                                        | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                              |
|-----------------------------------|-------------------------|-------------------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | scheduleTrigger         | Periodic workflow trigger every 10 minutes            | -                                | Get RSS Links                    | ## 1. Trigger & Source - Kick-off every 10 min; Grab feed list from Google Sheet                         |
| Get RSS Links                    | googleSheets            | Fetch list of RSS feed URLs from Google Sheets        | Schedule Trigger                 | Batch RSS Items                  | ## 1. Trigger & Source                                                                                   |
| Batch RSS Items                  | splitInBatches          | Process feeds one at a time                            | Get RSS Links                   | Fetch RSS Feed                  | ## 2. Fetch & Batch                                                                                      |
| Fetch RSS Feed                  | httpRequest             | Download RSS XML content                               | Batch RSS Items                 | Parse RSS XML                  | ## 2. Fetch & Batch                                                                                      |
| Parse RSS XML                  | xml                     | Convert RSS XML to JSON                                | Fetch RSS Feed                 | If newly published             | ## 3. Parse & Filter                                                                                    |
| If newly published             | code                    | Filter videos published within freshness window        | Parse RSS XML                 | Check for new videos           | ## 3. Parse & Filter                                                                                    |
| Check for new videos           | if                      | Check if video is new                                   | If newly published            | Filter for New Video, Batch RSS Items | ## 3. Parse & Filter                                                                              |
| Filter for New Video           | googleSheets            | Check if video already exists in processed sheet       | Check for new videos           | Check if the Video Exists      | ## 3. Parse & Filter                                                                                    |
| Check if the Video Exists      | if                      | Determine if video is new or processed                  | Filter for New Video           | Log New Videos, Link is Processed | ## 3. Parse & Filter                                                                                  |
| Link is Processed              | set                     | Mark video as already processed                         | Check if the Video Exists      | Batch RSS Items                | ## 3. Parse & Filter                                                                                    |
| Log New Videos                | googleSheets            | Append new video link to Google Sheets                  | Check if the Video Exists      | Fetch Video Details            | ## 5. Log & Notify                                                                                      |
| Fetch Video Details           | youTube                 | Retrieve video metadata from YouTube API                | Log New Videos                | Fetch Subtitles URL            | ## 4. Enrich & Summarize                                                                               |
| Fetch Subtitles URL           | httpRequest             | Get subtitle URLs via RapidAPI                           | Fetch Video Details           | Format Response               | ## 4. Enrich & Summarize                                                                               |
| Format Response              | code                    | Pick best English subtitle URL and format               | Fetch Subtitles URL           | Get Subtitles                 | ## 4. Enrich & Summarize                                                                               |
| Get Subtitles                | httpRequest             | Download subtitle XML transcript                         | Format Response              | Get Timestamps                | ## 4. Enrich & Summarize                                                                               |
| Get Timestamps              | code                    | Extract chapter timestamps from video description       | Get Subtitles                | Check for Timestamps          | ## 4. Enrich & Summarize                                                                               |
| Check for Timestamps        | if                      | Decide if chapters exist to route transcript processing | Get Timestamps              | Formatted Chapter Text, Formatted Transcript w/o Chapters | ## 4. Enrich & Summarize                                                               |
| Formatted Chapter Text      | code                    | Map transcript XML to chapters and merge text           | Check for Timestamps          | Build AI Payload             | ## 4. Enrich & Summarize                                                                               |
| Formatted Transcript w/o Chapters | code                    | Merge transcript text without chapters                   | Check for Timestamps          | Generate Summary             | ## 4. Enrich & Summarize                                                                               |
| Build AI Payload            | code                    | Prepare AI input JSON payload from chapters or transcript | Formatted Chapter Text       | Generate Summary             | ## 4. Enrich & Summarize                                                                               |
| Generate Summary            | langchain.openAi        | Generate detailed summary and article via GPT-4o-mini   | Build AI Payload, Formatted Transcript w/o Chapters | Generate Slack Blocks - Detailed Summary | ## 4. Enrich & Summarize                                                               |
| Generate Slack Blocks - Detailed Summary | code                    | Parse AI JSON response and format Slack message blocks   | Generate Summary             | Post to Slack               | ## 4. Enrich & Summarize                                                                               |
| Post to Slack              | slack                   | Post formatted video summary digest to Slack channel    | Generate Slack Blocks - Detailed Summary | Batch RSS Items            | ## 5. Log & Notify                                                                                      |
| Slack Trigger              | slackTrigger            | Listen for Slack mentions to manage RSS feeds            | -                            | Get Channel ID              | ## 6. Slack Command Interface                                                                          |
| Get Channel ID             | httpRequest             | Retrieve YouTube channel details from URL via RapidAPI   | Slack Trigger               | RSS Feed URL               | ## 6. Slack Command Interface                                                                          |
| RSS Feed URL               | set                     | Construct RSS feed URL from channel ID                    | Get Channel ID              | AI Agent                  | ## 6. Slack Command Interface                                                                          |
| AI Agent                  | langchain.agent         | Determine add/remove intent and update Google Sheets      | RSS Feed URL, Slack Trigger  | Get Rows, Append Row, Delete Row | ## 6. Slack Command Interface                                                                   |
| Get Rows                  | googleSheetsTool        | Check if RSS URL exists in the sheet                      | AI Agent                   | AI Agent                  | ## 6. Slack Command Interface                                                                          |
| Append Row                | googleSheetsTool        | Add RSS URL to the feed list                              | AI Agent                   | AI Agent                  | ## 6. Slack Command Interface                                                                          |
| Delete Row                | googleSheetsTool        | Remove RSS URL from the feed list                         | AI Agent                   | AI Agent                  | ## 6. Slack Command Interface                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to run every 10 minutes.

3. **Add a Google Sheets node ("Get RSS Links"):**  
   - Configure Google Sheets OAuth2 credentials.  
   - Set to read the sheet containing YouTube RSS feed URLs (sheet ID and tab as per your setup).

4. **Add a SplitInBatches node ("Batch RSS Items"):**  
   - Connect output of "Get RSS Links" to this node.  
   - Configure to process one item per batch.

5. **Add HTTP Request node ("Fetch RSS Feed"):**  
   - URL expression: use the RSS URL from batch item JSON (e.g., `{{$json['RSS URL']}}`).  
   - Connect from "Batch RSS Items".

6. **Add XML node ("Parse RSS XML"):**  
   - Connect from "Fetch RSS Feed".  
   - Configure to parse XML content to JSON.

7. **Add a Code node ("If newly published"):**  
   - Connect from "Parse RSS XML".  
   - Write JavaScript to filter entries published within the last ~1000 minutes.  
   - Output only new items.

8. **Add If node ("Check for new videos"):**  
   - Check if the newVideo flag is true.  
   - True branch continues; false branch loops back to next batch.

9. **Add Google Sheets node ("Filter for New Video"):**  
   - Configure to check if video link exists in "Video Links" sheet.  
   - Connect from "Check for new videos".

10. **Add If node ("Check if the Video Exists"):**  
    - Condition: if Google Sheets returns empty, video is new.  
    - True branch to "Log New Videos", False branch to "Link is Processed".

11. **Add Set node ("Link is Processed"):**  
    - Set message "Link is already processed".  
    - Connect to loop back to "Batch RSS Items".

12. **Add Google Sheets node ("Log New Videos"):**  
    - Append new video link to "Video Links" sheet.  
    - Connect from "Check if the Video Exists".

13. **Add YouTube node ("Fetch Video Details"):**  
    - Configure YouTube OAuth2 credentials.  
    - Extract video ID from feed entry link.  
    - Connect from "Log New Videos".

14. **Add HTTP Request node ("Fetch Subtitles URL"):**  
    - Configure RapidAPI credentials for YouTube subtitles endpoint.  
    - Query parameter with video ID.  
    - Connect from "Fetch Video Details".

15. **Add Code node ("Format Response"):**  
    - Extract English subtitles URL with `fmt=srv1` format.  
    - Connect from "Fetch Subtitles URL".

16. **Add HTTP Request node ("Get Subtitles"):**  
    - Download subtitles XML from the URL.  
    - Connect from "Format Response".

17. **Add Code node ("Get Timestamps"):**  
    - Parse video description for chapter timestamps using regex.  
    - Connect from "Get Subtitles".

18. **Add If node ("Check for Timestamps"):**  
    - Check if timestamps exist.  
    - True branch to "Formatted Chapter Text".  
    - False branch to "Formatted Transcript w/o Chapters".

19. **Add Code node ("Formatted Chapter Text"):**  
    - Map subtitles to chapter timestamps, merge text per chapter.  
    - Connect from "Check for Timestamps".

20. **Add Code node ("Formatted Transcript w/o Chapters"):**  
    - Merge all subtitle text into one string.  
    - Connect from "Check for Timestamps".

21. **Add Code node ("Build AI Payload"):**  
    - Combine chapter text or full transcript with video metadata into JSON.  
    - Connect from "Formatted Chapter Text".

22. **Add OpenAI node ("Generate Summary"):**  
    - Use GPT-4o-mini model.  
    - Configure prompt to generate detailed summary and article JSON based on input.  
    - Connect both "Build AI Payload" and "Formatted Transcript w/o Chapters" into this node.

23. **Add Code node ("Generate Slack Blocks - Detailed Summary"):**  
    - Parse AI JSON response, format Slack message blocks and fallback text.  
    - Connect from "Generate Summary".

24. **Add Slack node ("Post to Slack"):**  
    - Configure Slack OAuth2 credentials with appropriate scopes.  
    - Post message with text and blocks to target Slack channel.  
    - Connect from "Generate Slack Blocks - Detailed Summary".

25. **Connect the "Post to Slack" node output to "Batch RSS Items" to process next feed.**

26. **For Slack command interface:**  
    - Add Slack Trigger node (app_mention event).  
    - Add HTTP Request node ("Get Channel ID") querying RapidAPI channel details.  
    - Add Set node ("RSS Feed URL") to build RSS URL from channel ID.  
    - Add langchain Agent node to parse add/remove commands from Slack text.  
    - Connect to Google Sheets Tool nodes "Get Rows", "Append Row", and "Delete Row" to modify RSS feed list accordingly.

27. **Ensure all credentials for Google Sheets (OAuth2), Slack (OAuth2), YouTube (OAuth2), RapidAPI (HTTP Header Auth), and OpenAI are created and properly linked.**

28. **Test the workflow end-to-end:**  
    - Add/remove RSS feeds via Slack mention commands.  
    - Confirm new videos are detected and summarized in Slack every 10 minutes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Google Sheets Template:** Make a copy of the master sheet here: `https://docs.google.com/spreadsheets/d/1i3jZ_0npsEVtrMUSzGPV3Lbta3q7sArHHeNomlAuSMg/edit?usp=sharing`. Connect it with a Google Sheets OAuth2 credential that has edit access for reading/writing rows.                                                                                                                                                                                                                                                  | Setup instructions for Google Sheets data store.                                                                                                     |
| **RapidAPI – Subtitles Endpoint:** Sign up at `https://yt-api.p.rapidapi.com/subtitles` (free 300 calls/month). Store your X-RapidAPI-Key in an n8n HTTP Header Auth credential. Never hard-code keys.                                                                                                                                                                                                                                                                                                               | Required API for fetching YouTube subtitles.                                                                                                         |
| **OpenAI API Key:** Create at `https://platform.openai.com/account/api-keys` and save in an n8n OpenAI credential. Monitor token usage carefully, as long transcripts consume many tokens.                                                                                                                                                                                                                                                                                                                          | AI summarization service using GPT-4o-mini.                                                                                                         |
| **Slack OAuth Setup:** Create a Slack App at `https://api.slack.com/apps`. Add bot scopes: `chat:write`, `channels:read`, and `groups:read`. Enable event subscriptions with your n8n webhook URL, subscribe to `app_mention`. Install app and invite bot to your channel. Test commands like `@YourBot add/remove {YouTube Channel link}`. Reinstall app after changing scopes.                                                                                                                                              | Slack credential and event trigger setup.                                                                                                           |
| The workflow is designed to handle errors gracefully by filtering invalid or duplicate videos early, but network failures, API quota limits, and auth token expiries require monitoring and retries configured in n8n settings.                                                                                                                                                                                                                                                                                     | Operational considerations.                                                                                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.