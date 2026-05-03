Extract Clean Transcripts from Your YouTube Channel Videos using Data API

https://n8nworkflows.xyz/workflows/extract-clean-transcripts-from-your-youtube-channel-videos-using-data-api-11795


# Extract Clean Transcripts from Your YouTube Channel Videos using Data API

---

### 1. Workflow Overview

This workflow, titled **"YouTube Data API Caption Extractor"**, is designed to extract clean, plain-text transcripts from the captions of YouTube videos belonging exclusively to the authenticated user's own YouTube channel. It leverages the YouTube Data API v3 to list available captions, selects captions based on a preferred language or falls back to the first available one, downloads the caption file in VTT format, and processes it to produce a cleaned transcript suitable for AI processing or content generation.

**Target Use Cases:**
- Extracting transcripts for summarization, sentiment analysis, or keyword extraction
- Generating content from videos via clean transcript data
- Batch processing of video captions for archival or AI workflows
- Handling videos without captions gracefully with suggestions for fallback options

---

**Logical Blocks:**

- **1.1 Input Reception & Variable Setup**  
  Accepts inputs `youtubeVideoId` and `preferredLanguage` from an external trigger and initializes variables.

- **1.2 Caption Listing & Validation**  
  Calls YouTube Data API to list available captions for the specified video and checks if captions exist.

- **1.3 Caption Language Selection**  
  Selects the caption ID matching the preferred language or defaults to the first available caption.

- **1.4 Caption Download & Conversion**  
  Downloads the selected caption file (VTT format) and extracts the textual content.

- **1.5 Transcript Cleaning**  
  Processes the raw VTT text to remove timestamps, headers, sound effect tags, duplicate words, and normalizes whitespace, producing a clean transcript.

- **1.6 Error Handling (No Captions)**  
  Handles cases with no captions by returning a structured error message and gracefully stopping workflow execution.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Variable Setup

**Overview:**  
Triggers the workflow with input parameters, then sets internal variables for downstream processing.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Set Variables

---

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point receiving JSON input from an external workflow trigger.  
  - Configuration: Accepts inputs `youtubeVideoId` and `preferredLanguage`.  
  - Input Connections: None (trigger node)  
  - Output Connections: Set Variables  
  - Edge Cases: Missing or malformed input JSON could cause variable assignment issues.

- **Set Variables**  
  - Type: Set Node  
  - Role: Assigns `youtubeVideoId` and `preferredLanguage` from input JSON to workflow variables for consistent referencing.  
  - Configuration: Copies `youtubeVideoId` and `preferredLanguage` from trigger JSON to node variables.  
  - Input Connections: When Executed by Another Workflow  
  - Output Connections: List Captions  
  - Edge Cases: If inputs are undefined, subsequent API calls may fail or return empty results.

---

#### 1.2 Caption Listing & Validation

**Overview:**  
Queries YouTube Data API to list all captions for the video, then checks if any captions are available.

**Nodes Involved:**  
- List Captions  
- IF Has Captions?

---

**Node Details:**

- **List Captions**  
  - Type: HTTP Request  
  - Role: Calls YouTube Data API `captions.list` endpoint to retrieve available captions for the video ID.  
  - Configuration:  
    - URL: `https://www.googleapis.com/youtube/v3/captions?part=snippet&videoId={{youtubeVideoId}}`  
    - Authentication: YouTube OAuth2 credential with scopes including `youtube.captions.read`.  
  - Input Connections: Set Variables  
  - Output Connections: IF Has Captions?  
  - Edge Cases: OAuth token expiry or insufficient scope errors; API quota limits; video ID invalid or not belonging to authenticated channel.

- **IF Has Captions?**  
  - Type: If Node  
  - Role: Checks if the API call returned any captions by verifying if `items.length > 0`.  
  - Configuration: Condition: `items.length > 0` (numeric greater than zero)  
  - Input Connections: List Captions  
  - Output Connections:  
    - TRUE: Caption Language Selector  
    - FALSE: No Captions Fallback  
  - Edge Cases: Empty or malformed API response; unexpected data structure.

---

#### 1.3 Caption Language Selection

**Overview:**  
Selects the caption that matches the preferred language or defaults to the first available caption.

**Nodes Involved:**  
- Caption Language Selector

---

**Node Details:**

- **Caption Language Selector**  
  - Type: Code Node (JavaScript)  
  - Role: Processes the captions list to find the caption ID matching the preferred language, or picks the first available caption as fallback.  
  - Configuration:  
    - Reads captions array from API response  
    - Uses variable `preferredLanguage` set earlier (defaulting to "es" if undefined)  
    - Returns an object containing `captionId`, `language`, `videoId`, `totalCaptions`, and status flags.  
  - Key Expressions: Uses `.find()` to locate preferred language caption; falls back to first item if none matches.  
  - Input Connections: IF Has Captions? (TRUE branch)  
  - Output Connections: Download VTT  
  - Edge Cases: No captions (should not occur here due to IF node); preferred language not found leading to fallback; malformed caption objects.

---

#### 1.4 Caption Download & Conversion

**Overview:**  
Downloads the selected caption file in VTT format and extracts its plain text content from the file.

**Nodes Involved:**  
- Download VTT  
- Caption File Conversion

---

**Node Details:**

- **Download VTT**  
  - Type: HTTP Request  
  - Role: Downloads the caption file by caption ID from YouTube Data API.  
  - Configuration:  
    - URL: `https://www.googleapis.com/youtube/v3/captions/{{captionId}}`  
    - Authentication: Same YouTube OAuth2 credentials  
    - Expected Response: VTT subtitle file (binary/text)  
  - Input Connections: Caption Language Selector  
  - Output Connections: Caption File Conversion  
  - Edge Cases: OAuth failure; caption ID invalid or expired; network errors.

- **Caption File Conversion**  
  - Type: Extract From File  
  - Role: Extracts text content from the downloaded VTT file for processing.  
  - Configuration: Extracts text data and stores it in key `content`.  
  - Input Connections: Download VTT  
  - Output Connections: Clean Transcript  
  - Edge Cases: Corrupted or unexpected file formats; empty files.

---

#### 1.5 Transcript Cleaning

**Overview:**  
Processes raw subtitle text to remove timestamps, WEBVTT headers, sound effect tags, duplicate words, and extra whitespace to produce a clean transcript string.

**Nodes Involved:**  
- Clean Transcript

---

**Node Details:**

- **Clean Transcript**  
  - Type: Code Node (JavaScript)  
  - Role:  
    - Reads the raw subtitle content  
    - Splits into lines and filters out timestamps, headers, empty lines  
    - Removes tags like `[Música]`  
    - Removes repeated adjacent words (optional)  
    - Joins cleaned lines into a normalized single string  
    - Outputs transcript text along with metadata (word/character count, language, videoId, status)  
  - Input Connections: Caption File Conversion  
  - Output Connections: None (final output)  
  - Edge Cases: Unexpected VTT formatting; non-standard subtitle tags; excessive whitespace or special characters.

---

#### 1.6 Error Handling (No Captions)

**Overview:**  
Handles workflow execution when no captions exist by providing a structured error message and stopping execution gracefully.

**Nodes Involved:**  
- No Captions Fallback  
- Stop and Error

---

**Node Details:**

- **No Captions Fallback**  
  - Type: Code Node (JavaScript)  
  - Role: Returns structured JSON indicating no captions found, including videoId, language, error message, suggestion to use Whisper AI fallback, and status.  
  - Input Connections: IF Has Captions? (FALSE branch)  
  - Output Connections: Stop and Error  
  - Edge Cases: None, designed for error handling.

- **Stop and Error**  
  - Type: Stop and Error Node  
  - Role: Stops the workflow execution and raises an error with message `"There's no captions in YouTube Video"`.  
  - Input Connections: No Captions Fallback  
  - Output Connections: None  
  - Edge Cases: Ensures no silent failure; workflow terminates cleanly.

---

### 3. Summary Table

| Node Name                   | Node Type                    | Functional Role                                  | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                            |
|-----------------------------|------------------------------|-------------------------------------------------|--------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger    | Entry trigger receiving input parameters         | None                           | Set Variables                  | ## 1. Input Processing\n• Captures workflow execution trigger\n• Sets youtubeVideoId and preferredLanguage variables    |
| Set Variables               | Set Node                     | Assigns variables from input                       | When Executed by Another Workflow | List Captions                 | ## 1. Input Processing\n• Captures workflow execution trigger\n• Sets youtubeVideoId and preferredLanguage variables    |
| List Captions               | HTTP Request                 | Calls YouTube API to list video captions          | Set Variables                  | IF Has Captions?               | ## 2. API Discovery & Validation\n• Lists all available captions\n• Checks if captions exist\n• Requires OAuth2 scope   |
| IF Has Captions?            | If Node                     | Checks if captions exist                           | List Captions                  | Caption Language Selector (TRUE), No Captions Fallback (FALSE) | ## ❌ Error Handling Path\nWhen no captions available: returns error response and graceful termination                  |
| Caption Language Selector   | Code Node                   | Selects caption by preferred language or fallback | IF Has Captions? (TRUE)        | Download VTT                  | ## ⚠️ Language Priority Logic\n1. Exact match: preferredLanguage\n2. Fallback: first available caption                  |
| Download VTT               | HTTP Request                 | Downloads VTT caption file                         | Caption Language Selector      | Caption File Conversion        | ## 3. Transcript Processing\n• Downloads VTT\n• Extracts text\n• Cleans timestamps, headers, sound tags, duplicates      |
| Caption File Conversion     | Extract From File            | Extracts text content from VTT file                | Download VTT                  | Clean Transcript               | ## 3. Transcript Processing\n• Downloads VTT\n• Extracts text\n• Cleans timestamps, headers, sound tags, duplicates      |
| Clean Transcript            | Code Node                   | Cleans subtitle text to produce plain transcript  | Caption File Conversion        | None                         | ## ✅ Final Output Format\nSuccess response includes videoId, language, text, wordCount, charCount, status, source        |
| No Captions Fallback        | Code Node                   | Provides structured error when no captions exist  | IF Has Captions? (FALSE)       | Stop and Error                | ## ❌ Error Handling Path\nReturns structured error response and suggests Whisper AI fallback                           |
| Stop and Error             | Stop and Error Node          | Stops workflow with error message                  | No Captions Fallback           | None                         | ## ❌ Error Handling Path\nPrevents silent failures, terminates workflow cleanly                                        |
| Sticky Note7               | Sticky Note                 | Detailed workflow overview, use cases, setup notes | None                          | None                         | ## 🚀 Try It Out!\nYouTube Caption Extractor (Your Channel Only)\n⚠️ API Limitation: Your own channel videos only\n---\nUse Cases, How It Works, Setup, Requirements |
| Sticky Note6               | Sticky Note                 | Describes input processing block                    | None                          | None                         | ## 1. Input Processing\nExample input JSON                                                                           |
| Sticky Note                | Sticky Note                 | Describes API discovery and validation block       | None                          | None                         | ## 2. API Discovery & Validation\nYouTube API scopes required                                                        |
| Sticky Note3               | Sticky Note                 | Describes transcript processing block               | None                          | None                         | ## 3. Transcript Processing\nCleaning details                                                                        |
| Sticky Note4               | Sticky Note                 | Describes final output format                         | None                          | None                         | ## ✅ Final Output Format\nSuccess response JSON example                                                             |
| Sticky Note1               | Sticky Note                 | Describes language priority logic in caption selection | None                          | None                         | ## ⚠️ Language Priority Logic\nExact match or fallback                                                                |
| Sticky Note2               | Sticky Note                 | Describes error handling path                         | None                          | None                         | ## ❌ Error Handling Path\nStructured error and graceful termination                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "When Executed by Another Workflow"**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: Define parameters `youtubeVideoId` (string) and `preferredLanguage` (string).  
   - Position: Start of workflow.

2. **Create "Set Variables" Node**  
   - Type: Set Node  
   - Assign variables:  
     - `youtubeVideoId` = `{{$json.youtubeVideoId}}`  
     - `preferredLanguage` = `{{$json.preferredLanguage}}`  
   - Connect trigger output → Set Variables input.

3. **Create "List Captions" Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/captions?part=snippet&videoId={{ $json.youtubeVideoId }}`  
   - Authentication: Use OAuth2 credential with YouTube scopes including `youtube.captions.read` and `youtube.force-ssl`.  
   - Connect Set Variables output → List Captions input.

4. **Create "IF Has Captions?" Node**  
   - Type: If Node  
   - Condition: Check if the length of `items` array in response is greater than 0 (`{{$json.items.length}} > 0`).  
   - Connect List Captions output → IF Has Captions input.

5. **Create "Caption Language Selector" Node**  
   - Type: Code Node  
   - JavaScript code:  
     - Extract captions array from input.  
     - Retrieve preferredLanguage variable.  
     - Search for caption matching preferredLanguage; if none, fallback to first caption.  
     - Return object with `captionId`, `language`, `videoId`, `totalCaptions`, and `status`.  
   - Connect IF Has Captions TRUE output → Caption Language Selector input.

6. **Create "Download VTT" Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/captions/{{ $json.captionId }}`  
   - Authentication: Same YouTube OAuth2 credential as before.  
   - Connect Caption Language Selector output → Download VTT input.

7. **Create "Caption File Conversion" Node**  
   - Type: Extract From File  
   - Operation: Text extraction  
   - Destination key: `content` (stores subtitle text)  
   - Connect Download VTT output → Caption File Conversion input.

8. **Create "Clean Transcript" Node**  
   - Type: Code Node  
   - JavaScript code:  
     - Acquire subtitle content from `Caption File Conversion`.  
     - Split text by newlines and remove: timestamps, WEBVTT headers, empty lines, tags like `[Música]`, repeated adjacent words, extra whitespace.  
     - Join cleaned lines into one normalized string.  
     - Output JSON with keys: `videoId`, `language`, `text`, `wordCount`, `charCount`, `status` ("success"), and `source` ("youtube_captions").  
   - Connect Caption File Conversion output → Clean Transcript input.

9. **Create "No Captions Fallback" Node**  
   - Type: Code Node  
   - JavaScript code:  
     - Return structured JSON with `videoId`, `language`, error message `"No captions available for this video"`, suggestion `"Use Whisper AI fallback"`, status `"no_captions"`, and source `"youtube_api"`.  
   - Connect IF Has Captions FALSE output → No Captions Fallback input.

10. **Create "Stop and Error" Node**  
    - Type: Stop and Error Node  
    - Error Message: `"There's no captions in YouTube Video"`  
    - Connect No Captions Fallback output → Stop and Error input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| ## 🚀 Try It Out! YouTube Caption Extractor (Your Channel Only) - Extracts transcripts from your own YouTube channel videos only. API limitation: does not work for external/public videos. Use cases include AI summarization, keyword extraction, and batch processing. Requires YouTube OAuth2 with `youtube.captions.read` scope. Setup instructions included in workflow sticky note. | Workflow overview sticky note                                                    |
| API credentials must be updated in the **List Captions** and **Download VTT** nodes to your own YouTube OAuth2 credentials with appropriate scopes.                                                                                                                                                                                                                                   | Credential setup note in sticky notes                                            |
| The workflow can be triggered as a sub-workflow using JSON payload: `{"youtubeVideoId": "YOUR_VIDEO_ID", "preferredLanguage": "es"}`. Alternatively, replace the trigger node with a Webhook or Form trigger as needed.                                                                                                                                                                  | Usage note in workflow sticky notes                                              |
| Error handling ensures graceful termination with structured response and suggestion to fallback to Whisper AI for transcription if captions are unavailable. Prevents silent failures.                                                                                                                                                                                               | Error handling sticky note                                                       |
| For community support or questions, visit the official [n8n Forum](https://community.n8n.io/).                                                                                                                                                                                                                                                                                        | Support link                                                                      |

---

**Disclaimer:** The provided text is exclusively from an automation workflow created with n8n integration tool. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---