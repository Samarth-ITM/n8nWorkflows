Summarize YouTube video transcripts in Discord with Gemini and Supabase

https://n8nworkflows.xyz/workflows/summarize-youtube-video-transcripts-in-discord-with-gemini-and-supabase-13749


# Summarize YouTube video transcripts in Discord with Gemini and Supabase

This document provides a technical analysis of the "Summarize YouTube video transcripts in Discord with Gemini and Supabase" n8n workflow.

### 1. Workflow Overview
This workflow automates the process of extracting, summarizing, and archiving YouTube video content shared within a Discord server. When a user posts a YouTube link, the system fetches the video's metadata and transcript, uses Google Gemini to generate a structured summary in the original language, saves the data to a Supabase database, and replies to the Discord thread with the summary.

**Logical Blocks:**
*   **1.1 Input Reception & Validation:** Listens to Discord messages and filters for valid YouTube URLs using Regex.
*   **1.2 Media Extraction (yt-dlp):** Uses command-line tools to fetch video metadata and download auto-generated subtitles.
*   **1.3 Transcript Processing:** Reads raw subtitle files (VTT), cleans timestamps/HTML tags, and prepares a plain-text transcript.
*   **1.4 AI Summarization:** Utilizes Gemini 2.5 Flash to create a TLDR and a detailed summary.
*   **1.5 Persistence & Reporting:** Saves data to Supabase and manages Discord replies (including message chunking for long summaries).
*   **1.6 Error Management:** A global error handler that logs failures to Supabase and notifies the Discord channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
*   **Nodes Involved:** `Discord Trigger`, `Extract YouTube URL`, `Is YouTube URL?`, `Discord Not YouTube Reply`.
*   **Role:** Captures every message in a specific channel. A Code node uses a regular expression to identify standard YouTube URLs, Shorts, or Live links. If no URL is found, the workflow either stops or sends a "Not a YouTube link" message.
*   **Edge Cases:** Non-YouTube links are rejected. Messages without text content return an empty array.

#### 2.2 Media Extraction (yt-dlp)
*   **Nodes Involved:** `yt-dlp Get Metadata`, `Parse Metadata`.
*   **Role:** Executes a shell command using `yt-dlp`. It retrieves metadata (title, views, channel) and attempts to download `.vtt` subtitle files to a local `/tmp` directory.
*   **Technical Details:** Requires `yt-dlp` installed on the host/container. Uses a `cookies.txt` file to bypass bot detection.
*   **Edge Cases:** Private, deleted, or geo-blocked videos will cause the `yt-dlp` command to fail.

#### 2.3 Transcript Processing
*   **Nodes Involved:** `Read Subtitle File`, `Parse Transcript`.
*   **Role:** Reads the downloaded subtitle file from the system disk. The `Parse Transcript` node removes VTT headers (WEBVTT), timestamps, styling tags, and duplicate lines caused by scrolling captions.
*   **Edge Cases:** If no subtitles are available (e.g., music videos or videos with disabled captions), the workflow throws a "No subtitles available" error.

#### 2.4 AI Summarization
*   **Nodes Involved:** `Message a model` (Gemini), `Prepare Insert Data`.
*   **Role:** Passes the cleaned transcript to Gemini 2.5 Flash. The prompt enforces a specific "TLDR" and "Summary" format and insists the AI writes in the same language as the transcript.
*   **Technical Details:** Uses the `models/gemini-2.5-flash` model via Google AI Studio.

#### 2.5 Persistence & Reporting
*   **Nodes Involved:** `Save to Supabase`, `Prepare Success Log`, `Log Run`, `Prepare Messages for Discord`, `Discord Reply`.
*   **Role:** Inserts the metadata, transcript, and AI summary into the `videos` table. It then logs the successful execution into the `runs` table.
*   **Technical Details:** Discord has a 2000-character limit. The `Prepare Messages for Discord` node intelligently chunks the summary into multiple messages while preserving word integrity and adding a header to the first chunk.

#### 2.6 Error Management
*   **Nodes Involved:** `Error Trigger`, `Prepare Error Data`, `Log Run Error`, `Discord Error Reply`.
*   **Role:** Acts as a safety net. It classifies errors based on which node failed (e.g., `yt_dlp_failed` vs `no_subtitles`) and logs the technical details to Supabase before informing the user.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Discord Trigger | Discord Trigger | Entry Point | None | Extract YouTube URL | 1. Trigger & URL Detection |
| Extract YouTube URL | Code | URL Extraction | Discord Trigger | Is YouTube URL? | 1. Trigger & URL Detection |
| Is YouTube URL? | If | Logic Filter | Extract YouTube URL | yt-dlp Get Metadata, Discord Not YouTube Reply | 1. Trigger & URL Detection |
| Discord Not YouTube Reply | Discord | Rejection Notice | Is YouTube URL? | None | 1. Trigger & URL Detection |
| yt-dlp Get Metadata | Execute Command | External Tool | Is YouTube URL? | Parse Metadata | 2. Video Data Extraction |
| Parse Metadata | Code | Data Cleaning | yt-dlp Get Metadata | Read Subtitle File | 2. Video Data Extraction |
| Read Subtitle File | Execute Command | File I/O | Parse Metadata | Parse Transcript | 3. Transcript Processing |
| Parse Transcript | Code | Text Cleaning | Read Subtitle File | Message a model | 3. Transcript Processing |
| Message a model | Google Gemini | AI Analysis | Parse Transcript | Prepare Insert Data | 4. AI Summarization |
| Prepare Insert Data | Code | Data Mapping | Message a model | Save to Supabase | 4. AI Summarization |
| Save to Supabase | Supabase | Database Storage | Prepare Insert Data | Prepare Success Log | 5. Save & Notify |
| Prepare Success Log | Code | Log Prep | Save to Supabase | Log Run | 5. Save & Notify |
| Log Run | Supabase | Audit Logging | Prepare Success Log | Prepare Messages for Discord | 5. Save & Notify |
| Prepare Messages for Discord | Code | Message Chunking | Log Run | Discord Reply | 5. Save & Notify |
| Discord Reply | Discord | User Notification | Prepare Messages for Discord | None | 5. Save & Notify |
| Error Trigger | Error Trigger | Global Error Catch | None | Prepare Error Data | 6. Error Handling |
| Prepare Error Data | Code | Error Classification | Error Trigger | Log Run Error | 6. Error Handling |
| Log Run Error | Supabase | Error Logging | Prepare Error Data | Discord Error Reply | 6. Error Handling |
| Discord Error Reply | Discord | Error Notification | Log Run Error | None | 6. Error Handling |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites
1.  **n8n Environment:** Ensure `yt-dlp` is installed on the machine running n8n. Create a `cookies.txt` file at `/home/node/.n8n/cookies.txt` if you encounter "Sign in to confirm you are not a bot" errors.
2.  **Discord:** Create an application in the [Discord Developer Portal](https://discord.com/developers/applications). Enable **Message Content Intent**.
3.  **Google AI Studio:** Obtain an API key for Gemini 2.5 Flash.
4.  **Supabase:** Create two tables using the following SQL:
    ```sql
    CREATE TABLE videos (video_id TEXT PRIMARY KEY, title TEXT, channel TEXT, upload_date TEXT, duration INT, view_count INT, description TEXT, transcript TEXT, ai_summary TEXT, thumbnail_url TEXT, discord_shared_at TIMESTAMPTZ, channel_id TEXT);
    CREATE TABLE runs (video_id TEXT PRIMARY KEY, process_status TEXT NOT NULL, error_type TEXT, notes TEXT, date_added TIMESTAMPTZ DEFAULT now());
    ```

#### Step-by-Step Build
1.  **Trigger Setup:** Add a **Discord Trigger** node. Set the pattern to "every" and specify your Guild and Channel IDs.
2.  **URL Parsing:** Add a **Code** node (`Extract YouTube URL`) using RegEx to capture the 11-character video ID from the message content.
3.  **Validation:** Add an **If** node to route the workflow only if `is_youtube` is true.
4.  **Metadata Fetching:** Add an **Execute Command** node. Use `yt-dlp` to fetch JSON metadata and download subtitles to `/tmp/yt_{video_id}`.
5.  **Metadata Parsing:** Add a **Code** node to extract specific fields (Title, Channel, Views) from the `yt-dlp` stdout.
6.  **Subtitle Retrieval:** Add an **Execute Command** node to `cat` the `.vtt` file from the `/tmp` folder. Set **On Error** to "Continue".
7.  **Transcript Cleaning:** Add a **Code** node to strip VTT syntax and HTML tags, resulting in a clean string.
8.  **AI Integration:** Add a **Google Gemini** node. Set the prompt to request a TLDR and a Summary in the transcript's language.
9.  **Data Persistence:** Add a **Supabase** node to insert the combined object into the `videos` table.
10. **Audit Log:** Add another **Supabase** node to insert the execution status into the `runs` table.
11. **Discord Output:** Add a **Code** node to split the summary into chunks under 1900 characters. Follow this with a **Discord** node (Message resource) to send the chunks back to the channel.
12. **Error Path:** Add an **Error Trigger** linked to a Code node that parses error messages, followed by a Supabase node for logging and a Discord node for user alerts.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Discord Developer Portal | [Configure Bot & Intents](https://discord.com/developers/applications) |
| Google AI Studio | [Get Gemini API Key](https://aistudio.google.com/apikey) |
| Supabase Management | [Database Setup](https://supabase.com) |
| yt-dlp Documentation | [Advanced Command Usage](https://github.com/yt-dlp/yt-dlp) |
| Message Splitting Logic | Handles Discord's 2000 character limit per message to prevent API rejection. |