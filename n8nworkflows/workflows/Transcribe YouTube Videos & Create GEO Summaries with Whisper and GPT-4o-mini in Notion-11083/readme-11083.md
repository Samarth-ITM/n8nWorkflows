Transcribe YouTube Videos & Create GEO Summaries with Whisper and GPT-4o-mini in Notion

https://n8nworkflows.xyz/workflows/transcribe-youtube-videos---create-geo-summaries-with-whisper-and-gpt-4o-mini-in-notion-11083


# Transcribe YouTube Videos & Create GEO Summaries with Whisper and GPT-4o-mini in Notion

### 1. Workflow Overview

This workflow automates the process of transforming YouTube video content into structured GEO (Goal, Execution, Outcome) summaries stored in Notion. It is designed for content analysts, marketers, educators, or knowledge managers who want to convert video information into searchable, actionable insights with minimal manual effort.

**Use Cases:**
- Summarizing educational or tutorial videos with clear objectives and outcomes.
- Creating structured content briefs from video assets for easy reference.
- Enhancing content discoverability in Notion databases via metadata and keywords.
- Automating repetitive transcription and summarization tasks on a schedule.

**Logical Blocks:**

- **1.1 Input Reception & Scheduling:** Initiates the workflow on a scheduled interval.
- **1.2 Video Metadata Fetch:** Retrieves YouTube video metadata using OAuth2.
- **1.3 Audio Download & Transcription:** Downloads audio via RapidAPI and transcribes it using OpenAI Whisper.
- **1.4 Transcript Validation & Metadata Merge:** Filters empty transcripts and merges text with metadata.
- **1.5 AI Analysis & GEO Summary Extraction:** Uses GPT-4o-mini with structured output parsing to generate GEO summaries.
- **1.6 Data Formatting & Storage:** Parses AI JSON output and creates a new page in Notion with the summary and metadata.
- **1.7 Security & Credential Management:** Handles API credentials and ensures secure access.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Scheduling

- **Overview:** Triggers the workflow periodically (default hourly) to process new or specified YouTube videos.
- **Nodes Involved:** 
  - Schedule Trigger
- **Node Details:**
  - **Schedule Trigger**
    - Type: Schedule trigger node
    - Configured to run hourly via interval setting
    - Initiates the workflow chain by triggering YouTube video fetch
    - Potential failure: Misconfigured trigger interval or disabled trigger may prevent workflow execution.

#### 1.2 Video Metadata Fetch

- **Overview:** Fetches detailed metadata for a specified YouTube video using OAuth2 authentication.
- **Nodes Involved:** 
  - YouTube - Fetch Video Details
  - Set - Prepare Video Metadata
- **Node Details:**
  - **YouTube - Fetch Video Details**
    - Type: YouTube API node
    - Operation: Get video resource details
    - Credentials: YouTube OAuth2 required
    - Outputs video metadata including title, description, publish date, thumbnail, and video ID
    - Failure cases: OAuth token expiration, API quota exceeded, invalid video ID
  - **Set - Prepare Video Metadata**
    - Type: Set node
    - Purpose: Extracts and organizes key metadata fields (videoId, title, description, publish date, thumbnail URL, video URL)
    - Uses expressions to map fields from YouTube API response
    - Input: YouTube video details JSON
    - Output: Simplified metadata JSON for downstream use
    - Edge cases: Missing fields if API response changes, incorrect expression syntax

#### 1.3 Audio Download & Transcription

- **Overview:** Downloads the video’s audio track using a RapidAPI service and transcribes it to text via OpenAI Whisper.
- **Nodes Involved:** 
  - HTTP - Get YouTube Audio
  - HTTP - Download Audio File
  - OpenAI - Transcribe Audio (Whisper)
- **Node Details:**
  - **HTTP - Get YouTube Audio**
    - Type: HTTP Request node
    - Calls RapidAPI YouTube audio downloader with videoId parameter
    - Requires a valid RapidAPI key (replace placeholder)
    - Output: URL to audio file
    - Failure: Invalid API key, rate limit, unavailable audio URL
  - **HTTP - Download Audio File**
    - Type: HTTP Request node
    - Downloads audio file as binary data from URL provided
    - Configured to return file response format
    - Failure: Network issues, invalid URL, file size limits
  - **OpenAI - Transcribe Audio (Whisper)**
    - Type: OpenAI Whisper node
    - Transcribes audio binary to text transcript
    - Credentials: OpenAI API key
    - Potential failures: Audio format unsupported, API timeout, transcription errors

#### 1.4 Transcript Validation & Metadata Merge

- **Overview:** Ensures that empty transcripts are filtered out and merges transcription text with video metadata for AI analysis.
- **Nodes Involved:** 
  - IF - Skip if No Transcript
  - Merge - Attach Metadata + Transcript
- **Node Details:**
  - **IF - Skip if No Transcript**
    - Type: If node
    - Condition: Checks if the transcript text is not empty
    - Outputs to either continue or skip downstream processing
    - Edge case: Empty or null transcripts skip AI processing
  - **Merge - Attach Metadata + Transcript**
    - Type: Merge node (combine mode)
    - Combines metadata and transcript JSON objects by position (pairwise)
    - Output: Single JSON item containing both metadata and transcript fields
    - Potential failure: Misalignment of inputs, missing data on one branch

#### 1.5 AI Analysis & GEO Summary Extraction

- **Overview:** Uses GPT-4o-mini to analyze transcripts and video metadata, generating structured GEO summaries in JSON format.
- **Nodes Involved:** 
  - AI Agent - GEO Analyzer
  - Memory - Conversation Buffer
  - OpenAI Chat Model - GPT-4o-mini
  - Output Parser - Structured JSON
- **Node Details:**
  - **AI Agent - GEO Analyzer**
    - Type: LangChain AI Agent node
    - Receives combined metadata and transcript text
    - Uses a system prompt instructing the AI to produce a structured GEO JSON with goal, execution, outcome, and keywords arrays
    - Has output parser enabled for strict JSON output validation
    - Edge cases: AI hallucination mitigated by system instructions, potential parsing errors if AI output deviates
  - **Memory - Conversation Buffer**
    - Type: LangChain memory buffer node
    - Manages session context keyed by "GEO-session" for conversation continuity
    - Input to AI Agent for context retention
  - **OpenAI Chat Model - GPT-4o-mini**
    - Type: LangChain OpenAI chat model node
    - Model: gpt-4o-mini, chosen for cost-efficiency and structured output capability
    - Connected as the language model for AI Agent
  - **Output Parser - Structured JSON**
    - Type: LangChain output parser node
    - Defines expected JSON schema example for GEO output
    - Parses AI response to ensure valid JSON structure

#### 1.6 Data Formatting & Storage

- **Overview:** Formats AI-generated GEO JSON fields for Notion and creates a new page in a Notion database with video metadata and summaries.
- **Nodes Involved:** 
  - Function - Parse GEO JSON
  - Notion - Create GEO Summary Page
- **Node Details:**
  - **Function - Parse GEO JSON**
    - Type: Function (JavaScript) node
    - Transforms AI JSON output arrays into formatted strings suitable for Notion rich text properties:
      - Joins arrays with bullet points and line breaks
      - Produces comma-separated keywords string
    - Input: AI Agent output JSON
    - Output: Formatted GEO summary fields for Notion
    - Potential errors: Missing or malformed AI output
  - **Notion - Create GEO Summary Page**
    - Type: Notion node
    - Creates a new page in a specified Notion database
    - Database ID must be replaced to match user workspace
    - Maps GEO fields to Notion rich_text properties: Goal, Execution, Outcomes, Keywords
    - Credentials: Notion API integration required
    - Edge cases: API rate limits, invalid database ID, insufficient permissions

#### 1.7 Security & Credential Management

- **Overview:** Lists all necessary credentials and advises on secure handling before sharing the workflow.
- **Nodes Involved:** 
  - Sticky Note - Security
- **Node Details:**
  - **Sticky Note - Security**
    - Informational note highlighting required credentials:
      - YouTube OAuth2
      - OpenAI API key
      - Notion API integration
      - RapidAPI key (must replace placeholder)
    - Warns to remove personal tokens before sharing workflow templates

---

### 3. Summary Table

| Node Name                        | Node Type                                      | Functional Role                     | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                     |
|---------------------------------|------------------------------------------------|-----------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------|
| Schedule Trigger                | Schedule Trigger                               | Periodic workflow trigger         |                                  | YouTube - Fetch Video Details    | See "Sticky Note - Overview" for setup and usage instructions  |
| YouTube - Fetch Video Details  | YouTube API                                    | Fetch video metadata              | Schedule Trigger                 | Set - Prepare Video Metadata     | See "Sticky Note - Fetch" for details on metadata retrieval     |
| Set - Prepare Video Metadata   | Set                                            | Organize video metadata           | YouTube - Fetch Video Details    | Merge - Attach Metadata + Transcript, HTTP - Get YouTube Audio | See "Sticky Note - Fetch"                                         |
| HTTP - Get YouTube Audio       | HTTP Request                                   | Retrieve audio download URL       | Set - Prepare Video Metadata     | HTTP - Download Audio File       | Replace RapidAPI key placeholder (see "Sticky Note - Audio")    |
| HTTP - Download Audio File     | HTTP Request                                   | Download audio file               | HTTP - Get YouTube Audio          | OpenAI - Transcribe Audio (Whisper) | See "Sticky Note - Audio"                                       |
| OpenAI - Transcribe Audio (Whisper) | OpenAI Whisper                             | Transcribe audio to text          | HTTP - Download Audio File        | IF - Skip if No Transcript       | See "Sticky Note - Audio"                                        |
| IF - Skip if No Transcript     | If                                             | Filter empty transcripts          | OpenAI - Transcribe Audio         | Merge - Attach Metadata + Transcript (true branch) |                                                                |
| Merge - Attach Metadata + Transcript | Merge                                     | Combine metadata with transcript  | Set - Prepare Video Metadata, IF - Skip if No Transcript | AI Agent - GEO Analyzer          |                                                                |
| AI Agent - GEO Analyzer        | LangChain AI Agent                             | Generate GEO summary JSON         | Merge - Attach Metadata + Transcript | Function - Parse GEO JSON      | See "Sticky Note - AI" for AI processing details                |
| Memory - Conversation Buffer   | LangChain Memory Buffer                        | Maintain AI session context       |                                 | AI Agent - GEO Analyzer (ai_memory) | See "Sticky Note - AI"                                         |
| OpenAI Chat Model - GPT-4o-mini| LangChain OpenAI Chat Model                    | Language model for AI Agent       |                                 | AI Agent - GEO Analyzer (ai_languageModel) | See "Sticky Note - AI"                                     |
| Output Parser - Structured JSON| LangChain Output Parser                        | Parse AI output to valid JSON     |                                 | AI Agent - GEO Analyzer (ai_outputParser) | See "Sticky Note - AI"                                     |
| Function - Parse GEO JSON      | Function (JavaScript)                          | Format GEO JSON for Notion        | AI Agent - GEO Analyzer           | Notion - Create GEO Summary Page |                                                                |
| Notion - Create GEO Summary Page | Notion API                                   | Store GEO summary in Notion       | Function - Parse GEO JSON         |                                 | See "Sticky Note - Storage" for storage explanation             |
| Sticky Note - Overview         | Sticky Note                                    | Workflow overview & instructions  |                                  |                                 | Covers entire workflow overview and setup                       |
| Sticky Note - Fetch            | Sticky Note                                    | Video metadata explanation        |                                  |                                 | Covers nodes related to fetching video metadata                |
| Sticky Note - Audio            | Sticky Note                                    | Audio download and transcription  |                                  |                                 | Covers audio and transcription nodes                            |
| Sticky Note - AI               | Sticky Note                                    | AI analysis & GEO extraction      |                                  |                                 | Covers AI-related nodes                                         |
| Sticky Note - Storage          | Sticky Note                                    | Data storage in Notion            |                                  |                                 | Covers Notion storage node                                      |
| Sticky Note - Security         | Sticky Note                                    | Credentials & security notes      |                                  |                                 | Covers security and credential requirements                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**
   - Type: Schedule Trigger
   - Set interval to run every hour (or desired frequency)
   - Position as entry node

2. **Add YouTube - Fetch Video Details**
   - Type: YouTube node
   - Resource: Video
   - Operation: Get video details
   - Connect input from Schedule Trigger
   - Configure YouTube OAuth2 credentials
   - Set video ID to be fetched (replace with dynamic input or hardcoded for testing)

3. **Add Set - Prepare Video Metadata**
   - Type: Set node
   - Connect input from YouTube node
   - Assign fields:
     - videoId: `{{$json.id}}`
     - title: `{{$json.snippet.title}}`
     - description: `{{$json.snippet.description}}`
     - publishedAt: `{{$json.snippet.publishedAt}}`
     - thumbnailUrl: `{{$json.snippet.thumbnails.default.url}}`
     - videoUrl: `https://www.youtube.com/watch?v={{$json.id.videoId}}`

4. **Add HTTP - Get YouTube Audio**
   - Type: HTTP Request
   - Connect input from Set node
   - URL: `https://youtube-video-fast-downloader-24-7.p.rapidapi.com/download_audio/{{$json.videoId}}?quality=140`
   - Method: GET
   - Headers:
     - `x-rapidapi-host`: `youtube-video-fast-downloader-24-7.p.rapidapi.com`
     - `x-rapidapi-key`: Replace with your RapidAPI key
     - `Content-Type`: `application/json`
   - Configure to send headers and body as needed

5. **Add HTTP - Download Audio File**
   - Type: HTTP Request
   - Connect input from previous node (HTTP - Get YouTube Audio)
   - URL: Use the `file` property from previous output
   - Response format: File (binary)
   - Download the audio file for transcription

6. **Add OpenAI - Transcribe Audio (Whisper)**
   - Type: OpenAI Whisper node
   - Connect input from HTTP - Download Audio File
   - Credentials: OpenAI API key
   - Operation: Transcribe audio to text

7. **Add IF - Skip if No Transcript**
   - Type: If node
   - Connect input from OpenAI Whisper node
   - Condition: Check if `{{$json.text}}` is not empty (`notEmpty`)
   - True branch continues, False branch ends or stops processing

8. **Add Merge - Attach Metadata + Transcript**
   - Type: Merge node (combine mode)
   - Connect inputs from:
     - Set - Prepare Video Metadata (metadata)
     - IF - Skip if No Transcript (transcript)
   - Combine by position to merge metadata and transcript into one JSON object

9. **Add AI Agent - GEO Analyzer**
   - Type: LangChain AI Agent node
   - Connect input from Merge node
   - Configure system message to instruct AI to analyze video metadata and transcript and produce GEO summary JSON
   - Enable output parser with JSON schema (goal, execution, outcome, keywords)
   - Credentials: OpenAI API key

10. **Add Memory - Conversation Buffer**
    - Type: LangChain Memory Buffer node
    - Set sessionKey to "GEO-session"
    - Connect as AI memory input to AI Agent

11. **Add OpenAI Chat Model - GPT-4o-mini**
    - Type: LangChain OpenAI Chat Model node
    - Model: gpt-4o-mini
    - Connect as Language Model input to AI Agent

12. **Add Output Parser - Structured JSON**
    - Type: LangChain Output Parser node
    - Define JSON schema example for GEO summary
    - Connect as Output Parser input to AI Agent

13. **Add Function - Parse GEO JSON**
    - Type: Function (JavaScript)
    - Connect input from AI Agent output
    - Add JavaScript code to:
      - Format arrays into bullet-point strings for Goals, Execution, Outcomes
      - Convert keywords array into comma-separated string
    - Output formatted fields for Notion

14. **Add Notion - Create GEO Summary Page**
    - Type: Notion node
    - Connect input from Function node
    - Resource: Database Page
    - Database ID: Replace with your Notion database ID
    - Map properties:
      - Goal → rich_text → formatted Goals string
      - Execution → rich_text → formatted Execution string
      - Outcomes → rich_text → formatted Outcomes string
      - Keywords → rich_text → comma-separated keywords string
    - Credentials: Notion API integration

15. **Add Sticky Notes** (optional but recommended for documentation)
    - Add sticky notes near relevant node groups to explain purpose and setup steps:
      - Workflow overview and setup instructions
      - Video metadata fetch details
      - Audio download and transcription notes
      - AI analysis explanation
      - Data storage notes
      - Credential and security reminders

16. **Test Workflow**
    - Replace hardcoded video ID with a test video
    - Run workflow manually
    - Verify transcript generation, AI GEO summary correctness, and Notion page creation
    - Adjust parameters and error handling as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow uses the RapidAPI YouTube Audio Downloader service. Replace the placeholder API key in the HTTP request node before running.                                                                                     | https://rapidapi.com/                                                                                  |
| The AI model GPT-4o-mini is selected for cost-effective structured output but can be replaced with other GPT-4 variants if available.                                                                                           | https://openai.com/                                                                                    |
| Notion database ID must be replaced with your own workspace’s database to store GEO summaries properly.                                                                                                                          | https://developers.notion.com/                                                                         |
| The workflow includes detailed sticky notes for setup instructions, credential requirements, and block explanations, ensuring ease of maintenance and scalability.                                                               | See sticky notes within workflow                                                                        |
| Remove all personal API tokens and credentials before sharing or publishing this workflow template to ensure security and privacy.                                                                                              | Security best practices                                                                                 |
| The GEO summary format (Goal, Execution, Outcome) follows a structured approach to summarizing video content for clarity and SEO optimization.                                                                                   | Workflow design principle                                                                               |
| For advanced users: consider extending the workflow with dynamic playlist processing or multi-video batch handling by modifying the video ID input source and looping.                                                            | Customization tip                                                                                       |
| This workflow is an example of integrating multiple APIs and AI models in n8n for automated content processing, showcasing best practices in error handling, data merging, and structured AI prompt engineering.                   | n8n community and documentation                                                                        |

---

**Disclaimer:**  
The provided text is exclusively sourced from an n8n automated workflow. It fully complies with content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.