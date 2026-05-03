Transform Podcasts into Viral Social Media Clips with Gemini AI and Multi-Platform Posting

https://n8nworkflows.xyz/workflows/transform-podcasts-into-viral-social-media-clips-with-gemini-ai-and-multi-platform-posting-11505


# Transform Podcasts into Viral Social Media Clips with Gemini AI and Multi-Platform Posting

### 1. Workflow Overview

This workflow automates the transformation of long-form podcast videos into viral social media clips, primarily targeting TikTok. It takes YouTube podcast URLs and copyright-free background video URLs as input, extracts audio, transcribes it, identifies highlights, generates clips with engaging titles, and posts them to TikTok automatically.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception and Audio Downloading:** Receives user inputs (podcast and background video URLs), downloads the audio in multiple stages, and waits for processing completion.
- **1.2 Podcast Audio Transcription and Processing:** Uploads audio to AssemblyAI for transcription, polls for completion, and structures the transcript data.
- **1.3 Highlight Extraction:** Uses Google Gemini AI to analyze the transcript and extract key highlight segments suitable for short clips.
- **1.4 Clip Generation and Editing:** Generates video clips from highlights using external APIs, edits clips by merging with background videos, and prepares them for posting.
- **1.5 Title Generation and Final Clip Preparation:** Uses Google Gemini AI to create compelling TikTok titles and captions, downloads the final video clip.
- **1.6 Posting and Scheduling:** Uploads the prepared clips to TikTok and manages posting intervals.
- **1.7 Workflow Setup and Instructions:** Contains configuration notes, sticky notes with usage instructions and setup guides.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Audio Downloading

- **Overview:**  
  Collects user input URLs for the podcast video and background video, initiates multi-stage audio downloading, and waits for download completion.

- **Nodes Involved:**  
  - `Send the YouTube Video Links` (Form Trigger)  
  - `Download Audio Stage 1` (HTTP Request)  
  - `Download Audio Stage 2` (HTTP Request)  
  - `If4` (If)  
  - `Download Audio Stage 3` (HTTP Request)  
  - `Wait3` (Wait)  
  - `If` (If)  
  - `Wait 1` (Wait)  

- **Node Details:**  
  - **Send the YouTube Video Links:**  
    - Type: Form Trigger  
    - Role: Receives YouTube podcast and background video URLs from user input form.  
    - Config: Two required fields: "YouTube Video URL" and "Copyright Free Background Video".  
    - Outputs to `Download Audio Stage 1`.  
    - Edge Cases: Missing or invalid URLs, user input errors.

  - **Download Audio Stage 1:**  
    - Type: HTTP Request  
    - Role: Starts audio extraction process via external webhook (lemolex.app).  
    - Config: POST with JSON body including podcast URL and authorization token (membership key).  
    - Credentials: None (token in body).  
    - Outputs to `Download Audio Stage 2`.  
    - Edge Cases: API auth errors, network timeouts, invalid token.

  - **Download Audio Stage 2:**  
    - Type: HTTP Request  
    - Role: Polls status of audio extraction job started in Stage 1.  
    - Config: POST with job-id from previous node's response.  
    - Outputs to `If4`.  
    - Edge Cases: Job failure, invalid job-id, timeouts.

  - **If4:**  
    - Type: If  
    - Role: Checks if audio extraction job status is "finished".  
    - Outputs: If yes, proceeds to `Download Audio Stage 3`; else waits 30 seconds (`Wait3`) and retries.  
    - Edge Cases: Stuck jobs, indefinite waiting.

  - **Download Audio Stage 3:**  
    - Type: HTTP Request  
    - Role: Downloads extracted audio file from URL provided by `If4`.  
    - Config: Response configured to file download.  
    - Outputs to `Transcribe Podcast Audio` node in next block.  
    - Edge Cases: Download failures, file corruption.

  - **Wait3:**  
    - Type: Wait  
    - Role: Waits 30 seconds before retrying status check.  
    - Edge Cases: Excessive wait time, retries limit not enforced.

  - **If:**  
    - Type: If  
    - Role: Used later in transcription status polling (see block 2.2).  

  - **Wait 1:**  
    - Type: Wait  
    - Role: Used later in transcription status polling (see block 2.2).

---

#### 2.2 Podcast Audio Transcription and Processing

- **Overview:**  
  Uploads downloaded audio to AssemblyAI for transcription, polls for transcription completion, extracts transcription text and word timing details, and structures transcript data for downstream use.

- **Nodes Involved:**  
  - `Transcribe Podcast Audio` (HTTP Request)  
  - `Submit Job` (HTTP Request)  
  - `Check Status` (HTTP Request)  
  - `If` (If)  
  - `Wait 1` (Wait)  
  - `Extract Results` (Set)  
  - `Structure Transcription` (Code)  
  - `Podcast Best Moments Extraction` (Langchain LLM Chain)  
  - `Split Out1` (Split Out)  

- **Node Details:**  
  - **Transcribe Podcast Audio:**  
    - Type: HTTP Request  
    - Role: Uploads audio binary data to AssemblyAI’s upload endpoint.  
    - Credentials: AssemblyAI HTTP Header Auth.  
    - Outputs to `Submit Job`.  
    - Edge Cases: Upload failures, file size limits.

  - **Submit Job:**  
    - Type: HTTP Request  
    - Role: Submits transcription job with uploaded audio URL to AssemblyAI.  
    - Credentials: AssemblyAI HTTP Header Auth.  
    - Outputs to `Wait 1`.  
    - Edge Cases: API rate limits, auth errors.

  - **Check Status:**  
    - Type: HTTP Request  
    - Role: Polls transcription job status using job ID returned by `Submit Job`.  
    - Credentials: AssemblyAI HTTP Header Auth.  
    - Outputs to `If`.  
    - Edge Cases: Incomplete transcription, network errors.

  - **If:**  
    - Type: If  
    - Role: Checks if transcription status is "completed".  
    - If true, outputs to `Extract Results`; else waits 1 minute (`Wait 1`) and retries.  
    - Edge Cases: Stuck or failed transcription jobs.

  - **Wait 1:**  
    - Type: Wait  
    - Role: Waits 1 minute between status polls.  

  - **Extract Results:**  
    - Type: Set  
    - Role: Extracts transcription text and detailed word timing from transcription response.  
    - Assigns variables: `transcription_text` and `word_details` array.  
    - Outputs to `Structure Transcription`.  

  - **Structure Transcription:**  
    - Type: Code  
    - Role: Formats detailed word timing data into a structured JSON array with fields: text, index, start time, end time.  
    - Throws error if data missing or incorrect format.  
    - Outputs structured JSON string for highlight extraction.  
    - Edge Cases: Data format inconsistencies.

  - **Podcast Best Moments Extraction:**  
    - Type: Langchain LLM Chain (Google Gemini AI)  
    - Role: Analyzes structured transcript JSON to extract coherent highlight segments for clip generation.  
    - Constraints: Total highlights ≤ 600 seconds, each 30-120 seconds, no overlap.  
    - Outputs plain JSON array with highlight segments (transcript text, start and end word indices).  
    - Edge Cases: AI output format errors, prompt failures.  
    - Connected to `Split Out1`.

  - **Split Out1:**  
    - Type: Split Out  
    - Role: Splits the highlight segments array into individual items for looping in clip generation.  

---

#### 2.3 Clip Generation and Editing

- **Overview:**  
  Iterates over extracted highlight segments to generate video clips, merges them with background videos, extracts audio, and prepares clips for final editing.

- **Nodes Involved:**  
  - `Loop Over Clips` (Split In Batches)  
  - `Generating Main Clip` (HTTP Request)  
  - `If5` (If)  
  - `Clip Length Calculation` (Code)  
  - `Generating Background Clip` (HTTP Request)  
  - `Extracting Main Clip` (HTTP Request)  
  - `Extracting Background Clip` (HTTP Request)  
  - `If6` (If)  
  - `Wait4` (Wait)  
  - `Wait5` (Wait)  
  - `Extracting Audio from Clips` (HTTP Request)  
  - `Extracting Audio from Main Clip 2` (HTTP Request)  
  - `If7` (If)  
  - `Wait6` (Wait)  
  - `Download Clip Audio` (HTTP Request)  
  - `Audio Transcription` (HTTP Request)  
  - `Structure Transcription 2` (Code)  
  - `Editing Clips` (HTTP Request)  

- **Node Details:**  
  - **Loop Over Clips:**  
    - Type: Split In Batches  
    - Role: Iterates over individual highlight segments to process clips sequentially or concurrently.  
    - Outputs to `Generating Main Clip`.  

  - **Generating Main Clip:**  
    - Type: HTTP Request  
    - Role: Calls external webhook to generate clipped podcast video segment based on start and end seconds.  
    - Inputs: Podcast URL and segment start/end from current batch item.  
    - Outputs to `Extracting Main Clip`.  
    - Edge Cases: API failures, invalid timecodes.

  - **If5:**  
    - Type: If  
    - Role: Checks if main clip generation status is "finished".  
    - If yes, proceeds to calculate clip length; else waits 30 seconds (`Wait4`) and retries.  

  - **Clip Length Calculation:**  
    - Type: Code  
    - Role: Calculates duration of current clip segment plus 3 seconds buffer.  
    - Outputs interval for background clip generation.  

  - **Generating Background Clip:**  
    - Type: HTTP Request  
    - Role: Generates background video clip matching interval length from user-provided background video URL.  
    - Inputs: Background video URL, fixed start at 3 seconds, interval calculated previously.  
    - Outputs to `Extracting Background Clip`.  

  - **Extracting Main Clip:**  
    - Type: HTTP Request  
    - Role: Checks extraction progress/status of main clip generation job.  
    - Outputs to `If5`.  

  - **Extracting Background Clip:**  
    - Type: HTTP Request  
    - Role: Checks extraction progress/status of background clip generation job.  
    - Outputs to `If6`.  

  - **If6:**  
    - Type: If  
    - Role: Checks if background clip status is "finished".  
    - If yes, proceeds; else waits 1 minute (`Wait5`) and retries.  

  - **Wait4:**  
    - Type: Wait  
    - Role: Waits 30 seconds before retrying main clip extraction status.  

  - **Wait5:**  
    - Type: Wait  
    - Role: Waits 1 minute before retrying background clip extraction status.  

  - **Extracting Audio from Clips:**  
    - Type: HTTP Request  
    - Role: Extracts audio track from generated video clip segment.  
    - Inputs: Podcast URL and start/end times from current batch item.  
    - Outputs to `Extracting Audio from Main Clip 2`.  

  - **Extracting Audio from Main Clip 2:**  
    - Type: HTTP Request  
    - Role: Checks extraction status of audio from clip.  
    - Outputs to `If7`.  

  - **If7:**  
    - Type: If  
    - Role: Checks if audio extraction status is "finished".  
    - If yes, proceeds to download clip audio; else waits 1 minute (`Wait6`) and retries.  

  - **Wait6:**  
    - Type: Wait  
    - Role: Waits 1 minute before retrying audio extraction status.  

  - **Download Clip Audio:**  
    - Type: HTTP Request  
    - Role: Downloads audio file extracted from clip for transcription.  
    - Response configured as binary file.  
    - Outputs to `Audio Transcription` node for further processing.  

  - **Audio Transcription:**  
    - Type: HTTP Request  
    - Role: Uploads downloaded clip audio to OpenAI Whisper API for detailed transcription with word-level timestamps.  
    - Credentials: OpenAI API.  
    - Outputs to `Structure Transcription 2`.  

  - **Structure Transcription 2:**  
    - Type: Code  
    - Role: Groups transcribed words into chunks of 3 words, capitalizes first letter of each word, and prepares transcript chunks with start/end times for clip captions.  
    - Outputs to `Editing Clips`.  

  - **Editing Clips:**  
    - Type: HTTP Request  
    - Role: Sends clip URLs (main clip, background clip) and transcript chunks to external video-generation API (Andynocode) to generate final edited clip.  
    - Credentials: Andynocode API key.  
    - Outputs to `Wait1` for clip processing completion check.  

---

#### 2.4 Title Generation and Final Clip Preparation

- **Overview:**  
  After clip editing, generates a viral TikTok title and captions using AI, downloads the final clip, and prepares it for posting.

- **Nodes Involved:**  
  - `If1` (If)  
  - `Structure Links` (Set)  
  - `Google Gemini Chat Model1` (Langchain AI Language Model)  
  - `Structured Output Parser1` (Langchain Output Parser)  
  - `Generate Title` (Langchain LLM Chain)  
  - `Download Final Clip` (HTTP Request)  
  - `Post To TikTok` (HTTP Request)  
  - `Posting Interval` (Wait)  

- **Node Details:**  
  - **If1:**  
    - Type: If  
    - Role: Checks if clip editing job status is "done".  
    - If yes, proceeds to `Structure Links`; else waits 1 minute (`Wait1`) and retries.  

  - **Structure Links:**  
    - Type: Set  
    - Role: Constructs final video URL from editing job response for downloading.  

  - **Google Gemini Chat Model1:**  
    - Type: Langchain AI Language Model (Google Gemini)  
    - Role: Generates viral TikTok title and captions based on transcript text.  
    - Model: "models/gemini-2.0-flash-001".  
    - Outputs to `Structured Output Parser1`.  

  - **Structured Output Parser1:**  
    - Type: Langchain Output Parser  
    - Role: Parses AI response JSON to extract title and captions.  

  - **Generate Title:**  
    - Type: Langchain LLM Chain  
    - Role: Wrapper node integrating Google Gemini AI for title generation with prompt and parser.  

  - **Download Final Clip:**  
    - Type: HTTP Request  
    - Role: Downloads final edited video clip file for posting.  
    - Configured to receive file response.  
    - Outputs to `Post To TikTok`.  

  - **Post To TikTok:**  
    - Type: HTTP Request  
    - Role: Uploads final clip with title and user info to TikTok via Upload-Post API.  
    - Credentials: Upload-Post API key (HTTP Header Auth).  
    - Body includes multipart form data with video file and title.  
    - Outputs to `Posting Interval`.  

  - **Posting Interval:**  
    - Type: Wait  
    - Role: Waits defined minutes before processing next clip batch or reposting.  
    - Webhook ID defined for external triggering if needed.  

---

#### 2.5 Workflow Setup and Instructions

- **Overview:**  
  Contains sticky notes with setup instructions, workflow usage, and membership information.

- **Nodes Involved:**  
  - Multiple Sticky Note nodes (`Sticky Note`, `Sticky Note1`, ..., `Sticky Note9`)

- **Node Details:**  
  - These nodes provide guidance on:  
    - Signing up and configuring credentials for AssemblyAI, Google AI Studio, Andynocode, Upload-Post, OpenAI.  
    - Membership activation steps with payment link and key usage.  
    - Workflow usage instructions including example background videos.  
    - General workflow overview and technology cost disclaimer.  
  - Sticky notes contain useful external links to services and setup guidance.  
  - Important to read before deploying and operating the workflow.  

---

### 3. Summary Table

| Node Name                    | Node Type                                   | Functional Role                         | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                      |
|------------------------------|---------------------------------------------|---------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Send the YouTube Video Links | Form Trigger                               | Input reception for podcast & background URLs | None                            | Download Audio Stage 1          |                                                                                                 |
| Download Audio Stage 1        | HTTP Request                              | Start audio extraction job            | Send the YouTube Video Links    | Download Audio Stage 2          |                                                                                                 |
| Download Audio Stage 2        | HTTP Request                              | Poll audio extraction status          | Download Audio Stage 1          | If4                           |                                                                                                 |
| If4                          | If                                         | Check if audio extraction finished     | Download Audio Stage 2          | Download Audio Stage 3, Wait3  |                                                                                                 |
| Download Audio Stage 3        | HTTP Request                              | Download extracted audio file          | If4                           | Transcribe Podcast Audio       |                                                                                                 |
| Wait3                        | Wait                                       | Wait 30s before retrying audio status  | If4                           | Download Audio Stage 2          |                                                                                                 |
| Transcribe Podcast Audio      | HTTP Request                              | Upload audio to AssemblyAI             | Download Audio Stage 3          | Submit Job                    |                                                                                                 |
| Submit Job                   | HTTP Request                              | Submit transcription job               | Transcribe Podcast Audio        | Wait 1                       |                                                                                                 |
| Check Status                 | HTTP Request                              | Poll transcription job status          | Wait 1                       | If                           |                                                                                                 |
| If                           | If                                         | Check if transcription completed       | Check Status                  | Extract Results, Wait 1        |                                                                                                 |
| Wait 1                      | Wait                                       | Wait 1 minute before retrying transcription status | If                           | Check Status                  |                                                                                                 |
| Extract Results              | Set                                        | Extract transcription text & words     | If                            | Structure Transcription       |                                                                                                 |
| Structure Transcription       | Code                                       | Format transcription data              | Extract Results              | Podcast Best Moments Extraction |                                                                                                 |
| Podcast Best Moments Extraction | Langchain LLM Chain (Google Gemini)      | Extract podcast highlights             | Structure Transcription       | Split Out1                   |                                                                                                 |
| Split Out1                  | Split Out                                  | Split highlights array for looping     | Podcast Best Moments Extraction | Loop Over Clips              |                                                                                                 |
| Loop Over Clips             | Split In Batches                           | Iterate over highlight segments        | Split Out1                   | Generating Main Clip          |                                                                                                 |
| Generating Main Clip         | HTTP Request                              | Generate podcast clip segment           | Loop Over Clips              | Extracting Main Clip          |                                                                                                 |
| If5                         | If                                         | Check main clip generation status      | Extracting Main Clip          | Clip Length Calculation, Wait4 |                                                                                                 |
| Clip Length Calculation      | Code                                       | Calculate clip duration + buffer       | If5                         | Generating Background Clip    |                                                                                                 |
| Generating Background Clip   | HTTP Request                              | Generate background video clip          | Clip Length Calculation      | Extracting Background Clip    |                                                                                                 |
| Extracting Main Clip         | HTTP Request                              | Check main clip extraction status       | Generating Main Clip         | If5                         |                                                                                                 |
| Extracting Background Clip   | HTTP Request                              | Check background clip extraction status | Generating Background Clip   | If6                         |                                                                                                 |
| If6                         | If                                         | Check if background clip finished       | Extracting Background Clip   | Extracting Audio from Clips, Wait5 |                                                                                                 |
| Wait4                       | Wait                                       | Wait 30s before retrying main clip status | If5                         | Extracting Main Clip          |                                                                                                 |
| Wait5                       | Wait                                       | Wait 1 min before retrying background clip status | If6                         | Extracting Background Clip    |                                                                                                 |
| Extracting Audio from Clips  | HTTP Request                              | Extract audio from generated clip       | If6                         | Extracting Audio from Main Clip 2 |                                                                                                 |
| Extracting Audio from Main Clip 2 | HTTP Request                          | Check audio extraction job status       | Extracting Audio from Clips  | If7                         |                                                                                                 |
| If7                         | If                                         | Check if audio extraction finished      | Extracting Audio from Main Clip 2 | Download Clip Audio, Wait6  |                                                                                                 |
| Wait6                       | Wait                                       | Wait 1 min before retrying audio extraction | If7                         | Extracting Audio from Main Clip 2 |                                                                                                 |
| Download Clip Audio          | HTTP Request                              | Download extracted audio for clip       | If7                         | Audio Transcription           |                                                                                                 |
| Audio Transcription          | HTTP Request                              | Transcribe clip audio with OpenAI Whisper | Download Clip Audio          | Structure Transcription 2     |                                                                                                 |
| Structure Transcription 2    | Code                                       | Format transcribed words into chunks    | Audio Transcription          | Editing Clips                |                                                                                                 |
| Editing Clips               | HTTP Request                              | Generate final edited clip video         | Structure Transcription 2     | Wait1                       |                                                                                                 |
| Wait1                       | Wait                                       | Wait before checking clip editing status | Editing Clips               | Clip Ready?                 |                                                                                                 |
| Clip Ready?                 | HTTP Request                              | Check if edited clip is ready            | Wait1                       | If1                         |                                                                                                 |
| If1                         | If                                         | Check clip editing job status            | Clip Ready?                 | Structure Links, Wait1       |                                                                                                 |
| Structure Links             | Set                                        | Format final video URL                    | If1                         | Generate Title              |                                                                                                 |
| Google Gemini Chat Model1   | Langchain AI Language Model (Google Gemini) | Generate viral TikTok title & captions  | Structure Links             | Structured Output Parser1    |                                                                                                 |
| Structured Output Parser1   | Langchain Output Parser                    | Parse AI-generated title & captions      | Google Gemini Chat Model1   | Generate Title              |                                                                                                 |
| Generate Title              | Langchain LLM Chain                       | Wrapper for Google Gemini AI title generation | Structured Output Parser1 | Download Final Clip         |                                                                                                 |
| Download Final Clip         | HTTP Request                              | Download final processed video clip      | Generate Title              | Post To TikTok              |                                                                                                 |
| Post To TikTok             | HTTP Request                              | Upload final video clip to TikTok         | Download Final Clip         | Posting Interval            |                                                                                                 |
| Posting Interval           | Wait                                       | Wait between posting clips                | Post To TikTok              | Loop Over Clips             |                                                                                                 |
| Sticky Note nodes (multiple) | Sticky Note                                | Setup instructions, usage guides          | None                        | None                        | Contains external links and detailed setup instructions and usage notes (see notes section). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node "Send the YouTube Video Links":**  
   - Type: Form Trigger  
   - Configure two required fields: "YouTube Video URL" and "Copyright Free Background Video".  
   - Position: Input start.  

2. **Add HTTP Request "Download Audio Stage 1":**  
   - POST to `https://lemolex.app.n8n.cloud/webhook/74c67c3f-3883-40c2-b0ee-d67ecd1ff9fd`  
   - Body: JSON with `podcast_url` from form input and `auth-token` membership key (setup guide required).  
   - Headers: `Content-Type: application/json`.  
   - Connect from Form Trigger.  

3. **Add HTTP Request "Download Audio Stage 2":**  
   - POST to `https://lemolex.app.n8n.cloud/webhook/40a6b1-58ba-4ce6-9e52-55cb0cfebafa`  
   - Body: JSON with `job-id` from previous node's response `id`.  
   - Connect from "Download Audio Stage 1".  

4. **Add If Node "If4":**  
   - Condition: `$json.status` equals `"finished"`.  
   - True: Connect to "Download Audio Stage 3".  
   - False: Connect to "Wait3".  

5. **Add HTTP Request "Download Audio Stage 3":**  
   - URL from `$json["download-url"]` of `If4` node.  
   - Response: File download configured.  
   - Connect True branch of "If4".  

6. **Add Wait Node "Wait3":**  
   - Wait 30 seconds.  
   - Connect False branch of "If4" back to "Download Audio Stage 2".  

7. **Add HTTP Request "Transcribe Podcast Audio":**  
   - POST binary audio data to AssemblyAI upload endpoint.  
   - Credentials: AssemblyAI HTTP Header Auth (setup required).  
   - Connect from "Download Audio Stage 3".  

8. **Add HTTP Request "Submit Job":**  
   - POST to AssemblyAI transcript endpoint with JSON body containing audio_url from previous node.  
   - Credentials: AssemblyAI HTTP Header Auth.  
   - Connect from "Transcribe Podcast Audio".  

9. **Add Wait Node "Wait 1":**  
   - Wait 1 minute.  
   - Connect from "Submit Job".  

10. **Add HTTP Request "Check Status":**  
    - GET transcription status from AssemblyAI using job ID from "Submit Job".  
    - Credentials: AssemblyAI HTTP Header Auth.  
    - Connect from "Wait 1".  

11. **Add If Node "If":**  
    - Condition: `$json.status` equals `"completed"`.  
    - True: Connect to "Extract Results".  
    - False: Connect to "Wait 1".  

12. **Add Set Node "Extract Results":**  
    - Assign variables:  
      - `transcription_text` = `$json.text`  
      - `word_details` = `$json.words` (array)  
    - Connect from True branch of "If".  

13. **Add Code Node "Structure Transcription":**  
    - JavaScript code to map `word_details` into array of objects with `text`, `index`, `start`, `end`.  
    - Throws error if data missing or invalid.  
    - Connect from "Extract Results".  

14. **Add Langchain Chain Node "Podcast Best Moments Extraction":**  
    - Use Google Gemini AI model `models/gemini-2.0-flash`.  
    - Prompt instructions to extract highlight segments with constraints (max 600s total, 30-120s each, no overlap).  
    - Input: JSON string from "Structure Transcription".  
    - Output parser: Structured Output Parser expecting JSON array.  
    - Connect from "Structure Transcription".  

15. **Add Structured Output Parser Node:**  
    - Define JSON schema as array of objects with keys: transcript, start_index, end_index.  
    - Connect AI output to this node.  

16. **Add Split Out Node "Split Out1":**  
    - Splits array of highlights into individual items.  
    - Connect from "Podcast Best Moments Extraction".  

17. **Add Split In Batches Node "Loop Over Clips":**  
    - Handles iteration over each highlight segment.  
    - Connect from "Split Out1".  

18. **Add HTTP Request "Generating Main Clip":**  
    - POST to external webhook to generate clip with parameters: podcast_url, startSeconds, endSeconds from current batch item.  
    - Connect from "Loop Over Clips".  

19. **Add If Node "If5":**  
    - Check if clip generation status is "finished".  
    - True: Proceed to "Clip Length Calculation".  
    - False: Wait 30 seconds ("Wait4") and retry.  

20. **Add Code Node "Clip Length Calculation":**  
    - Calculate clip duration plus 3 seconds buffer from start/end indices.  
    - Connect from "If5" True branch.  

21. **Add HTTP Request "Generating Background Clip":**  
    - POST to generate background clip using background video URL, startSeconds=3, endSeconds=interval from previous node.  
    - Connect from "Clip Length Calculation".  

22. **Add HTTP Request "Extracting Main Clip":**  
    - Check main clip extraction status using job ID from "Generating Main Clip".  
    - Connect from "Generating Main Clip".  

23. **Add HTTP Request "Extracting Background Clip":**  
    - Check background clip extraction status using job ID from "Generating Background Clip".  
    - Connect from "Generating Background Clip".  

24. **Add If Node "If6":**  
    - Check if background clip extraction status is "finished".  
    - True: Proceed to "Extracting Audio from Clips".  
    - False: Wait 1 minute ("Wait5") and retry.  

25. **Add Wait Nodes "Wait4" and "Wait5":**  
    - Wait 30 seconds and 1 minute respectively for retry delays.  

26. **Add HTTP Request "Extracting Audio from Clips":**  
    - POST to extract audio from clip with podcast URL and start/end seconds from current batch item.  
    - Connect from "If6" True branch.  

27. **Add HTTP Request "Extracting Audio from Main Clip 2":**  
    - Check audio extraction job status from previous node's job ID.  
    - Connect from "Extracting Audio from Clips".  

28. **Add If Node "If7":**  
    - Check if audio extraction status is "finished".  
    - True: Proceed to "Download Clip Audio".  
    - False: Wait 1 minute ("Wait6") and retry.  

29. **Add Wait Node "Wait6":**  
    - Wait 1 minute before retrying audio extraction.  

30. **Add HTTP Request "Download Clip Audio":**  
    - Download extracted audio file, configured for file response.  
    - Connect from "If7" True branch.  

31. **Add HTTP Request "Audio Transcription":**  
    - Upload clip audio to OpenAI Whisper API for detailed transcription with timestamps.  
    - Credentials: OpenAI API.  
    - Connect from "Download Clip Audio".  

32. **Add Code Node "Structure Transcription 2":**  
    - Chunk transcription words into 3-word phrases with capitalized first letters.  
    - Outputs transcript chunks.  
    - Connect from "Audio Transcription".  

33. **Add HTTP Request "Editing Clips":**  
    - POST to Andynocode API to generate final clip with clip URLs and transcript chunks.  
    - Credentials: Andynocode API.  
    - Connect from "Structure Transcription 2".  

34. **Add Wait Node "Wait1":**  
    - Wait before checking clip editing status.  
    - Connect from "Editing Clips".  

35. **Add HTTP Request "Clip Ready?":**  
    - Check clip editing job status.  
    - Connect from "Wait1".  

36. **Add If Node "If1":**  
    - Check if clip editing status is "done".  
    - True: Proceed to "Structure Links".  
    - False: Wait 1 minute ("Wait1") and retry.  

37. **Add Set Node "Structure Links":**  
    - Generate final video URL from editing response.  
    - Connect from "If1" True branch.  

38. **Add Langchain Chain "Generate Title":**  
    - Use Google Gemini AI to generate TikTok title and captions based on clip transcript.  
    - Connect from "Structure Links".  

39. **Add Structured Output Parser "Structured Output Parser1":**  
    - Parse AI-generated JSON title and captions.  

40. **Add HTTP Request "Download Final Clip":**  
    - Download final edited clip file.  
    - Connect from "Generate Title".  

41. **Add HTTP Request "Post To TikTok":**  
    - Upload final clip to TikTok via Upload-Post API with title and user info.  
    - Credentials: Upload-Post API key.  
    - Connect from "Download Final Clip".  

42. **Add Wait Node "Posting Interval":**  
    - Wait before processing next clip or batch.  
    - Connect from "Post To TikTok".  

43. **Add Sticky Note nodes:**  
    - Add sticky notes with detailed instructions, setup guides, and usage notes as per source.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow requires accounts and API keys for AssemblyAI (audio transcription), Google AI Studio (Gemini AI), Andynocode (video editing), Upload-Post (TikTok upload), and OpenAI (Whisper transcription).                                                                                                                                                                            | https://www.assemblyai.com, https://aistudio.google.com, https://www.andynocode.com, https://www.upload-post.com, https://openai.com |
| Membership subscription at $29/month needed to activate API token used for audio downloading stage. Activation link: https://lemolex.gumroad.com/l/ypjdr                                                                                                                                                                                                                         | Membership activation and payment page link                                                                 |
| Example copyright-free background videos recommended for best results: Minecraft Parkour (https://youtu.be/XBIaqOm0RKQ?si=gre39-555aRTVsJ1), GTA 5 No Copyright Gameplay (https://youtu.be/z121mUPexGc?si=JGsTOI88tOxXdWYx)                                                                                                                                                     | Example background videos for user input                                                                    |
| Setup instructions detail credential configuration for each node requiring API keys or tokens, including where to paste keys and expected parameter names.                                                                                                                                                                                                                      | See Sticky Note9 content in workflow                                                                        |
| The workflow includes multiple polling loops with wait nodes to handle asynchronous external API jobs gracefully. Consider timeout and retry limits in production.                                                                                                                                                                                                                  | Consider adding retry limits or timeouts to avoid infinite loops                                            |
| The AI models used (Google Gemini) require prompt formatting strictly to produce parsable JSON outputs; errors or format changes may break downstream nodes.                                                                                                                                                                                                                        | Use Langchain output parsers to enforce output structure                                                   |
| The workflow is currently inactive (`active:false`), so enable before production use.                                                                                                                                                                                                                                                                                              | n8n workflow activation required                                                                             |
| The workflow supports multipart form data upload for TikTok video posting and handles file downloads carefully to prevent data loss.                                                                                                                                                                                                                                              | Ensure file size limits and API quotas are respected                                                       |
| For further guidance, contact developer via LinkedIn: https://www.linkedin.com/in/mateofioritorocha/                                                                                                                                                                                                                                                                               | Developer contact                                                                                            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.