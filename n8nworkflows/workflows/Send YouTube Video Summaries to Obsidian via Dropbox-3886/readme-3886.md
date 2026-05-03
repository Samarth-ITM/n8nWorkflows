Send YouTube Video Summaries to Obsidian via Dropbox

https://n8nworkflows.xyz/workflows/send-youtube-video-summaries-to-obsidian-via-dropbox-3886


# Send YouTube Video Summaries to Obsidian via Dropbox

### 1. Workflow Overview

This workflow automates the process of generating detailed, well-formatted YouTube video summaries and saving them as Markdown notes in Dropbox, optimized for use with Obsidian. It is designed to run on a configurable schedule (default every 10 minutes), process all videos in a specified YouTube playlist, and then remove processed videos from that playlist to avoid duplication.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Video Enumeration**: Triggered by a schedule, fetches videos from a specified YouTube playlist.
- **1.2 Video Details and Transcript Retrieval**: For each video, obtains detailed metadata and the transcript via RapidAPI.
- **1.3 Transcript Cleaning**: Processes the raw transcript text to remove noise and format it for AI summarization.
- **1.4 AI Processing and Content Generation**: Uses OpenAI GPT models to generate a structured video summary, YAML frontmatter, and Obsidian-style internal links.
- **1.5 Note Assembly and File Creation**: Combines all generated content into a Markdown file and converts it to a binary file.
- **1.6 Saving and Cleanup**: Saves the Markdown file to Dropbox and removes the processed video from the source playlist.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Video Enumeration

- **Overview:**  
  This block initiates the workflow on a schedule and retrieves the list of videos from the target YouTube playlist.

- **Nodes Involved:**  
  - Run Workflow on Schedule  
  - Get Playlist Videos

- **Node Details:**

  - **Run Workflow on Schedule**  
    - *Type & Role:* Schedule Trigger — starts the workflow automatically at defined intervals (default every 10 minutes).  
    - *Configuration:* Interval and timing can be adjusted in the node parameters.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Triggers "Get Playlist Videos".  
    - *Edge Cases:* Misconfiguration can cause missed runs or excessive API calls.

  - **Get Playlist Videos**  
    - *Type & Role:* YouTube node — fetches all videos from the specified playlist using YouTube OAuth2 credentials.  
    - *Configuration:* Requires the Playlist ID to be set to the target playlist.  
    - *Inputs:* Trigger from schedule node.  
    - *Outputs:* Passes video list to "Get Video Details".  
    - *Edge Cases:* API quota limits, invalid playlist ID, OAuth token expiration.

---

#### 1.2 Video Details and Transcript Retrieval

- **Overview:**  
  For each video, this block obtains detailed metadata and fetches the transcript in VTT format via RapidAPI.

- **Nodes Involved:**  
  - Get Video Details  
  - Get Video Transcript

- **Node Details:**

  - **Get Video Details**  
    - *Type & Role:* HTTP Request — calls RapidAPI's YouTube API to fetch video metadata such as title, channel, likes, duration, and subtitle availability.  
    - *Configuration:* Uses RapidAPI key (via environment variable or direct input). Receives video ID from "Get Playlist Videos".  
    - *Inputs:* Video ID from previous node.  
    - *Outputs:* Video metadata to "Get Video Transcript".  
    - *Edge Cases:* API key issues, rate limiting, missing subtitles.

  - **Get Video Transcript**  
    - *Type & Role:* HTTP Request — fetches the raw transcript (VTT format) from the subtitle URL obtained in "Get Video Details".  
    - *Configuration:* Uses RapidAPI key similarly.  
    - *Inputs:* Subtitle URL from "Get Video Details".  
    - *Outputs:* Raw transcript to "Clean Transcript".  
    - *Edge Cases:* Transcript unavailable, HTTP errors, malformed VTT.

---

#### 1.3 Transcript Cleaning

- **Overview:**  
  Processes the raw VTT transcript to remove timestamps, tags, and unwanted characters, preparing a clean text for AI summarization.

- **Nodes Involved:**  
  - Clean Transcript

- **Node Details:**

  - **Clean Transcript**  
    - *Type & Role:* Code (JavaScript) — custom script that:  
      - Removes timestamps and formatting tags.  
      - Consolidates lines into paragraphs.  
      - Removes vowels (likely a specific cleaning step).  
    - *Configuration:* Custom JavaScript code embedded in the node.  
    - *Inputs:* Raw VTT transcript from "Get Video Transcript".  
    - *Outputs:* Cleaned transcript text to "Summarize Video".  
    - *Edge Cases:* Unexpected transcript formats, script errors, empty transcripts.

---

#### 1.4 AI Processing and Content Generation

- **Overview:**  
  Uses OpenAI GPT models to generate a structured summary of the video, YAML frontmatter for Obsidian, and internal links based on video content.

- **Nodes Involved:**  
  - Summarize Video  
  - Combine Content  
  - Create Frontmatter  
  - Create Links

- **Node Details:**

  - **Summarize Video**  
    - *Type & Role:* OpenAI GPT (GPT-4o) — generates a detailed, structured summary of the video content based on the cleaned transcript and video metadata.  
    - *Configuration:* Uses GPT-4o model, prompt includes video title, channel, and link context.  
    - *Inputs:* Cleaned transcript from "Clean Transcript", video details.  
    - *Outputs:* Summary text to "Combine Content".  
    - *Edge Cases:* API rate limits, prompt failures, incomplete data.

  - **Combine Content**  
    - *Type & Role:* Set node — aggregates summary with key video details (title, channel, likes, duration, link) into a Markdown block.  
    - *Configuration:* Uses expressions to combine data from "Summarize Video", "Get Video Details", and "Get Playlist Videos".  
    - *Inputs:* Summary from "Summarize Video", video metadata.  
    - *Outputs:* Combined Markdown content to "Create Frontmatter".  
    - *Edge Cases:* Missing fields, expression errors.

  - **Create Frontmatter**  
    - *Type & Role:* OpenAI GPT (GPT-4o Mini) — generates YAML frontmatter with tags, aliases, and metadata for Obsidian notes.  
    - *Configuration:* Prompt instructs AI to create appropriate frontmatter based on combined content and video metadata.  
    - *Inputs:* Combined content from "Combine Content".  
    - *Outputs:* YAML frontmatter to "Create Links".  
    - *Edge Cases:* YAML formatting errors, AI output inconsistencies.

  - **Create Links**  
    - *Type & Role:* OpenAI GPT (GPT-4o Mini) — creates Obsidian-style internal links (e.g., `[[Topic]]`) by analyzing the video title and main content.  
    - *Configuration:* Prompt focuses on extracting relevant keywords and concepts for linking.  
    - *Inputs:* Frontmatter from "Create Frontmatter", video title, and combined content.  
    - *Outputs:* Formatted internal links to "Assemble Note".  
    - *Edge Cases:* Link formatting errors, irrelevant keywords.

---

#### 1.5 Note Assembly and File Creation

- **Overview:**  
  Constructs the final Markdown note by assembling frontmatter, metadata, summary, and links; then converts it into a file format suitable for Dropbox upload.

- **Nodes Involved:**  
  - Assemble Note  
  - Create File

- **Node Details:**

  - **Assemble Note**  
    - *Type & Role:* Set node — concatenates YAML frontmatter, video metadata (including calculated duration), AI summary, and internal links into a single Markdown text block.  
    - *Configuration:* Uses expressions to merge inputs from "Create Frontmatter", "Get Video Details", "Get Playlist Videos", "Summarize Video", and "Create Links".  
    - *Inputs:* Frontmatter, video metadata, summary, links.  
    - *Outputs:* Complete Markdown text to "Create File".  
    - *Edge Cases:* Missing data, expression errors.

  - **Create File**  
    - *Type & Role:* Convert to File — converts the assembled Markdown text into a binary file object for storage.  
    - *Configuration:* Sets file extension to `.md`.  
    - *Inputs:* Markdown text from "Assemble Note".  
    - *Outputs:* File binary data to "Save Note to Dropbox".  
    - *Edge Cases:* Conversion failures, encoding issues.

---

#### 1.6 Saving and Cleanup

- **Overview:**  
  Saves the generated Markdown note to Dropbox and removes the processed video from the original YouTube playlist to prevent reprocessing.

- **Nodes Involved:**  
  - Save Note to Dropbox  
  - Remove from Source Playlist

- **Node Details:**

  - **Save Note to Dropbox**  
    - *Type & Role:* Dropbox node — uploads the Markdown file to a specified Dropbox folder (`/Transcripts/` by default).  
    - *Configuration:* Uses Dropbox OAuth2 credentials; filename is a cleaned version of the video title. Save path can be customized.  
    - *Inputs:* File binary data from "Create File".  
    - *Outputs:* Triggers "Remove from Source Playlist".  
    - *Edge Cases:* Authentication errors, path issues, file overwrite conflicts.

  - **Remove from Source Playlist**  
    - *Type & Role:* YouTube node — removes the processed video from the original playlist using the video ID.  
    - *Configuration:* Uses YouTube OAuth2 credentials.  
    - *Inputs:* Video ID from "Get Playlist Videos".  
    - *Outputs:* None (end of workflow for this video).  
    - *Edge Cases:* API quota limits, permission errors, video already removed.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                              | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                                   |
|-------------------------|----------------------------------|----------------------------------------------|--------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------|
| Run Workflow on Schedule | Schedule Trigger                 | Starts workflow on schedule                   | None                     | Get Playlist Videos       |                                                                                                               |
| Get Playlist Videos      | YouTube                         | Fetches videos from target playlist           | Run Workflow on Schedule | Get Video Details         |                                                                                                               |
| Get Video Details        | HTTP Request                    | Retrieves video metadata via RapidAPI         | Get Playlist Videos      | Get Video Transcript      |                                                                                                               |
| Get Video Transcript     | HTTP Request                    | Fetches raw VTT transcript from subtitle URL | Get Video Details        | Clean Transcript          |                                                                                                               |
| Clean Transcript         | Code (JavaScript)               | Cleans and formats raw transcript text        | Get Video Transcript     | Summarize Video           |                                                                                                               |
| Summarize Video          | OpenAI (GPT-4o)                 | Generates detailed video summary               | Clean Transcript         | Combine Content           |                                                                                                               |
| Combine Content          | Set                            | Aggregates summary and metadata into Markdown | Summarize Video          | Create Frontmatter        |                                                                                                               |
| Create Frontmatter       | OpenAI (GPT-4o Mini)            | Creates YAML frontmatter for Obsidian note    | Combine Content          | Create Links              |                                                                                                               |
| Create Links             | OpenAI (GPT-4o Mini)            | Generates Obsidian-style internal links        | Create Frontmatter       | Assemble Note             | Uses GPT-4o Mini to create Obsidian-style internal links (e.g., `[[Topic]]`).                                 |
| Assemble Note            | Set                            | Builds final Markdown note combining all parts | Create Links             | Create File               |                                                                                                               |
| Create File              | Convert to File                 | Converts Markdown text to binary file          | Assemble Note            | Save Note to Dropbox      |                                                                                                               |
| Save Note to Dropbox     | Dropbox                        | Saves Markdown file to Dropbox folder          | Create File              | Remove from Source Playlist |                                                                                                               |
| Remove from Source Playlist | YouTube                      | Removes processed video from original playlist | Save Note to Dropbox     | None                      |                                                                                                               |
| Sticky Note(s)           | Sticky Note                    | Notes and comments                             | N/A                      | N/A                       | Multiple sticky notes present but empty content in provided JSON.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval (default every 10 minutes)  
   - Connect output to "Get Playlist Videos"

2. **Create a YouTube node named "Get Playlist Videos"**  
   - Type: YouTube  
   - Credentials: YouTube OAuth2  
   - Operation: Get Playlist Items  
   - Set Playlist ID to your target playlist  
   - Connect output to "Get Video Details"

3. **Create an HTTP Request node named "Get Video Details"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: RapidAPI endpoint for video details (configured per RapidAPI yt-api docs)  
   - Headers: Include `x-rapidapi-key` from environment variable `RAPIDAPI_API_KEY` or direct input  
   - Query Parameters: Video ID from "Get Playlist Videos"  
   - Connect output to "Get Video Transcript"

4. **Create an HTTP Request node named "Get Video Transcript"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Subtitle URL from "Get Video Details" response  
   - Headers: Include `x-rapidapi-key` as above  
   - Connect output to "Clean Transcript"

5. **Create a Code node named "Clean Transcript"**  
   - Type: Code (JavaScript)  
   - Paste custom script to:  
     - Remove timestamps and tags from VTT transcript  
     - Consolidate lines  
     - Remove vowels (as per original script)  
   - Input: Raw transcript from "Get Video Transcript"  
   - Output: Cleaned transcript text  
   - Connect output to "Summarize Video"

6. **Create an OpenAI node named "Summarize Video"**  
   - Type: OpenAI (GPT-4o)  
   - Credentials: OpenAI API key  
   - Model: GPT-4o  
   - Prompt: Instruct AI to generate structured video summary using cleaned transcript and video metadata (title, channel, link)  
   - Input: Cleaned transcript and video details  
   - Connect output to "Combine Content"

7. **Create a Set node named "Combine Content"**  
   - Type: Set  
   - Configure to combine summary, video title, channel, likes, duration, and link into Markdown text block  
   - Use expressions to pull data from "Summarize Video", "Get Video Details", and "Get Playlist Videos"  
   - Connect output to "Create Frontmatter"

8. **Create an OpenAI node named "Create Frontmatter"**  
   - Type: OpenAI (GPT-4o Mini)  
   - Credentials: OpenAI API key  
   - Model: GPT-4o Mini  
   - Prompt: Generate YAML frontmatter with tags, aliases, and metadata based on combined content and video details  
   - Input: Combined Markdown content from "Combine Content"  
   - Connect output to "Create Links"

9. **Create an OpenAI node named "Create Links"**  
   - Type: OpenAI (GPT-4o Mini)  
   - Credentials: OpenAI API key  
   - Model: GPT-4o Mini  
   - Prompt: Extract relevant keywords and generate Obsidian-style internal links (e.g., `[[Topic]]`) from video title and content  
   - Input: Frontmatter from "Create Frontmatter"  
   - Connect output to "Assemble Note"

10. **Create a Set node named "Assemble Note"**  
    - Type: Set  
    - Configure to concatenate frontmatter, video metadata (including duration calculation), AI summary, and internal links into a single Markdown text  
    - Use expressions to merge inputs from "Create Frontmatter", "Get Video Details", "Get Playlist Videos", "Summarize Video", and "Create Links"  
    - Connect output to "Create File"

11. **Create a Convert to File node named "Create File"**  
    - Type: Convert to File  
    - Set file extension to `.md`  
    - Input: Markdown text from "Assemble Note"  
    - Connect output to "Save Note to Dropbox"

12. **Create a Dropbox node named "Save Note to Dropbox"**  
    - Type: Dropbox  
    - Credentials: Dropbox OAuth2  
    - Operation: Upload file  
    - Path: `/Transcripts/` (default, configurable)  
    - Filename: Cleaned video title from "Get Playlist Videos"  
    - Input: File binary from "Create File"  
    - Connect output to "Remove from Source Playlist"

13. **Create a YouTube node named "Remove from Source Playlist"**  
    - Type: YouTube  
    - Credentials: YouTube OAuth2  
    - Operation: Remove video from playlist  
    - Video ID: From "Get Playlist Videos"  
    - Input: Trigger from "Save Note to Dropbox"  
    - No output (end node)

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Quick Start Guide (PDF) included with the workflow download for step-by-step setup instructions.         | Setup instructions reference in workflow description.                                          |
| Bonus tip: Symlink the Dropbox folder to your Obsidian vault using the Dropbox desktop app for seamless note access. | Enhances workflow integration with Obsidian.                                                  |
| Requires valid OAuth2 credentials for YouTube and Dropbox, and API keys for OpenAI and RapidAPI (yt-api). | Credential and API key setup critical for workflow operation.                                  |
| Video overview available: [![Click Here!](https://i.imgur.com/KDE34Ay.png)](https://www.youtube.com/watch?v=B6kP8vdGP1M) | Official video explaining the workflow.                                                        |

---

This documentation enables advanced users and AI agents to fully understand, reproduce, and modify the workflow while anticipating potential integration and runtime issues.