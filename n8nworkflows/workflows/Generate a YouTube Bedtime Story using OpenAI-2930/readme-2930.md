Generate a YouTube Bedtime Story using OpenAI

https://n8nworkflows.xyz/workflows/generate-a-youtube-bedtime-story-using-openai-2930


# Generate a YouTube Bedtime Story using OpenAI

### 1. Workflow Overview

This n8n workflow automates the creation of YouTube bedtime story videos, targeting content creators, digital marketers, storytellers, and educators who want to produce engaging, professional videos with minimal manual effort. It transforms audio bedtime stories into fully rendered videos with AI-generated visuals, voiceovers, background music, and automated YouTube publishing.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Reception:** Scheduled trigger initiates the workflow and prepares the input data.
- **1.2 Audio Processing & Transcription:** Handles audio input, generates voiceover, and transcribes audio to text using OpenAI Whisper.
- **1.3 Image Generation:** Creates thematic image prompts, generates images via AI, and stores them.
- **1.4 Video Compilation:** Combines images, voiceover, and music into a video JSON, sends it for rendering, and waits for completion.
- **1.5 YouTube Publishing & Management:** Uploads the video to YouTube, generates metadata, updates Google Sheets, and sends Telegram notifications.
- **1.6 Supporting Utilities:** Includes nodes for data cleaning, splitting, mapping, and error handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

- **Overview:** This block starts the workflow on a schedule and prepares the initial input, including selecting a random animal theme for the story.
- **Nodes Involved:**  
  - Schedule Trigger1  
  - Grab Animal  
  - Generate Voice Over (entry point for next block)

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at defined intervals.  
    - Configuration: Default scheduling parameters (not explicitly shown).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Grab Animal".  
    - Edge Cases: Missed triggers if n8n instance is down.

  - **Grab Animal**  
    - Type: Code  
    - Role: Selects or prepares an animal theme or keyword for the story.  
    - Configuration: Custom JavaScript code to pick or generate an animal name or theme.  
    - Inputs: From Schedule Trigger1.  
    - Outputs: Connects to "Generate Voice Over".  
    - Edge Cases: Code errors or empty output if data source is unavailable.

---

#### 1.2 Audio Processing & Transcription

- **Overview:** This block generates voiceover audio, cleans it, prepares and sends it to OpenAI TTS, saves the audio to Google Cloud Storage (GCS), downloads it back, and transcribes it using OpenAI Whisper.
- **Nodes Involved:**  
  - Generate Voice Over  
  - Clean Voice Over  
  - Prepare OpenAI TTS  
  - OPENai TTS  
  - Wait 5 sec  
  - Save Audio GSC  
  - Map Public Link  
  - Download Audio from GSC  
  - Transcribe with OpenAI Whisper  
  - Create a list of Image Text1 (entry point for next block)

- **Node Details:**

  - **Generate Voice Over**  
    - Type: HTTP Request  
    - Role: Calls an external service or API to generate voiceover audio from text.  
    - Configuration: HTTP POST with text payload, likely to a TTS API.  
    - Inputs: From "Grab Animal".  
    - Outputs: Connects to "Clean Voice Over".  
    - Edge Cases: API errors, timeouts, invalid text input.

  - **Clean Voice Over**  
    - Type: Code  
    - Role: Processes and cleans the voiceover data for further use.  
    - Configuration: JavaScript code to sanitize or format audio data.  
    - Inputs: From "Generate Voice Over".  
    - Outputs: Connects to "Prepare OpenAI TTS".  
    - Edge Cases: Code execution errors.

  - **Prepare OpenAI TTS**  
    - Type: Set  
    - Role: Sets parameters and payload for OpenAI TTS API call.  
    - Configuration: Defines request body, headers, or other parameters.  
    - Inputs: From "Clean Voice Over".  
    - Outputs: Connects to "OPENai TTS".  
    - Edge Cases: Misconfiguration leading to invalid API calls.

  - **OPENai TTS**  
    - Type: HTTP Request  
    - Role: Sends request to OpenAI TTS endpoint to generate speech audio.  
    - Configuration: HTTP POST with retry on failure enabled (wait 5 seconds between tries).  
    - Inputs: From "Prepare OpenAI TTS".  
    - Outputs: Connects to "Wait 5 sec".  
    - Edge Cases: API rate limits, authentication errors.

  - **Wait 5 sec**  
    - Type: Wait  
    - Role: Pauses workflow to allow audio processing completion.  
    - Inputs: From "OPENai TTS".  
    - Outputs: Connects to "Save Audio GSC".  
    - Edge Cases: None significant.

  - **Save Audio GSC**  
    - Type: Google Cloud Storage  
    - Role: Uploads generated audio file to GCS bucket.  
    - Configuration: Uses GCS credentials, target bucket, and file path.  
    - Inputs: From "Wait 5 sec".  
    - Outputs: Connects to "Map Public Link".  
    - Edge Cases: Authentication errors, bucket permission issues.

  - **Map Public Link**  
    - Type: Set  
    - Role: Constructs a public URL for the uploaded audio file.  
    - Inputs: From "Save Audio GSC".  
    - Outputs: Connects to "Download Audio from GSC".  
    - Edge Cases: Incorrect URL formatting.

  - **Download Audio from GSC**  
    - Type: HTTP Request  
    - Role: Downloads the audio file from GCS for transcription.  
    - Inputs: From "Map Public Link".  
    - Outputs: Connects to "Transcribe with OpenAI Whisper".  
    - Edge Cases: Network errors, file not found.

  - **Transcribe with OpenAI Whisper**  
    - Type: HTTP Request  
    - Role: Sends audio file to OpenAI Whisper API for transcription.  
    - Inputs: From "Download Audio from GSC".  
    - Outputs: Connects to "Create a list of Image Text1".  
    - Edge Cases: API errors, transcription inaccuracies.

---

#### 1.3 Image Generation

- **Overview:** This block processes the transcript, generates image prompts, calls an AI image generation API, converts images to files, saves them to GCS, and prepares image URLs.
- **Nodes Involved:**  
  - Create a list of Image Text1  
  - Split Out1  
  - Combine Image Style and Elements  
  - Split Out - Image prompts  
  - Make Image Prompts Gemini  
  - Clean Image prompts  
  - Get image Base  
  - Convert to File  
  - Wait 6 sec  
  - Save images to GC  
  - Limit  
  - Image Link  
  - Combine Transcript1 (entry point for next block)

- **Node Details:**

  - **Create a list of Image Text1**  
    - Type: Code  
    - Role: Generates a list of text snippets for image prompts based on transcription.  
    - Inputs: From "Transcribe with OpenAI Whisper".  
    - Outputs: Connects to "Split Out1".  
    - Edge Cases: Empty or malformed text input.

  - **Split Out1**  
    - Type: Split Out  
    - Role: Splits the list of image text prompts into individual items for parallel processing.  
    - Inputs: From "Create a list of Image Text1".  
    - Outputs: Connects to "Combine Image Style and Elements".  
    - Edge Cases: Empty list handling.

  - **Combine Image Style and Elements**  
    - Type: Code  
    - Role: Combines style elements with image text prompts to form full image generation prompts.  
    - Inputs: From "Split Out1".  
    - Outputs: Connects to "Split Out - Image prompts".  
    - Edge Cases: Code errors or missing style data.

  - **Split Out - Image prompts**  
    - Type: Split Out  
    - Role: Further splits combined prompts for individual API calls.  
    - Inputs: From "Combine Image Style and Elements".  
    - Outputs: Connects to "Make Image Prompts Gemini".  
    - Edge Cases: Empty or invalid prompts.

  - **Make Image Prompts Gemini**  
    - Type: HTTP Request  
    - Role: Calls an AI image generation API (Gemini) to create images from prompts.  
    - Inputs: From "Split Out - Image prompts".  
    - Outputs: Connects to "Clean Image prompts".  
    - Edge Cases: API failures, rate limits.

  - **Clean Image prompts**  
    - Type: Code  
    - Role: Cleans and formats image generation API responses.  
    - Inputs: From "Make Image Prompts Gemini".  
    - Outputs: Connects to "Get image Base".  
    - Edge Cases: Parsing errors.

  - **Get image Base**  
    - Type: HTTP Request  
    - Role: Retrieves base image data or downloads images.  
    - Inputs: From "Clean Image prompts".  
    - Outputs: Connects to "Convert to File".  
    - Edge Cases: Network errors, invalid URLs.

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts image data into file format for storage.  
    - Inputs: From "Get image Base".  
    - Outputs: Connects to "Wait 6 sec".  
    - Edge Cases: Conversion failures.

  - **Wait 6 sec**  
    - Type: Wait  
    - Role: Allows time for file processing or API rate limits.  
    - Inputs: From "Convert to File".  
    - Outputs: Connects to "Save images to GC".  
    - Edge Cases: None significant.

  - **Save images to GC**  
    - Type: Google Cloud Storage  
    - Role: Uploads image files to GCS bucket.  
    - Inputs: From "Wait 6 sec".  
    - Outputs: Connects to "Limit".  
    - Edge Cases: Authentication or permission errors.

  - **Limit**  
    - Type: Limit  
    - Role: Controls concurrency or limits number of images processed.  
    - Inputs: From "Save images to GC".  
    - Outputs: Connects to "Image Link".  
    - Edge Cases: Over-limit errors.

  - **Image Link**  
    - Type: Set  
    - Role: Sets or formats public URLs for the uploaded images.  
    - Inputs: From "Limit".  
    - Outputs: Connects to "Combine Transcript1".  
    - Edge Cases: URL formatting errors.

---

#### 1.4 Video Compilation

- **Overview:** This block selects background music, generates video JSON, creates the video via API, waits for rendering, checks progress, and sets the final video URL.
- **Nodes Involved:**  
  - Combine Transcript1  
  - Get Random BG music  
  - Pick Random song From GSC  
  - Map Music  
  - Generate Json to Video  
  - Create Video  
  - Wait 6 min for Rendering  
  - Get Video Progress  
  - Set Correct Video Url  
  - Clean Transcript  
  - Generate Transcript With AI  
  - Generate Title with AI  
  - Append GS (entry point for next block)

- **Node Details:**

  - **Combine Transcript1**  
    - Type: Code  
    - Role: Combines transcript text and image links into a format for video generation.  
    - Inputs: From "Image Link".  
    - Outputs: Connects to "Get Random BG music".  
    - Edge Cases: Data mismatch or empty inputs.

  - **Get Random BG music**  
    - Type: Google Cloud Storage  
    - Role: Retrieves a list or bucket of background music files.  
    - Inputs: From "Combine Transcript1".  
    - Outputs: Connects to "Pick Random song From GSC".  
    - Edge Cases: Empty bucket or permission issues.

  - **Pick Random song From GSC**  
    - Type: Code  
    - Role: Selects a random background music track from the list.  
    - Inputs: From "Get Random BG music".  
    - Outputs: Connects to "Map Music".  
    - Edge Cases: Empty input list.

  - **Map Music**  
    - Type: Set  
    - Role: Sets music metadata or URL for video generation.  
    - Inputs: From "Pick Random song From GSC".  
    - Outputs: Connects to "Generate Json to Video".  
    - Edge Cases: Incorrect mapping.

  - **Generate Json to Video**  
    - Type: Code  
    - Role: Generates a JSON payload describing the video composition (images, audio, music).  
    - Inputs: From "Map Music".  
    - Outputs: Connects to "Create Video".  
    - Edge Cases: JSON formatting errors.

  - **Create Video**  
    - Type: HTTP Request  
    - Role: Sends video JSON to rendering API (samautomation.work) to create the video.  
    - Inputs: From "Generate Json to Video".  
    - Outputs: Connects to "Wait 6 min for Rendering".  
    - Edge Cases: API errors, timeouts.

  - **Wait 6 min for Rendering**  
    - Type: Wait  
    - Role: Pauses workflow to allow video rendering to complete.  
    - Inputs: From "Create Video".  
    - Outputs: Connects to "Get Video Progress".  
    - Edge Cases: Insufficient wait time.

  - **Get Video Progress**  
    - Type: HTTP Request  
    - Role: Checks the rendering status and retrieves the video URL.  
    - Inputs: From "Wait 6 min for Rendering".  
    - Outputs: Connects to "Set Correct Video Url".  
    - Edge Cases: API failures, incomplete rendering.

  - **Set Correct Video Url**  
    - Type: Set  
    - Role: Formats or corrects the final video URL for further use.  
    - Inputs: From "Get Video Progress".  
    - Outputs: Connects to "Clean Transcript".  
    - Edge Cases: URL formatting errors.

  - **Clean Transcript**  
    - Type: Code  
    - Role: Cleans and prepares the transcript for metadata generation.  
    - Inputs: From "Set Correct Video Url".  
    - Outputs: Connects to "Generate Transcript With AI".  
    - Edge Cases: Code errors.

  - **Generate Transcript With AI**  
    - Type: HTTP Request  
    - Role: Uses AI to generate a refined transcript or description.  
    - Inputs: From "Clean Transcript".  
    - Outputs: Connects to "Generate Title with AI".  
    - Edge Cases: API errors.

  - **Generate Title with AI**  
    - Type: HTTP Request  
    - Role: Generates a video title using AI based on transcript or story content.  
    - Inputs: From "Generate Transcript With AI".  
    - Outputs: Connects to "Append GS".  
    - Edge Cases: API errors.

---

#### 1.5 YouTube Publishing & Management

- **Overview:** This block uploads the video to YouTube, updates Google Sheets with video data, and sends Telegram notifications.
- **Nodes Involved:**  
  - Append GS  
  - Grab Video from GS  
  - Download Video  
  - YouTube Seos Kanaal  
  - Upgrade Progress on GS  
  - Send message with Telegram

- **Node Details:**

  - **Append GS**  
    - Type: Google Sheets  
    - Role: Appends new video metadata and details to a Google Sheet for tracking.  
    - Inputs: From "Generate Title with AI".  
    - Outputs: None (end of chain).  
    - Edge Cases: API quota limits, permission errors.

  - **Grab Video from GS**  
    - Type: Google Sheets  
    - Role: Retrieves video data from Google Sheets for further processing or download.  
    - Inputs: None (likely manual or triggered separately).  
    - Outputs: Connects to "Download Video".  
    - Edge Cases: Empty or missing data.

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads the video file from storage or URL.  
    - Inputs: From "Grab Video from GS".  
    - Outputs: Connects to "YouTube Seos Kanaal".  
    - Edge Cases: Network errors.

  - **YouTube Seos Kanaal**  
    - Type: YouTube  
    - Role: Uploads the video to YouTube channel with SEO metadata.  
    - Inputs: From "Download Video".  
    - Outputs: Connects to "Upgrade Progress on GS".  
    - Edge Cases: Authentication errors, upload failures.

  - **Upgrade Progress on GS**  
    - Type: Google Sheets  
    - Role: Updates the Google Sheet with upload status or progress.  
    - Inputs: From "YouTube Seos Kanaal".  
    - Outputs: Connects to "Send message with Telegram".  
    - Edge Cases: API errors.

  - **Send message with Telegram**  
    - Type: Telegram  
    - Role: Sends notification messages about workflow progress or completion.  
    - Inputs: From "Upgrade Progress on GS".  
    - Outputs: None (workflow end).  
    - Edge Cases: Telegram API errors.

---

#### 1.6 Supporting Utilities

- **Overview:** Various code and utility nodes support data cleaning, splitting, and formatting throughout the workflow.
- **Nodes Involved:**  
  - Combine Transcript1  
  - Create a list of Image Text1  
  - Split Out1  
  - Combine Image Style and Elements  
  - Clean Image prompts  
  - Clean Transcript  
  - Clean Voice Over  
  - Pick Random song From GSC  
  - Map Music  
  - Map Public Link  
  - Image Link  
  - Limit  
  - Wait nodes (5 sec, 6 sec, 6 min)

- **Node Details:**  
  These nodes perform data transformations, splitting lists for parallel processing, waiting for asynchronous operations, and mapping data fields. They are critical for smooth data flow and error handling.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                         |
|---------------------------|------------------------|----------------------------------------|--------------------------------|--------------------------------|-----------------------------------|
| Schedule Trigger1          | Schedule Trigger       | Initiates workflow on schedule         | None                           | Grab Animal                    |                                   |
| Grab Animal               | Code                   | Selects animal theme for story         | Schedule Trigger1              | Generate Voice Over            |                                   |
| Generate Voice Over       | HTTP Request           | Generates voiceover audio               | Grab Animal                   | Clean Voice Over               |                                   |
| Clean Voice Over          | Code                   | Cleans voiceover data                   | Generate Voice Over            | Prepare OpenAI TTS             |                                   |
| Prepare OpenAI TTS        | Set                    | Prepares TTS API request parameters    | Clean Voice Over               | OPENai TTS                    |                                   |
| OPENai TTS               | HTTP Request           | Calls OpenAI TTS API                    | Prepare OpenAI TTS             | Wait 5 sec                   |                                   |
| Wait 5 sec               | Wait                   | Waits for audio processing              | OPENai TTS                    | Save Audio GSC                |                                   |
| Save Audio GSC            | Google Cloud Storage   | Uploads audio to GCS                     | Wait 5 sec                   | Map Public Link               |                                   |
| Map Public Link           | Set                    | Creates public URL for audio             | Save Audio GSC                | Download Audio from GSC       |                                   |
| Download Audio from GSC   | HTTP Request           | Downloads audio file from GCS            | Map Public Link               | Transcribe with OpenAI Whisper |                                   |
| Transcribe with OpenAI Whisper | HTTP Request      | Transcribes audio to text                | Download Audio from GSC        | Create a list of Image Text1  |                                   |
| Create a list of Image Text1 | Code                | Creates image prompt texts from transcript | Transcribe with OpenAI Whisper | Split Out1                   |                                   |
| Split Out1               | Split Out              | Splits image text list for processing   | Create a list of Image Text1   | Combine Image Style and Elements |                                   |
| Combine Image Style and Elements | Code             | Combines style with image prompts       | Split Out1                   | Split Out - Image prompts     |                                   |
| Split Out - Image prompts | Split Out              | Splits combined prompts                  | Combine Image Style and Elements | Make Image Prompts Gemini    |                                   |
| Make Image Prompts Gemini | HTTP Request           | Calls AI image generation API            | Split Out - Image prompts      | Clean Image prompts           |                                   |
| Clean Image prompts       | Code                   | Cleans image generation responses        | Make Image Prompts Gemini      | Get image Base               |                                   |
| Get image Base            | HTTP Request           | Retrieves base image data                 | Clean Image prompts            | Convert to File              |                                   |
| Convert to File           | Convert To File        | Converts image data to file format        | Get image Base                | Wait 6 sec                  |                                   |
| Wait 6 sec               | Wait                   | Waits for file processing                 | Convert to File               | Save images to GC            |                                   |
| Save images to GC         | Google Cloud Storage   | Uploads images to GCS                      | Wait 6 sec                   | Limit                       |                                   |
| Limit                    | Limit                  | Limits concurrency or number of images    | Save images to GC             | Image Link                  |                                   |
| Image Link               | Set                    | Sets public URLs for images                | Limit                        | Combine Transcript1          |                                   |
| Combine Transcript1       | Code                   | Combines transcript and image data        | Image Link                   | Get Random BG music          |                                   |
| Get Random BG music       | Google Cloud Storage   | Retrieves background music list            | Combine Transcript1           | Pick Random song From GSC    |                                   |
| Pick Random song From GSC | Code                   | Selects random background music            | Get Random BG music           | Map Music                   |                                   |
| Map Music                | Set                    | Maps music data for video generation       | Pick Random song From GSC     | Generate Json to Video       |                                   |
| Generate Json to Video    | Code                   | Creates video composition JSON             | Map Music                   | Create Video                |                                   |
| Create Video             | HTTP Request           | Sends video JSON to rendering API          | Generate Json to Video        | Wait 6 min for Rendering    |                                   |
| Wait 6 min for Rendering | Wait                   | Waits for video rendering completion        | Create Video                 | Get Video Progress          |                                   |
| Get Video Progress       | HTTP Request           | Checks video rendering status               | Wait 6 min for Rendering      | Set Correct Video Url       |                                   |
| Set Correct Video Url    | Set                    | Formats final video URL                      | Get Video Progress           | Clean Transcript            |                                   |
| Clean Transcript         | Code                   | Cleans transcript for metadata generation   | Set Correct Video Url        | Generate Transcript With AI |                                   |
| Generate Transcript With AI | HTTP Request         | Generates refined transcript via AI          | Clean Transcript             | Generate Title with AI      |                                   |
| Generate Title with AI   | HTTP Request           | Generates video title using AI                | Generate Transcript With AI  | Append GS                  |                                   |
| Append GS                | Google Sheets          | Appends video metadata to Google Sheets       | Generate Title with AI       | None                      |                                   |
| Grab Video from GS       | Google Sheets          | Retrieves video data from Google Sheets        | None                       | Download Video             |                                   |
| Download Video           | HTTP Request           | Downloads video file                            | Grab Video from GS           | YouTube Seos Kanaal        |                                   |
| YouTube Seos Kanaal      | YouTube                | Uploads video to YouTube channel                 | Download Video              | Upgrade Progress on GS     |                                   |
| Upgrade Progress on GS   | Google Sheets          | Updates upload progress in Google Sheets          | YouTube Seos Kanaal         | Send message with Telegram |                                   |
| Send message with Telegram | Telegram              | Sends notification messages                      | Upgrade Progress on GS      | None                      |                                   |
| Sticky Note              | Sticky Note            | (Empty content)                                 | None                       | None                      |                                   |
| Sticky Note1             | Sticky Note            | (Empty content)                                 | None                       | None                      |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run at desired intervals (e.g., daily).  
   - Connect output to "Grab Animal".

2. **Create "Grab Animal" Code Node**  
   - Type: Code  
   - Write JavaScript to select or generate an animal theme keyword.  
   - Input: None (triggered by Schedule Trigger).  
   - Output: Connect to "Generate Voice Over".

3. **Create "Generate Voice Over" HTTP Request Node**  
   - Type: HTTP Request  
   - Configure to call TTS API with text from "Grab Animal".  
   - Method: POST, with appropriate headers and body.  
   - Output: Connect to "Clean Voice Over".

4. **Create "Clean Voice Over" Code Node**  
   - Type: Code  
   - Clean or format the voiceover response.  
   - Output: Connect to "Prepare OpenAI TTS".

5. **Create "Prepare OpenAI TTS" Set Node**  
   - Type: Set  
   - Define parameters for OpenAI TTS API call (e.g., voice, speed).  
   - Output: Connect to "OPENai TTS".

6. **Create "OPENai TTS" HTTP Request Node**  
   - Type: HTTP Request  
   - Configure to call OpenAI TTS endpoint with retry enabled (retry on fail, wait 5s).  
   - Output: Connect to "Wait 5 sec".

7. **Create "Wait 5 sec" Node**  
   - Type: Wait  
   - Wait for 5 seconds to allow processing.  
   - Output: Connect to "Save Audio GSC".

8. **Create "Save Audio GSC" Google Cloud Storage Node**  
   - Type: Google Cloud Storage  
   - Configure with GCS credentials and target bucket for audio upload.  
   - Output: Connect to "Map Public Link".

9. **Create "Map Public Link" Set Node**  
   - Type: Set  
   - Construct public URL for uploaded audio file.  
   - Output: Connect to "Download Audio from GSC".

10. **Create "Download Audio from GSC" HTTP Request Node**  
    - Type: HTTP Request  
    - Download audio file from public GCS URL.  
    - Output: Connect to "Transcribe with OpenAI Whisper".

11. **Create "Transcribe with OpenAI Whisper" HTTP Request Node**  
    - Type: HTTP Request  
    - Call OpenAI Whisper API to transcribe audio.  
    - Output: Connect to "Create a list of Image Text1".

12. **Create "Create a list of Image Text1" Code Node**  
    - Type: Code  
    - Generate list of image prompt texts from transcript.  
    - Output: Connect to "Split Out1".

13. **Create "Split Out1" Split Out Node**  
    - Type: Split Out  
    - Split image prompt list for parallel processing.  
    - Output: Connect to "Combine Image Style and Elements".

14. **Create "Combine Image Style and Elements" Code Node**  
    - Type: Code  
    - Combine style elements with prompts.  
    - Output: Connect to "Split Out - Image prompts".

15. **Create "Split Out - Image prompts" Split Out Node**  
    - Type: Split Out  
    - Further split combined prompts.  
    - Output: Connect to "Make Image Prompts Gemini".

16. **Create "Make Image Prompts Gemini" HTTP Request Node**  
    - Type: HTTP Request  
    - Call AI image generation API with prompts.  
    - Output: Connect to "Clean Image prompts".

17. **Create "Clean Image prompts" Code Node**  
    - Type: Code  
    - Clean and format image API responses.  
    - Output: Connect to "Get image Base".

18. **Create "Get image Base" HTTP Request Node**  
    - Type: HTTP Request  
    - Retrieve or download base image data.  
    - Output: Connect to "Convert to File".

19. **Create "Convert to File" Convert To File Node**  
    - Type: Convert To File  
    - Convert image data to file format.  
    - Output: Connect to "Wait 6 sec".

20. **Create "Wait 6 sec" Node**  
    - Type: Wait  
    - Wait 6 seconds for processing.  
    - Output: Connect to "Save images to GC".

21. **Create "Save images to GC" Google Cloud Storage Node**  
    - Type: Google Cloud Storage  
    - Upload images to GCS bucket.  
    - Output: Connect to "Limit".

22. **Create "Limit" Limit Node**  
    - Type: Limit  
    - Control concurrency or number of images processed.  
    - Output: Connect to "Image Link".

23. **Create "Image Link" Set Node**  
    - Type: Set  
    - Set public URLs for images.  
    - Output: Connect to "Combine Transcript1".

24. **Create "Combine Transcript1" Code Node**  
    - Type: Code  
    - Combine transcript and image URLs for video generation.  
    - Output: Connect to "Get Random BG music".

25. **Create "Get Random BG music" Google Cloud Storage Node**  
    - Type: Google Cloud Storage  
    - Retrieve background music list.  
    - Output: Connect to "Pick Random song From GSC".

26. **Create "Pick Random song From GSC" Code Node**  
    - Type: Code  
    - Select random background music track.  
    - Output: Connect to "Map Music".

27. **Create "Map Music" Set Node**  
    - Type: Set  
    - Map music metadata for video.  
    - Output: Connect to "Generate Json to Video".

28. **Create "Generate Json to Video" Code Node**  
    - Type: Code  
    - Generate JSON payload for video rendering.  
    - Output: Connect to "Create Video".

29. **Create "Create Video" HTTP Request Node**  
    - Type: HTTP Request  
    - Send JSON to video rendering API (samautomation.work).  
    - Output: Connect to "Wait 6 min for Rendering".

30. **Create "Wait 6 min for Rendering" Node**  
    - Type: Wait  
    - Wait 6 minutes for video rendering.  
    - Output: Connect to "Get Video Progress".

31. **Create "Get Video Progress" HTTP Request Node**  
    - Type: HTTP Request  
    - Check rendering status and get video URL.  
    - Output: Connect to "Set Correct Video Url".

32. **Create "Set Correct Video Url" Set Node**  
    - Type: Set  
    - Format final video URL.  
    - Output: Connect to "Clean Transcript".

33. **Create "Clean Transcript" Code Node**  
    - Type: Code  
    - Clean transcript for metadata.  
    - Output: Connect to "Generate Transcript With AI".

34. **Create "Generate Transcript With AI" HTTP Request Node**  
    - Type: HTTP Request  
    - Generate refined transcript or description.  
    - Output: Connect to "Generate Title with AI".

35. **Create "Generate Title with AI" HTTP Request Node**  
    - Type: HTTP Request  
    - Generate video title.  
    - Output: Connect to "Append GS".

36. **Create "Append GS" Google Sheets Node**  
    - Type: Google Sheets  
    - Append video metadata to Google Sheets.  
    - Output: None (end).

37. **Create "Grab Video from GS" Google Sheets Node**  
    - Type: Google Sheets  
    - Retrieve video data for download or upload.  
    - Output: Connect to "Download Video".

38. **Create "Download Video" HTTP Request Node**  
    - Type: HTTP Request  
    - Download video file.  
    - Output: Connect to "YouTube Seos Kanaal".

39. **Create "YouTube Seos Kanaal" YouTube Node**  
    - Type: YouTube  
    - Upload video to YouTube channel.  
    - Output: Connect to "Upgrade Progress on GS".

40. **Create "Upgrade Progress on GS" Google Sheets Node**  
    - Type: Google Sheets  
    - Update upload progress in Google Sheets.  
    - Output: Connect to "Send message with Telegram".

41. **Create "Send message with Telegram" Telegram Node**  
    - Type: Telegram  
    - Send notification message.  
    - Output: None (end).

42. **Add Wait Nodes**  
    - Wait 5 sec, Wait 6 sec, Wait 6 min as per above steps.

43. **Set up Credentials**  
    - OpenAI API key for Whisper and TTS.  
    - Google Cloud Storage credentials with proper bucket access.  
    - YouTube OAuth2 credentials for video upload.  
    - Telegram Bot API token for notifications.

44. **Test each node individually** to ensure API keys and parameters are correctly set.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Video rendering service used: samautomation.work - best and cheapest online video rendering service | [Samautomation.work](https://samautomation.work)                                                |
| Demo video showcasing the final output of the workflow                                             | [Demo Video](https://www.youtube.com/watch?v=Y3TL59rMay8&ab_channel=Samautomation)              |
| Workflow cost efficiency: approximately $0.25 per video produced                                   | Workflow description section                                                                    |
| Support available via WhatsApp for customization and setup                                         | Workflow description section                                                                    |
| Aspect ratio default is 9:16, can be changed to 16:9 for long-form content                         | Workflow description section                                                                    |
| API keys required: OpenAI, Google Cloud Storage, YouTube                                           | Setup Requirements section                                                                      |
| Workflow saves execution progress and uses error workflow for handling failures                    | Workflow settings                                                                               |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Generate a YouTube Bedtime Story using OpenAI" n8n workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with the workflow.