🚀 Transform Podcasts into Viral TikTok Clips with Gemini AI & Auto-Posting ✅

https://n8nworkflows.xyz/workflows/---transform-podcasts-into-viral-tiktok-clips-with-gemini-ai---auto-posting---4568


# 🚀 Transform Podcasts into Viral TikTok Clips with Gemini AI & Auto-Posting ✅

---

## 1. Workflow Overview

This workflow automates the transformation of long-form podcast videos into engaging, viral-ready TikTok clips using AI-powered transcription, highlight extraction, video clipping, and auto-posting. It is designed for content creators and marketers aiming to repurpose podcast content efficiently for social media virality.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Audio Download:** Receives podcast and background video URLs, downloads podcast audio.
- **1.2 Audio Transcription & Processing:** Uploads audio to AssemblyAI, transcribes it with word-level timestamps, and structures the transcription data.
- **1.3 Highlight Extraction with Google Gemini AI:** Uses AI to identify and extract key podcast moments suitable for clips.
- **1.4 Clip Generation & Editing:** Generates main and background clips based on highlights, merges and processes clips using an external video editing API.
- **1.5 Clip Preparation for Posting:** Generates engaging TikTok titles using AI, downloads final clips.
- **1.6 Auto-Posting & Scheduling:** Uploads clips to TikTok and manages posting intervals.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Audio Download

**Overview:**  
Receives the URLs for the podcast video and a copyright-free background video via a form trigger, then downloads the podcast audio in stages using a custom external service.

**Nodes Involved:**  
- Send the YouTube Video Links (Form Trigger)  
- Download Audio Stage 1  
- Download Audio Stage 2  
- If4  
- Download Audio Stage 3  

**Node Details:**

- **Send the YouTube Video Links**  
  - Type: Form Trigger  
  - Role: Entry point receiving user input (podcast and background video URLs).  
  - Config: Requires two required fields "YouTube Video URL" and "Copyright Free Background Video".  
  - Outputs to: Download Audio Stage 1.  
  - Edge cases: Missing required fields will block workflow start.

- **Download Audio Stage 1**  
  - Type: HTTP Request  
  - Role: Initiates podcast audio download via an external webhook with podcast URL and auth token.  
  - Config: POST request with JSON body containing URLs and membership token.  
  - Credentials: None required here, but membership token is mandatory for authorization.  
  - Outputs to: Download Audio Stage 2.  
  - Failure: Invalid token or service downtime.

- **Download Audio Stage 2**  
  - Type: HTTP Request  
  - Role: Polls or requests status/job info from the external audio download service using job ID.  
  - Inputs: Job ID from previous node.  
  - Outputs to: If4 node for status check.  
  - Failure: Job not found or timeout.

- **If4**  
  - Type: If Node  
  - Role: Checks if the audio download status equals "finished".  
  - True branch: Download Audio Stage 3.  
  - False branch: Wait3 (retry wait).  
  - Edge case: Stuck jobs or false statuses causing infinite retries.

- **Download Audio Stage 3**  
  - Type: HTTP Request  
  - Role: Downloads the actual audio file as a binary.  
  - Output: Binary audio file for transcription.  
  - Failure: Network issues or invalid URLs.

---

### 1.2 Audio Transcription & Processing

**Overview:**  
Uploads audio to AssemblyAI for transcription, monitors transcription status, extracts word-level timestamps, and structures transcription data for downstream processing.

**Nodes Involved:**  
- Transcribe Podcast Audio  
- Submit Job  
- Wait 1  
- Check Status  
- If  
- Extract Results  
- Structure Transcription  

**Node Details:**

- **Transcribe Podcast Audio**  
  - Type: HTTP Request  
  - Role: Uploads the binary audio to AssemblyAI's upload endpoint.  
  - Credentials: AssemblyAI API key via HTTP Header Auth.  
  - Output: Upload URL for the audio.  
  - Failure: Upload errors, auth failures.

- **Submit Job**  
  - Type: HTTP Request  
  - Role: Submits transcription job with uploaded audio URL to AssemblyAI.  
  - Input: Upload URL from previous node.  
  - Output: Transcription job ID.  
  - Failure: API errors, invalid URL.

- **Wait 1**  
  - Type: Wait Node  
  - Role: Waits 1 minute before checking transcription status to allow processing.  
  - Output: Check Status.

- **Check Status**  
  - Type: HTTP Request  
  - Role: Checks transcription job status via AssemblyAI API using job ID.  
  - Output: Status JSON.  
  - Failure: Network issues, job ID invalid.

- **If**  
  - Type: If Node  
  - Role: Branches based on transcription status.  
  - Condition: status == "completed".  
  - True branch: Extract Results.  
  - False branch: Wait 1 (retry loop).  
  - Edge case: Long transcription times causing repeated waits.

- **Extract Results**  
  - Type: Set Node  
  - Role: Extracts the full transcription text and detailed word-level timestamp array (`words`).  
  - Output: Variables "transcription_text" and "word_details".

- **Structure Transcription**  
  - Type: Code Node (JavaScript)  
  - Role: Converts raw word details into a formatted array with text, index, start, end for each word.  
  - Error handling: Throws error if input data missing or malformed.  
  - Output: JSON stringified transcript array.

---

### 1.3 Highlight Extraction with Google Gemini AI

**Overview:**  
Uses Google Gemini AI to analyze the structured transcript, extract and merge highlight segments that are emotionally impactful or informative, generating a list of clip-worthy segments.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- Podcast Best Moments Extraction  
- Split Out1  

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: AI Language Model (Google Gemini)  
  - Role: Processes the structured transcript JSON string to extract highlights.  
  - Model: "models/gemini-2.0-flash".  
  - Credentials: Google Palm API key.  
  - Failure: API key limits, latency.

- **Structured Output Parser**  
  - Type: AI Output Parser  
  - Role: Parses AI-generated JSON array of highlights with keys: transcript, start_index, end_index.  
  - Output: Structured highlight data.

- **Podcast Best Moments Extraction**  
  - Type: Chain LLM Node (Langchain)  
  - Role: Wraps AI prompt and output parser for highlight extraction.  
  - Retry on fail: Enabled (important for AI API reliability).  
  - Output: Array of highlight segments.

- **Split Out1**  
  - Type: Split Out Node  
  - Role: Splits highlights array into individual items for batch processing downstream.

---

### 1.4 Clip Generation & Editing

**Overview:**  
Generates video clips for each highlight, creates background clips, extracts audio, and merges clips using an external video editing API, preparing clips for posting.

**Nodes Involved:**  
- Loop Over Clips  
- Generating Main Clip  
- If5  
- Clip Length Calculation  
- Generating Background Clip  
- If6  
- Extracting Main Clip  
- Extracting Background Clip  
- Wait4  
- Wait5  
- Extracting Audio from Clips  
- Extracting Audio from Main Clip 2  
- If7  
- Download Clip Audio  
- Audio Transcription  
- Structure Transcription 2  
- Editing Clips  

**Node Details:**

- **Loop Over Clips**  
  - Type: Split In Batches  
  - Role: Iterates over each highlight segment to process individually.  
  - Batch size: default (likely 1).  
  - Output: Each highlight item sequentially.

- **Generating Main Clip**  
  - Type: HTTP Request  
  - Role: Calls external webhook to generate a clip from the podcast URL for highlight start/end seconds.  
  - Input: Podcast URL and highlight start/end seconds.  
  - Output: Job ID for clip generation.

- **If5**  
  - Type: If Node  
  - Role: Checks if main clip generation status is "finished".  
  - True branch: Clip Length Calculation.  
  - False branch: Wait4 (retry).  
  - Edge case: clip generation failures or timeout.

- **Clip Length Calculation**  
  - Type: Code Node  
  - Role: Calculates the clip duration plus 3 seconds buffer based on start/end indices.  
  - Output: interval (duration).

- **Generating Background Clip**  
  - Type: HTTP Request  
  - Role: Generates background clip from provided background video URL for calculated interval.  
  - Input: Background video URL, fixed start (3s), and interval duration.  
  - Output: Job ID for background clip.

- **If6**  
  - Type: If Node  
  - Role: Checks background clip job status "finished".  
  - True branch: Extracting Audio from Clips.  
  - False branch: Wait5 (retry).

- **Extracting Main Clip**  
  - Type: HTTP Request  
  - Role: Extracts the final main clip video using job ID from main clip generation.  
  - Output: Download URL for main clip.

- **Extracting Background Clip**  
  - Type: HTTP Request  
  - Role: Extracts the final background clip video using job ID.  
  - Output: Download URL for background clip.

- **Wait4 / Wait5**  
  - Type: Wait Nodes  
  - Role: Wait 30 seconds before re-checking clip generation status.

- **Extracting Audio from Clips**  
  - Type: HTTP Request  
  - Role: Extracts audio from clips using podcast URL and highlight start/end seconds.  
  - Output: Job ID for audio extraction.

- **Extracting Audio from Main Clip 2**  
  - Type: HTTP Request  
  - Role: Polls audio extraction job status.  
  - Output: Status JSON.

- **If7**  
  - Type: If Node  
  - Role: Checks if audio extraction status is "finished".  
  - True branch: Download Clip Audio.  
  - False branch: Wait6 (retry).

- **Download Clip Audio**  
  - Type: HTTP Request  
  - Role: Downloads audio file for clip.  
  - Output: Binary audio used for next transcription.

- **Audio Transcription**  
  - Type: HTTP Request (OpenAI Whisper)  
  - Role: Transcribes downloaded audio with word-level timestamps.  
  - Credentials: OpenAI API (Whisper).  
  - Output: Detailed transcription with word timestamps.

- **Structure Transcription 2**  
  - Type: Code Node  
  - Role: Groups words into chunks of three, capitalizing first letter, and adds start/end timestamps for clip subtitles.  
  - Output: Chunks array for clip subtitles.

- **Editing Clips**  
  - Type: HTTP Request  
  - Role: Calls external Andynocode API to merge main clip, background clip, and transcripts into a finalized clip.  
  - Credentials: Andynocode API key.  
  - Output: Job ID for final video processing.

---

### 1.5 Clip Preparation for Posting

**Overview:**  
Generates an engaging TikTok title from the podcast transcription using Google Gemini AI, downloads the finalized video clip, and prepares it for posting.

**Nodes Involved:**  
- If1  
- Structure Links  
- Google Gemini Chat Model1  
- Structured Output Parser1  
- Generate Title  
- Download Final Clip  

**Node Details:**

- **If1**  
  - Type: If Node  
  - Role: Checks if video editing job status is "done".  
  - True branch: Structure Links.  
  - False branch: Wait1 (retry).

- **Structure Links**  
  - Type: Set Node  
  - Role: Constructs final video URL from response data.  
  - Output: "final-video-url".

- **Google Gemini Chat Model1**  
  - Type: AI Language Model  
  - Role: Generates a catchy TikTok title and captions based on full transcription text.  
  - Model: "models/gemini-2.0-flash-001".  
  - Credentials: Google Palm API key.

- **Structured Output Parser1**  
  - Type: AI Output Parser  
  - Role: Parses AI-generated JSON with title and captions.

- **Generate Title**  
  - Type: Chain LLM Node  
  - Role: Wraps AI model and parser to generate final title JSON.  
  - Output: Title object.

- **Download Final Clip**  
  - Type: HTTP Request  
  - Role: Downloads final video clip file from constructed URL.  
  - Output: Binary video file for posting.

---

### 1.6 Auto-Posting & Scheduling

**Overview:**  
Uploads the completed clip to TikTok via Upload-Post API and controls posting intervals to manage rate limits and scheduling.

**Nodes Involved:**  
- Post To TikTok  
- Posting Interval  

**Node Details:**

- **Post To TikTok**  
  - Type: HTTP Request  
  - Role: Uploads the final video file along with title and user info to Upload-Post API targeting TikTok platform.  
  - Credentials: Upload-Post API Key (HTTP Header Auth).  
  - Body: Multipart form-data including title, username, platform array, and video binary.  
  - Failure: Upload failures, auth errors, invalid video.

- **Posting Interval**  
  - Type: Wait Node  
  - Role: Waits a defined number of minutes before restarting the batch loop to avoid API rate limits and manage posting frequency.  
  - Output: Loops back to Loop Over Clips.

---

## 3. Summary Table

| Node Name                    | Node Type                            | Functional Role                       | Input Node(s)                    | Output Node(s)                  | Sticky Note                                               |
|------------------------------|------------------------------------|------------------------------------|---------------------------------|--------------------------------|-----------------------------------------------------------|
| Send the YouTube Video Links  | Form Trigger                       | Receives podcast & background URLs | —                               | Download Audio Stage 1          |                                                           |
| Download Audio Stage 1        | HTTP Request                      | Starts audio download via webhook | Send the YouTube Video Links    | Download Audio Stage 2          |                                                           |
| Download Audio Stage 2        | HTTP Request                      | Polls audio download job status    | Download Audio Stage 1          | If4                           |                                                           |
| If4                          | If Node                          | Checks if audio download is finished | Download Audio Stage 2          | Download Audio Stage 3, Wait3   |                                                           |
| Download Audio Stage 3        | HTTP Request                      | Downloads audio file               | If4 (true branch)               | Transcribe Podcast Audio        |                                                           |
| Transcribe Podcast Audio      | HTTP Request (AssemblyAI Upload) | Uploads audio for transcription   | Download Audio Stage 3          | Submit Job                     |                                                           |
| Submit Job                   | HTTP Request                      | Submits transcription job         | Transcribe Podcast Audio        | Wait 1                        |                                                           |
| Wait 1                       | Wait Node                        | Waits for transcription processing | Submit Job, Check Status        | Check Status, Clip Ready?       |                                                           |
| Check Status                 | HTTP Request                     | Checks transcription status       | Wait 1                        | If                            |                                                           |
| If                           | If Node                         | Branches on transcription completion | Check Status                 | Extract Results, Wait 1         |                                                           |
| Extract Results              | Set Node                        | Extracts transcription text & words | If (true branch)              | Structure Transcription         |                                                           |
| Structure Transcription       | Code Node                       | Formats word-level transcription  | Extract Results                | Podcast Best Moments Extraction |                                                           |
| Google Gemini Chat Model      | AI Language Model (Gemini)       | Extracts highlight segments       | Structure Transcription        | Podcast Best Moments Extraction |                                                           |
| Structured Output Parser      | AI Output Parser                | Parses highlights JSON             | Google Gemini Chat Model       | Podcast Best Moments Extraction |                                                           |
| Podcast Best Moments Extraction | Chain LLM (Langchain)            | Wraps AI prompt and parsing       | Structured Output Parser       | Split Out1                     |                                                           |
| Split Out1                  | Split Out Node                  | Splits highlights array to items  | Podcast Best Moments Extraction | Loop Over Clips                |                                                           |
| Loop Over Clips              | Split In Batches                | Iterates over each highlight      | Split Out1                    | Generating Main Clip (per item) |                                                           |
| Generating Main Clip          | HTTP Request                    | Requests main clip generation     | Loop Over Clips               | Extracting Main Clip           |                                                           |
| If5                         | If Node                        | Checks main clip generation finished | Extracting Main Clip           | Clip Length Calculation, Wait4  |                                                           |
| Clip Length Calculation       | Code Node                      | Calculates clip duration          | If5 (true branch)             | Generating Background Clip      |                                                           |
| Generating Background Clip    | HTTP Request                   | Requests background clip generation | Clip Length Calculation       | Extracting Background Clip     |                                                           |
| If6                         | If Node                        | Checks background clip finished   | Extracting Background Clip     | Extracting Audio from Clips, Wait5 |                                                           |
| Extracting Main Clip          | HTTP Request                   | Extracts main clip video          | Generating Main Clip          | If5                          |                                                           |
| Extracting Background Clip    | HTTP Request                   | Extracts background clip video    | Generating Background Clip    | If6                          |                                                           |
| Wait3                       | Wait Node                      | Waits 30 sec before retrying download | If4 (false branch)           | Download Audio Stage 2          |                                                           |
| Wait4                       | Wait Node                      | Waits 30 sec before retrying main clip | If5 (false branch)           | Extracting Main Clip           |                                                           |
| Wait5                       | Wait Node                      | Waits 30 sec before retrying background clip | If6 (false branch)         | Extracting Background Clip     |                                                           |
| Extracting Audio from Clips   | HTTP Request                   | Extracts audio for clip           | If6 (true branch)             | Extracting Audio from Main Clip 2 |                                                           |
| Extracting Audio from Main Clip 2 | HTTP Request               | Polls audio extraction status     | Extracting Audio from Clips   | If7                          |                                                           |
| If7                         | If Node                        | Checks audio extraction finished  | Extracting Audio from Main Clip 2 | Download Clip Audio, Wait6    |                                                           |
| Download Clip Audio           | HTTP Request                   | Downloads audio for clip          | If7 (true branch)             | Audio Transcription            |                                                           |
| Audio Transcription           | HTTP Request (OpenAI Whisper) | Transcribes clip audio            | Download Clip Audio           | Structure Transcription 2       |                                                           |
| Structure Transcription 2     | Code Node                     | Groups words into chunks for subtitles | Audio Transcription          | Editing Clips                  |                                                           |
| Editing Clips                | HTTP Request                   | Merges clips and subtitles        | Structure Transcription 2     | Wait1                        |                                                           |
| Clip Ready?                  | HTTP Request                   | Checks final clip editing status | Editing Clips                | If1                         |                                                           |
| If1                         | If Node                        | Checks if final clip is ready     | Clip Ready?                  | Structure Links, Wait1          |                                                           |
| Structure Links              | Set Node                      | Builds final video URL            | If1 (true branch)            | Generate Title                 |                                                           |
| Google Gemini Chat Model1     | AI Language Model              | Generates TikTok title and captions | Structure Links             | Generate Title                 |                                                           |
| Structured Output Parser1     | AI Output Parser              | Parses title and captions JSON    | Google Gemini Chat Model1    | Generate Title                 |                                                           |
| Generate Title               | Chain LLM (Langchain)          | Wraps AI model and parser to create final titles | Structured Output Parser1   | Download Final Clip           |                                                           |
| Download Final Clip          | HTTP Request                  | Downloads finalized video file     | Generate Title               | Post To TikTok                |                                                           |
| Post To TikTok              | HTTP Request                  | Uploads clip to TikTok via Upload-Post API | Download Final Clip         | Posting Interval              |                                                           |
| Posting Interval            | Wait Node                    | Waits between posts to manage frequency | Post To TikTok             | Loop Over Clips              |                                                           |
| Various Sticky Notes         | Sticky Note                  | Provide section titles and setup guide | —                           | —                            | See Section 5 for full guide and links                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Input Form Trigger**  
   - Node Type: Form Trigger  
   - Configure with two required fields:  
     - "YouTube Video URL" (string)  
     - "Copyright Free Background Video" (string)  
   - Position this as the workflow entry point.

2. **Download Podcast Audio in Stages**  
   - Create HTTP Request "Download Audio Stage 1":  
     - Method: POST  
     - URL: `https://lemolex.app.n8n.cloud/webhook/74c67c3f-3883-40c2-b0ee-d67ecd1ff9fd`  
     - Body: JSON with keys "podcast_url" linked to form input, and "auth-token" (membership key).  
     - Headers: Content-Type: application/json  
   - Connect form trigger output to this node.

   - Create "Download Audio Stage 2":  
     - HTTP Request POST to `https://lemolex.app.n8n.cloud/webhook/40a697b1-58ba-4ce6-9e52-55cb0cfebafa`  
     - Body: JSON with "job-id" from Stage 1 response.  
   - Connect Stage 1 output to Stage 2.

   - Create If Node "If4":  
     - Condition: check if `status == "finished"` in Stage 2 response.  
     - True branch: Download Audio Stage 3.  
     - False branch: Wait 30 seconds (Wait3), then loop back to Stage 2.

   - Download Audio Stage 3:  
     - HTTP Request GET  
     - URL from `download-url` in If4 true branch response.  
     - Configure response to be saved as binary audio.  
   - Connect If4 true branch to this node.

3. **Upload Audio to AssemblyAI and Transcribe**  
   - Create "Transcribe Podcast Audio":  
     - HTTP Request POST to `https://api.assemblyai.com/v2/upload`  
     - Content-Type: binaryData, inputDataFieldName: audio (from previous node)  
     - Credentials: AssemblyAI API key via HTTP Header Auth.  
   - Connect Download Audio Stage 3 output to this node.

   - Create "Submit Job":  
     - HTTP Request POST to `https://api.assemblyai.com/v2/transcript`  
     - JSON Body with audio_url from above.  
     - Credentials: AssemblyAI.  
   - Link Transcribe Podcast Audio to Submit Job.

   - Create Wait Node "Wait 1" with 1-minute delay.

   - Create "Check Status":  
     - HTTP Request GET to `https://api.assemblyai.com/v2/transcript/{{id}}` using job ID from Submit Job.  
     - Credentials: AssemblyAI.  
   - Connect Wait 1 to Check Status.

   - Create If Node "If":  
     - Condition: `status == "completed"`  
     - True branch: Extract Results.  
     - False branch: Wait 1 (loop).  
   - Connect Check Status to If.

   - "Extract Results":  
     - Set Node extracting `text` as "transcription_text" and `words` as "word_details".  
   - Connect If true branch to Extract Results.

   - "Structure Transcription":  
     - Code Node transforming word_details array into objects with text, index, start, end.  
   - Connect Extract Results to Structure Transcription.

4. **Extract Podcast Highlights with Google Gemini**  
   - Create Google Gemini Chat Model node:  
     - Model: "models/gemini-2.0-flash"  
     - Credentials: Google Palm API key.  
     - Input: JSON string from Structure Transcription.  
   - Connect Structure Transcription output.

   - Create Structured Output Parser with JSON schema for highlights array.

   - Create Chain LLM node "Podcast Best Moments Extraction" linking model and parser.

   - Create Split Out Node "Split Out1" to split highlights array.

5. **Generate Clips for Each Highlight**  
   - Create Split In Batches "Loop Over Clips" connected to Split Out1.

   - For each batch item:  

     - HTTP Request "Generating Main Clip": POST to external webhook with podcast URL and highlight start/end seconds.

     - If Node "If5": checks if main clip status is "finished".  
       - True branch:  
         - Code Node "Clip Length Calculation" to calculate clip duration + 3 sec.  
         - HTTP Request "Generating Background Clip" with background video URL and calculated interval.  
       - False branch: Wait 30 sec (Wait4) then retry.

     - HTTP Request nodes "Extracting Main Clip" and "Extracting Background Clip" polling clip extraction status.

     - If Node "If6" checking background clip finished status:  
       - True branch: "Extracting Audio from Clips" to extract audio from clip.  
       - False branch: Wait 30 sec (Wait5).

     - Poll audio extraction status with "Extracting Audio from Main Clip 2".

     - If Node "If7" checks audio extraction finished:  
       - True branch: "Download Clip Audio" to download audio binary.  
       - False branch: Wait 1 minute (Wait6).

     - HTTP Request "Audio Transcription" using OpenAI Whisper with downloaded clip audio.

     - Code Node "Structure Transcription 2" to group words into chunks for subtitles.

     - HTTP Request "Editing Clips" to external API merging clips and adding subtitles.

6. **Prepare Clip for Posting**  
   - HTTP Request "Clip Ready?" polling editing status.

   - If Node "If1" checks final clip finished:  
     - True branch:  
       - Set Node "Structure Links" constructs final video URL.  
       - Google Gemini Chat Model1 generates TikTok title & captions (model "models/gemini-2.0-flash-001").  
       - Structured Output Parser1 parses title JSON.  
       - Chain LLM node "Generate Title" wraps model and parser.  
       - HTTP Request "Download Final Clip" downloads final video file.  
     - False branch: Wait 1 minute (Wait1).

7. **Post to TikTok and Manage Posting Frequency**  
   - HTTP Request "Post To TikTok" uploads video with title, username, platform via Upload-Post API.  
     - Credentials: Upload-Post API Key (HTTP Header Auth).  
     - Body: multipart form-data including video binary and metadata.

   - Wait Node "Posting Interval" delays next post to manage rate limits.

   - Connect Posting Interval back to Loop Over Clips for continuous processing.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| **Workflow Setup Step-by-Step Guide:** Includes detailed instructions for setting up credentials for AssemblyAI, Google AI Studio (Google Gemini), Andynocode, and Upload-Post, along with membership activation for the AI agent. Also explains usage steps for input URLs and membership token setup. Membership costs $29/month. Activation link: [https://lemolex.gumroad.com/l/ypjdr](https://lemolex.gumroad.com/l/ypjdr) | Sticky Note near workflow start; comprehensive credential and usage guide.                                                     |
| Suggested copyright-free background video examples: Minecraft Parkour Gameplay - [https://youtu.be/XBIaqOm0RKQ](https://youtu.be/XBIaqOm0RKQ), GTA 5 No Copyright Gameplay - [https://youtu.be/z121mUPexGc](https://youtu.be/z121mUPexGc)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Usage instructions in setup note.                                                                                               |
| The workflow heavily depends on multiple external APIs (AssemblyAI, Google Gemini, Andynocode, Upload-Post). Ensure API keys are valid, rate limits are respected, and network connectivity is stable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | General integration note.                                                                                                       |
| The AI prompt for highlight extraction enforces strict JSON-only output without markdown or commentary to ensure smooth parsing downstream.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Critical for AI output parsing nodes.                                                                                           |
| Membership key (auth-token) is required for audio download webhooks and must be kept secure and up to date.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Security note in setup guide.                                                                                                   |

---

# Disclaimer

The text provided originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.

---