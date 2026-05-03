Convert Form Inputs to Cinematic Videos with GPT-4, Dumpling AI & ElevenLabs Audio

https://n8nworkflows.xyz/workflows/convert-form-inputs-to-cinematic-videos-with-gpt-4--dumpling-ai---elevenlabs-audio-5393


# Convert Form Inputs to Cinematic Videos with GPT-4, Dumpling AI & ElevenLabs Audio

### 1. Workflow Overview

This workflow automates the creation of cinematic animal-themed videos with ambient audio based on user-submitted form inputs. Users provide a video title, four animal or country names, and a visual style via a form. The workflow generates vivid cinematic prompts using GPT-4, creates images with Dumpling AI, converts those images into animated motion videos using Replicate.com (Leonardo AI), generates ambient audio via ElevenLabs, and finally merges all video clips and audio into a single cinematic video using Creatomate. The completed video and audio files are uploaded to Google Drive, and metadata is logged into Google Sheets for tracking.

Logical blocks include:

- **1.1 Input Reception:** Receiving and formatting form submissions.
- **1.2 Cinematic Prompt Generation:** Creating detailed image prompts via GPT-4.
- **1.3 AI Image Generation:** Generating images using Dumpling AI.
- **1.4 Motion Video Prompt Creation:** Crafting motion video prompts for animation.
- **1.5 Motion Video Generation:** Producing short animated videos with Replicate.com.
- **1.6 Audio Prompt and Soundtrack Generation:** Creating sound prompts and ambient audio with GPT-4 and ElevenLabs.
- **1.7 Audio Upload and Sharing:** Uploading audio files to Google Drive and making them public.
- **1.8 Video and Audio Merging:** Combining clips and audio into one video via Creatomate.
- **1.9 Final Upload and Logging:** Uploading final video to Drive and logging information to Sheets.
- **1.10 Workflow Documentation:** Sticky note with overall process summary.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block captures user input from a form, formats the animal/country inputs into an array for iterative processing.
- **Nodes Involved:**  
  - Form: User Submission  
  - Format into an Array  
  - Split: Loop Through Array  

- **Node Details:**

  - **Form: User Submission**  
    - Type: `formTrigger`  
    - Role: Trigger node initiating workflow on form submission.  
    - Config: Form titled "Content" with required fields: Title, Country 1, Country 2, Country 4; Country 3 optional.  
    - Inputs: Webhook trigger  
    - Outputs: Form data JSON  
    - Edge cases: Missing required fields; malformed submissions.

  - **Format into an Array**  
    - Type: `set`  
    - Role: Converts individual form inputs ("animal 1" to "animal 4") into an array named `animals`.  
    - Config: Assigns `animals` as array of form fields values.  
    - Inputs: Form output JSON  
    - Outputs: JSON with `animals` array  
    - Edge cases: Empty or missing animal entries may cause issues downstream if not handled.

  - **Split: Loop Through Array**  
    - Type: `splitOut`  
    - Role: Splits the `animals` array into separate items for parallel processing.  
    - Config: Field to split on: `animals`  
    - Inputs: JSON with `animals` array  
    - Outputs: Individual animal objects for each array element  
    - Edge cases: Empty arrays produce no output; errors if `animals` is not an array.

---

#### 1.2 Cinematic Prompt Generation

- **Overview:** Transforms each animal or country name into a vivid, cinematic image prompt using GPT-4.
- **Nodes Involved:**  
  - GPT-4: Create Cinematic Prompt  
  - Clean: Remove Line Breaks from Prompt  

- **Node Details:**

  - **GPT-4: Create Cinematic Prompt**  
    - Type: OpenAI Langchain node  
    - Role: Generates detailed, cinematic prompts describing a mythical warrior with animal or country elements.  
    - Config: Uses GPT-4.1 model with detailed system prompt specifying output rules (max 950 chars, cinematic style, environment details).  
    - Inputs: Individual animal/country from Split node  
    - Outputs: Text prompt content  
    - Expressions: Uses `Country: {{ $json.animals }}` in system prompt to pass context.  
    - Edge cases: API rate limits, malformed inputs, overly long outputs truncated.

  - **Clean: Remove Line Breaks from Prompt**  
    - Type: `set`  
    - Role: Removes newline characters from GPT-4 prompt output to prepare for API calls.  
    - Config: Regex replace `\n` and escaped `\\n` with spaces on `message.content`.  
    - Inputs: GPT-4 output JSON  
    - Outputs: Cleaned prompt string property `prompt`  
    - Edge cases: If GPT-4 output format changes, regex may fail; empty or null inputs.

---

#### 1.3 AI Image Generation

- **Overview:** Uses Dumpling AI API to generate high-quality images based on cinematic prompts.
- **Nodes Involved:**  
  - Dumpling AI: Generate Image  

- **Node Details:**

  - **Dumpling AI: Generate Image**  
    - Type: `httpRequest`  
    - Role: Sends prompt to Dumpling AI API to generate an image.  
    - Config: POST to Dumpling AI endpoint with JSON body containing `"model": "FLUX.1-pro"` and prompt. Uses HTTP header auth with API key.  
    - Inputs: Cleaned prompt string  
    - Outputs: JSON with image URLs and metadata  
    - Credentials: Dumpling AI API key  
    - Edge cases: API failures, invalid prompts, network issues, rate limits.

---

#### 1.4 Motion Video Prompt Creation

- **Overview:** Converts image prompts into short motion video prompts describing natural movements for animation.
- **Nodes Involved:**  
  - GPT-4: Create motion prompt  

- **Node Details:**

  - **GPT-4: Create motion prompt**  
    - Type: OpenAI Langchain node  
    - Role: Rewrites image prompts into concise motion prompts for Leonardo AI.  
    - Config: GPT-4.1 model; system prompt instructs to add natural motion elements and keep output 1-2 sentences.  
    - Inputs: Cleaned image prompt from previous step  
    - Outputs: Motion video prompt text  
    - Edge cases: API errors, output length violations, malformed prompts.

---

#### 1.5 Motion Video Generation

- **Overview:** Generates short animated videos from static images using Replicate.com’s Leonardo AI model.
- **Nodes Involved:**  
  - Replicate.com: Create Motion Video  
  - Wait: Leonardo Processing  
  - Fetch: Download Motion Video  

- **Node Details:**

  - **Replicate.com: Create Motion Video**  
    - Type: `httpRequest`  
    - Role: Calls Replicate.com API to start motion video generation using input image and motion prompt.  
    - Config: POST JSON with image URL and prompt; headers include API key and accept type; waits for synchronous response if possible.  
    - Inputs: Image URL from Dumpling AI, motion prompt from GPT-4  
    - Outputs: Prediction ID and status  
    - Credentials: Replicate.com API key  
    - Edge cases: API errors, long processing times, invalid inputs.

  - **Wait: Leonardo Processing**  
    - Type: `wait`  
    - Role: Pauses workflow for 60 seconds to allow video generation to complete asynchronously.  
    - Config: Fixed 60 seconds delay  
    - Inputs: Triggered after Replicate.com call  
    - Outputs: Pass-through  
    - Edge cases: Insufficient wait time may cause premature fetch; excessive wait delays workflow.

  - **Fetch: Download Motion Video**  
    - Type: `httpRequest`  
    - Role: Retrieves generated motion video URL from Replicate.com API using prediction ID.  
    - Config: GET request to Replicate.com prediction endpoint with authentication header.  
    - Inputs: Prediction ID from previous node  
    - Outputs: JSON with video URL and metadata  
    - Credentials: Replicate.com API key  
    - Edge cases: API errors, video not ready, network failures.

---

#### 1.6 Audio Prompt and Soundtrack Generation

- **Overview:** Generates an ambient sound prompt based on user-selected style, then creates a 20-second ambient audio track.
- **Nodes Involved:**  
  - Limit: One Audio Track Per Run  
  - GPT-4: Generate Audio Prompt  
  - ElevenLabs: Create Ambient Soundtrack  

- **Node Details:**

  - **Limit: One Audio Track Per Run**  
    - Type: `limit`  
    - Role: Ensures only one audio prompt and audio track is generated per workflow execution.  
    - Config: Defaults (limit 1)  
    - Inputs: Pass-through from video branches  
    - Outputs: Single item  
    - Edge cases: Misconfiguration may block audio generation.

  - **GPT-4: Generate Audio Prompt**  
    - Type: OpenAI Langchain node  
    - Role: Creates a vivid, descriptive sound prompt for ambient audio using the user’s style input.  
    - Config: GPT-4.1 with system prompt to generate immersive audio scene descriptions, 1-2 sentences.  
    - Inputs: Style field from form submission  
    - Outputs: Text prompt for audio generation  
    - Edge cases: API failures, unclear style inputs.

  - **ElevenLabs: Create Ambient Soundtrack**  
    - Type: `httpRequest`  
    - Role: Calls ElevenLabs sound generation API to produce a 20-second ambient audio file from the text prompt.  
    - Config: POST with JSON body including text from GPT-4 and duration 20 seconds; uses HTTP header auth.  
    - Inputs: Audio prompt text  
    - Outputs: Audio file URL or binary data  
    - Credentials: ElevenLabs API key  
    - Edge cases: API limits, invalid text, auth errors.

---

#### 1.7 Audio Upload and Sharing

- **Overview:** Uploads the generated audio to Google Drive and sets sharing permissions to public.
- **Nodes Involved:**  
  - Upload: Save Audio to Google Drive  
  - Share: Make Audio Public  

- **Node Details:**

  - **Upload: Save Audio to Google Drive**  
    - Type: `googleDrive`  
    - Role: Saves the .mp3 audio file to a specified folder in Google Drive named by video title.  
    - Config: Drive "My Drive", folder ID for "Soundtrack" folder, filename uses form Title + ".mp3".  
    - Inputs: Audio file URL or binary from ElevenLabs  
    - Outputs: Metadata including file ID and webContentLink  
    - Credentials: Google Drive OAuth2  
    - Edge cases: Drive permission errors, quota limits, upload failures.

  - **Share: Make Audio Public**  
    - Type: `googleDrive`  
    - Role: Applies sharing permissions to make audio file accessible to anyone with the link.  
    - Config: Share role "reader", type "anyone", allow file discovery true.  
    - Inputs: File ID from upload node  
    - Outputs: Confirmation of sharing status  
    - Credentials: Google Drive OAuth2  
    - Edge cases: Permission denial, API quota issues.

---

#### 1.8 Video and Audio Merging

- **Overview:** Combines all four generated motion videos and the ambient audio track into a single cinematic video with text overlays.
- **Nodes Involved:**  
  - Merge: Combine Videos & Audio Branch  
  - Format Motion Video URLs  
  - Creatomate: Combine Videos & Audio  
  - Wait: Creatomate Rendering  
  - Download: Final MP4 from Creatomate  

- **Node Details:**

  - **Merge: Combine Videos & Audio Branch**  
    - Type: `merge`  
    - Role: Combines video branch outputs and audio branch output into one dataset for final processing.  
    - Config: Combine mode "combineAll" (all inputs merged into one array).  
    - Inputs: Video JSONs and audio file info  
    - Outputs: Combined JSON array  
    - Edge cases: Mismatched input counts cause incomplete merges.

  - **Format Motion Video URLs**  
    - Type: `code` (JavaScript)  
    - Role: Extracts and formats motion video URLs and metadata into a unified JSON array for Creatomate.  
    - Config: Iterates items to build array with keys: motionMP4URL, imageId, createdAt.  
    - Inputs: Merged video and audio data  
    - Outputs: Single JSON object with `urls` array  
    - Edge cases: Missing video URLs or malformed data.

  - **Creatomate: Combine Videos & Audio**  
    - Type: `httpRequest`  
    - Role: Sends a render request to Creatomate API with template ID and modifications to combine videos, audio, and text labels.  
    - Config: POST JSON body with 4 video sources, 1 audio source, and 4 text overlays using form animal names. Authorization header required (token not shown).  
    - Inputs: Form data, video URLs, audio link  
    - Outputs: Render job info including render URL  
    - Edge cases: API errors, invalid media URLs, auth failures.

  - **Wait: Creatomate Rendering**  
    - Type: `wait`  
    - Role: Pauses workflow 60 seconds to allow Creatomate to complete video rendering.  
    - Config: 60 seconds delay  
    - Inputs: Triggered after Creatomate request  
    - Outputs: Pass-through  
    - Edge cases: Insufficient wait time may cause premature download.

  - **Download: Final MP4 from Creatomate**  
    - Type: `httpRequest`  
    - Role: Downloads the final rendered MP4 video from Creatomate using render URL.  
    - Config: GET request to URL from previous node.  
    - Inputs: Render URL from Creatomate node  
    - Outputs: Binary video file or link metadata  
    - Edge cases: Download failures, URL expiration.

---

#### 1.9 Final Upload and Logging

- **Overview:** Uploads the final compiled video to Google Drive and logs metadata into a Google Sheet.
- **Nodes Involved:**  
  - Upload: Save Final Video to Drive  
  - Log: Add Video Title & Link to Sheet  

- **Node Details:**

  - **Upload: Save Final Video to Drive**  
    - Type: `googleDrive`  
    - Role: Saves the final MP4 to Google Drive folder "AI generated Videos" with the form title as filename.  
    - Config: Drive "My Drive", folder ID for video folder, filename from form Title + ".mp4".  
    - Inputs: Binary video from Creatomate download  
    - Outputs: Metadata including webViewLink  
    - Credentials: Google Drive OAuth2  
    - Edge cases: Storage limits, permission errors.

  - **Log: Add Video Title & Link to Sheet**  
    - Type: `googleSheets`  
    - Role: Appends a new row with video Title and share link to Google Sheets for record keeping.  
    - Config: Append operation to Sheet1 of specified spreadsheet; columns "Title" and "Generated videos".  
    - Inputs: Form Title and video webViewLink from upload node  
    - Outputs: Confirmation of append operation  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: API quota limits, sheet permission issues.

---

#### 1.10 Workflow Documentation

- **Overview:** Provides a sticky note with a detailed workflow summary and step descriptions for user reference.
- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: `stickyNote`  
    - Role: Contains a comprehensive description of the workflow steps and logic blocks for easy understanding.  
    - Config: Large text block summarizing all major nodes and their functions.  
    - Inputs: None (static)  
    - Outputs: None  

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                             | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                         |
|----------------------------------|----------------------------|---------------------------------------------|--------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------|
| Form: User Submission            | formTrigger                | Receive user form inputs                     | -                              | Format into an Array                   | 🟡 Trigger: Form Submission - starts workflow on form fill with title, animals, style              |
| Format into an Array             | set                        | Format form animal inputs into array         | Form: User Submission          | Split: Loop Through Array              | 🟡 Format Inputs - prepares animal inputs as array                                                 |
| Split: Loop Through Array        | splitOut                   | Split array for iterative processing          | Format into an Array           | GPT-4: Create Cinematic Prompt         |                                                                                                   |
| GPT-4: Create Cinematic Prompt   | OpenAI (langchain)         | Generate cinematic image prompts               | Split: Loop Through Array      | Clean: Remove Line Breaks from Prompt  | 🟡 Generate Cinematic Prompts (OpenAI) - transform animals into cinematic prompts                   |
| Clean: Remove Line Breaks from Prompt | set                    | Remove newlines from GPT-4 prompt output      | GPT-4: Create Cinematic Prompt | Dumpling AI: Generate Image            |                                                                                                   |
| Dumpling AI: Generate Image      | httpRequest                | Generate AI images from prompts                | Clean: Remove Line Breaks      | GPT-4: Create motion prompt            | 🟡 Create AI Image (Dumpling AI) - generate high quality images                                    |
| GPT-4: Create motion prompt      | OpenAI (langchain)         | Convert image prompt to motion video prompt   | Dumpling AI: Generate Image    | Replicate.com: Create Motion Video     | 🟡 Motion Prompt for Video (OpenAI) - rewrite prompt to add motion elements                        |
| Replicate.com: Create Motion Video | httpRequest              | Generate animated motion videos from images   | GPT-4: Create motion prompt    | Wait: Leonardo Processing              | 🟡 Generate Motion Video (Leonardo) - turn images into short animated clips                        |
| Wait: Leonardo Processing        | wait                       | Pause for video rendering completion          | Replicate.com: Create Motion Video | Fetch: Download Motion Video         |                                                                                                   |
| Fetch: Download Motion Video     | httpRequest                | Retrieve generated motion video URLs           | Wait: Leonardo Processing      | Merge: Combine Videos & Audio Branch   |                                                                                                   |
| Limit: One Audio Track Per Run   | limit                      | Limit audio generation to one per run          | Fetch: Download Motion Video   | GPT-4: Generate Audio Prompt           | 🟡 Limit to one audio track per workflow run                                                      |
| GPT-4: Generate Audio Prompt     | OpenAI (langchain)         | Generate immersive audio prompt from style     | Limit: One Audio Track Per Run | ElevenLabs: Create Ambient Soundtrack  | 🟡 Generate Audio Prompt (OpenAI) - vivid sound prompt based on style                              |
| ElevenLabs: Create Ambient Soundtrack | httpRequest             | Generate ambient audio from prompt             | GPT-4: Generate Audio Prompt   | Upload: Save Audio to Google Drive     | 🟡 Create Ambient Sound (ElevenLabs) - generate 20-second audio                                  |
| Upload: Save Audio to Google Drive | googleDrive              | Upload generated audio file to Drive            | ElevenLabs: Create Ambient Soundtrack | Share: Make Audio Public           | 🟡 Upload Audio to Drive - save and prepare for sharing                                           |
| Share: Make Audio Public         | googleDrive                | Set audio file sharing permissions public      | Upload: Save Audio to Google Drive | Merge: Combine Videos & Audio Branch |                                                                                                   |
| Merge: Combine Videos & Audio Branch | merge                   | Combine all video items and audio item          | Fetch: Download Motion Video, Share: Make Audio Public | Format Motion Video URLs        | 🟡 Merge all videos and audio into one workflow branch                                            |
| Format Motion Video URLs         | code                       | Format video URLs for Creatomate API            | Merge: Combine Videos & Audio Branch | Creatomate: Combine Videos & Audio |                                                                                                   |
| Creatomate: Combine Videos & Audio | httpRequest              | Request final video rendering combining clips and audio | Format Motion Video URLs   | Wait: Creatomate Rendering             | 🟡 Merge Videos and Audio (Creatomate) - stitch into final cinematic video                        |
| Wait: Creatomate Rendering       | wait                       | Pause for final video rendering completion      | Creatomate: Combine Videos & Audio | Download: Final MP4 from Creatomate |                                                                                                   |
| Download: Final MP4 from Creatomate | httpRequest             | Download final rendered video                    | Wait: Creatomate Rendering     | Upload: Save Final Video to Drive      |                                                                                                   |
| Upload: Save Final Video to Drive | googleDrive              | Upload final video to Google Drive folder        | Download: Final MP4 from Creatomate | Log: Add Video Title & Link to Sheet | 🟡 Upload Final Video - save to Drive in AI generated Videos folder                             |
| Log: Add Video Title & Link to Sheet | googleSheets            | Log video metadata (title, link) to Google Sheets | Upload: Save Final Video to Drive | -                                  | 🟡 Log Output to Sheet - video title and share link for future access                             |
| Sticky Note                     | stickyNote                 | Documentation summary of workflow                 | -                              | -                                      | Contains detailed workflow overview and step summaries                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Type: `formTrigger`
   - Configure form titled "Content" with fields:
     - Title (required)
     - Country 1 (required)
     - Country 2 (required)
     - Country 3 (optional)
     - Country 4 (required)
   - This node triggers workflow on form submission.

2. **Add `set` Node to Format Array:**
   - Name: "Format into an Array"
   - Assign variable `animals` as array:
     ```
     [
       "{{ $json['animal 1'] }}",
       "{{ $json['animal 2'] }}",
       "{{ $json['animal 3'] }}",
       "{{ $json['animal 4'] }}"
     ]
     ```
   - Connect output of formTrigger to this node.

3. **Add `splitOut` Node:**
   - Name: "Split: Loop Through Array"
   - Configure to split on field `animals`.
   - Connect from "Format into an Array".

4. **Add OpenAI Langchain Node:**
   - Name: "GPT-4: Create Cinematic Prompt"
   - Model: GPT-4.1
   - Set system prompt to transform input (country or animal) into a cinematic image prompt with specified rules (muscular warrior, natural habitat, cinematic style, max 950 chars).
   - Input content uses each split array item.
   - Connect from "Split: Loop Through Array".

5. **Add `set` Node to Clean Prompt:**
   - Name: "Clean: Remove Line Breaks from Prompt"
   - Use expression to replace newlines in GPT output:
     ```
     {{ $json.message.content.replace(/\n/g, ' ').replace(/\\n/g, ' ') }}
     ```
   - Connect from GPT-4 cinematic prompt node.

6. **Add HTTP Request Node for Dumpling AI:**
   - Name: "Dumpling AI: Generate Image"
   - Method: POST
   - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`
   - Body (JSON):
     ```json
     {
       "model": "FLUX.1-pro",
       "input": {
         "prompt": "={{ $json.prompt }}"
       }
     }
     ```
   - Auth: HTTP Header Auth with Dumpling AI API key.
   - Connect from "Clean: Remove Line Breaks from Prompt".

7. **Add OpenAI Langchain Node to Create Motion Prompt:**
   - Name: "GPT-4: Create motion prompt"
   - Model: GPT-4.1
   - System prompt instructs rewriting image prompt into a short (1-2 sentences) motion prompt with natural movements.
   - Input: Prompt from Dumpling AI node output.
   - Connect from Dumpling AI node.

8. **Add HTTP Request Node for Replicate.com Leonardo AI:**
   - Name: "Replicate.com: Create Motion Video"
   - Method: POST
   - URL: `https://api.replicate.com/v1/models/wavespeedai/wan-2.1-i2v-480p/predictions`
   - Body (JSON):
     ```json
     {
       "input": {
         "image": "{{ $json.images[0].url }}",
         "prompt": "{{ $json.message.content }}"
       }
     }
     ```
   - Auth: HTTP Header Auth with Replicate.com API key.
   - Connect from GPT-4 motion prompt node.

9. **Add Wait Node:**
   - Name: "Wait: Leonardo Processing"
   - Wait duration: 60 seconds
   - Connect from Replicate.com node.

10. **Add HTTP Request Node to Fetch Motion Video:**
    - Name: "Fetch: Download Motion Video"
    - Method: GET
    - URL: Use prediction status URL from Replicate.com response.
    - Auth: Replicate.com API key.
    - Connect from wait node.

11. **Add Limit Node:**
    - Name: "Limit: One Audio Track Per Run"
    - Default limit to 1
    - Connect from "Fetch: Download Motion Video".

12. **Add OpenAI Langchain Node for Audio Prompt:**
    - Name: "GPT-4: Generate Audio Prompt"
    - Model: GPT-4.1
    - System prompt to generate a vivid sound prompt from user-provided style.
    - Input: Style field from form submission.
    - Connect from Limit node.

13. **Add HTTP Request Node for ElevenLabs:**
    - Name: "ElevenLabs: Create Ambient Soundtrack"
    - Method: POST
    - URL: `https://api.elevenlabs.io/v1/sound-generation`
    - Body parameters: `text` from GPT-4 audio prompt, `duration_seconds`=20
    - Auth: HTTP Header Auth with ElevenLabs API key.
    - Connect from GPT-4 audio prompt node.

14. **Add Google Drive Upload Node for Audio:**
    - Name: "Upload: Save Audio to Google Drive"
    - Folder: "Soundtrack" folder ID
    - Filename: Form Title + ".mp3"
    - Credentials: Google Drive OAuth2
    - Connect from ElevenLabs node.

15. **Add Google Drive Node to Share Audio Publicly:**
    - Name: "Share: Make Audio Public"
    - Operation: Share
    - Permissions: role "reader", type "anyone", allow file discovery true
    - Connect from Upload Audio node.

16. **Add Merge Node to Combine Video and Audio Branches:**
    - Name: "Merge: Combine Videos & Audio Branch"
    - Mode: Combine All
    - Inputs: From "Fetch: Download Motion Video" and "Share: Make Audio Public"
    - Connect accordingly.

17. **Add Code Node to Format Video URLs:**
    - Name: "Format Motion Video URLs"
    - JavaScript code to extract video URLs and metadata into array
    - Connect from Merge node.

18. **Add HTTP Request Node for Creatomate:**
    - Name: "Creatomate: Combine Videos & Audio"
    - Method: POST
    - URL: Creatomate API endpoint for rendering
    - Body: JSON with template ID and modifications including video URLs, audio link, and text overlays from form animals
    - Auth: Bearer token (set accordingly)
    - Connect from Code node.

19. **Add Wait Node for Creatomate Rendering:**
    - Name: "Wait: Creatomate Rendering"
    - Wait 60 seconds
    - Connect from Creatomate node.

20. **Add HTTP Request Node to Download Final Video:**
    - Name: "Download: Final MP4 from Creatomate"
    - Method: GET
    - URL: Rendered video URL from Creatomate node
    - Connect from Wait node.

21. **Add Google Drive Upload Node for Final Video:**
    - Name: "Upload: Save Final Video to Drive"
    - Folder: "AI generated Videos" folder ID
    - Filename: Form Title + ".mp4"
    - Credentials: Google Drive OAuth2
    - Connect from Download Final MP4 node.

22. **Add Google Sheets Node to Log Metadata:**
    - Name: "Log: Add Video Title & Link to Sheet"
    - Operation: Append to Sheet1
    - Columns: Title (form Title), Generated videos (Drive video link)
    - Credentials: Google Sheets OAuth2
    - Connect from Upload Final Video node.

23. **Add Sticky Note Node:**
    - Name: "Sticky Note"
    - Content: Detailed workflow description (copy from existing content)
    - Position for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow utilizes multiple AI services: OpenAI GPT-4 for text generation, Dumpling AI for image generation, Replicate.com Leonardo AI for motion video, and ElevenLabs for ambient audio generation.                            | AI Services Integration                                                                                              |
| Google Drive is used for storage and sharing of audio and video files, with public sharing enabled for audio files to be accessible by Creatomate.                                                                                | Google Drive Integration                                                                                             |
| Google Sheets is used as a logging database to keep track of generated videos by title and shareable link.                                                                                                                       | Google Sheets Integration                                                                                            |
| Creatomate is used to combine multiple video clips and an audio track into a final cinematic video with text overlays, leveraging a predefined template ID.                                                                        | Creatomate API for Video Editing                                                                                    |
| Wait nodes with fixed delays (60 seconds) are used to accommodate asynchronous AI processing times; these durations may need adjustment based on actual service response times to avoid premature requests or excessive delays.      | Workflow Timing Notes                                                                                                |
| API credentials for all external services must be configured carefully with correct scopes and permissions to prevent authentication errors.                                                                                     | Credential Configuration                                                                                             |
| Error handling is not explicitly implemented; failures in API calls, rate limits, or missing inputs may cause the workflow to halt or produce incomplete results. Consider adding error handling and retries for production use. | Recommendations for Stability                                                                                       |
| Form input fields must match exactly the expected names ("animal 1", etc.) to avoid runtime errors.                                                                                                                               | Input Validation Notes                                                                                               |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.