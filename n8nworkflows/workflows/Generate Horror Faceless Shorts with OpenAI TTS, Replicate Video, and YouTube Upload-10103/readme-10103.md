Generate Horror Faceless Shorts with OpenAI TTS, Replicate Video, and YouTube Upload

https://n8nworkflows.xyz/workflows/generate-horror-faceless-shorts-with-openai-tts--replicate-video--and-youtube-upload-10103


# Generate Horror Faceless Shorts with OpenAI TTS, Replicate Video, and YouTube Upload

### 1. Workflow Overview

This workflow automates the creation and publishing of viral short horror videos (“faceless shorts”) using AI-generated story beats, text-to-speech narration, AI-generated images and videos, and YouTube video upload. It is designed for content creators aiming to produce engaging horror shorts with minimal manual intervention.

The workflow logically divides into four main blocks:

- **1.1 Input Reception & Control Flow**  
  Receives chat commands to trigger different workflow branches: generate story idea, create video, publish to YouTube, or clean temporary files.

- **1.2 Story Generation and Parsing**  
  Uses OpenAI and LangChain nodes to generate an 8-beat horror story with a twist, parses the JSON, and logs it to Google Sheets for tracking and refinement.

- **1.3 Video Creation Pipeline**  
  For each story beat, generates narration with proper pacing, creates matching images, synthesizes videos, merges audio and video, uploads intermediate beat files to Google Drive, and finally concatenates all beats into a single video file.

- **1.4 Publishing and Cleanup**  
  Prepares YouTube upload, uploads the final video, updates Google Sheets status, and optionally cleans up temporary files from Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Control Flow

**Overview:**  
This block listens for chat messages that control the workflow’s main actions based on keywords.

**Nodes Involved:**  
- When chat message received  
- Switch

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Entry point webhook that listens for chat commands (`idea`, `create`, `publish`, `clean drive`).  
  - Configuration: No special params; webhook triggers workflow on message.  
  - Outputs to Switch node.  
  - Failure modes: webhook down, invalid inputs.

- **Switch**  
  - Type: Switch control node  
  - Role: Routes workflow based on `$json.chatInput` string equality to four outputs: Generate Story Idea, Create Uploadable Video, Publish to YouTube, Clean Files in Drive.  
  - Configured with exact string matches (`idea`, `create`, `publish`, `clean drive`).  
  - Failure: unknown commands lead to no output; no default case specified.

---

#### 2.2 Story Generation and Parsing

**Overview:**  
Generates an 8-beat viral horror story with a surprise twist, parses its JSON output, and logs it to Google Sheets. This block ensures story quality and structured data for downstream processing.

**Nodes Involved:**  
- Story Idea Generator  
- Story Beat Generator  
- Story Idea Parser  
- Google Sheet Idea Log  
- Get Story Idea

**Node Details:**

- **Story Idea Generator**  
  - Type: LangChain Agent  
  - Role: Sends a prompt to generate a JSON structured horror story with 8 beats, YouTube title, description.  
  - Configuration: Strong system message enforcing JSON-only output, 8 beats, pacing, suspense, readability.  
  - Output: JSON with story beats and YouTube metadata.  
  - Failure modes: API limits, malformed JSON, timeout.

- **Story Beat Generator**  
  - Type: LangChain OpenAI Chat  
  - Role: (Appears connected but no detailed usage in JSON, likely a leftover or supporting node.)  
  - Uses GPT-4o-mini model with temperature 0.7 and penalties to generate text.

- **Story Idea Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the JSON output from Story Idea Generator, auto-fixes JSON errors if possible.  
  - Outputs structured JSON for beats and YouTube fields.  
  - Failure modes: parsing errors, invalid JSON structure.

- **Google Sheet Idea Log**  
  - Type: Google Sheets Append  
  - Role: Logs the generated story beats and metadata with timestamp and status "done" to a Google Sheet for tracking.  
  - Requires Google Sheets OAuth2 credentials.  
  - Failure modes: API quota, credential errors, sheet unavailable.

- **Get Story Idea**  
  - Type: Google Sheets Read  
  - Role: Reads stories with status "done" from the Google Sheet for use in video creation.  
  - Failure: read errors, empty sheet.

---

#### 2.3 Video Creation Pipeline

**Overview:**  
Processes each story beat to generate narration audio, AI-generated images, AI-generated videos, merges audio-video using FFmpeg, uploads beat videos to Google Drive, and finally concatenates all beats into a single final video.

**Nodes Involved:**  
- Narration Prompt Generator  
- Narration Output Parser  
- Image Prompt Generator  
- Image Output Parser  
- Check For Already Created Beats  
- Handle 0 Files  
- Create Beat Inputs  
- If Beats Remaining  
- Loop Over Items  
- 🎨 Image Generator (Replicate API)  
- 🎨 Video Generator (Replicate API)  
- HTTP Request (to fetch video)  
- Save Beat Image Locally  
- Generate Beat Audio (OpenAI TTS)  
- Save Speech Locally  
- Video Audio Merge Command (Code node generating FFmpeg command)  
- Run FFmpeg to Merge Media  
- Read Beat File  
- Upload Beat File (Google Drive)  
- Search Beat Files  
- Download Beat File  
- Write Beat File to Disk  
- Generate Final Video  
- Read Final Video from Disk  
- Upload Final Video (Google Drive)  
- Update Status to Ready (Google Sheets)  
- Wait (delay to ensure file readiness)

**Node Details:**

- **Narration Prompt Generator**  
  - Type: LangChain Agent  
  - Role: Formats horror story beats to add pacing (pauses, ellipses, emphasis) without changing text for TTS narration.  
  - Outputs exactly 8 narratives with pause marks in JSON.  
  - Failure: malformed input, API errors.

- **Narration Output Parser**  
  - Type: LangChain Output Parser  
  - Parses the narration JSON for downstream use.

- **Image Prompt Generator**  
  - Type: LangChain Agent  
  - Role: Extracts character traits (age, gender, complexion, hairstyle, clothing) consistently from story beats for stable image generation prompts.  
  - Outputs JSON with "character" description and negative prompts to block unwanted artifacts.  
  - Failure: parsing errors.

- **Image Output Parser**  
  - Type: LangChain Output Parser  
  - Parses the image prompt JSON.

- **Check For Already Created Beats**  
  - Type: Google Drive Search  
  - Role: Checks Google Drive folder for existing beat video files matching pattern (e.g., horrorfile*_1.mp4) to avoid redundant processing.  
  - Failure: Drive API errors.

- **Handle 0 Files**  
  - Type: Code node  
  - Ensures workflow continues even if no existing files are found.

- **Create Beat Inputs**  
  - Type: Code node  
  - Combines story beats, narration, and image prompts into per-beat objects with filenames, seeds, and skip flags if files already exist.  
  - Filters out empty beats and skips already created files.  
  - Failure: data inconsistencies, missing inputs.

- **If Beats Remaining**  
  - Type: If condition  
  - Checks if less than 8 beats exist and routes accordingly.

- **Loop Over Items**  
  - Type: Split in Batches  
  - Loops over each beat input for sequential processing.

- **🎨 Image Generator**  
  - Type: HTTP Request (Replicate API)  
  - Sends image prompt to a text-to-image AI model (black-forest-labs / flux-schnell) for oil painting style horror images.  
  - Uses seed for reproducibility and safety filters.  
  - Failure: API limits, invalid prompts.

- **🎨 Video Generator**  
  - Type: HTTP Request (Replicate API)  
  - Generates short video clips from images using an AI model (wan-video) with prompts and negative prompts.  
  - Waits synchronously for completion.  
  - Failure: API timeouts, invalid inputs.

- **HTTP Request**  
  - Type: HTTP Request  
  - Fetches generated video URL from prior node for download.

- **Save Beat Image Locally**  
  - Type: Read/Write File  
  - Saves generated image to local disk with beat file name.

- **Generate Beat Audio**  
  - Type: OpenAI TTS Node  
  - Generates narration audio from pacing-enhanced text using OpenAI’s TTS model (tts-1-hd).  
  - Failure: API limits, audio encoding errors.

- **Save Speech Locally**  
  - Type: Read/Write File  
  - Saves generated audio locally as MP3.

- **Video Audio Merge Command**  
  - Type: Code node  
  - Constructs FFmpeg command to merge beat video and generated audio with audio tempo adjustment and padding.  
  - Failure: command construction errors.

- **Run FFmpeg to Merge Media**  
  - Type: Execute Command  
  - Executes FFmpeg command to produce combined video with audio.  
  - Failure: FFmpeg not installed, runtime errors.

- **Read Beat File**  
  - Type: Read/Write File  
  - Reads merged beat video file from disk.

- **Upload Beat File**  
  - Type: Google Drive Upload  
  - Uploads merged beat video to Google Drive folder.

- **Search Beat Files**  
  - Type: Google Drive Search  
  - Searches for beat videos in Drive for concatenation.

- **Download Beat File**  
  - Type: Google Drive Download  
  - Downloads beat video files from Drive.

- **Write Beat File to Disk**  
  - Type: Read/Write File  
  - Writes downloaded beat files locally for final merging.

- **Generate Final Video**  
  - Type: Execute Command  
  - Runs FFmpeg command concatenating all beat videos with audio and overlay text “Midnight Cuts” using drawtext filter.  
  - Produces final combined video `horrorfile_short_final.mp4`.  
  - Failure: FFmpeg errors, missing input files.

- **Read Final Video from Disk**  
  - Type: Read/Write File  
  - Reads final video file for upload.

- **Upload Final Video**  
  - Type: Google Drive Upload  
  - Uploads final compiled video to Google Drive.

- **Update Status to Ready**  
  - Type: Google Sheets Update  
  - Updates the story row status to “ready” in Google Sheets after video creation.  
  - Failure: sheet update errors.

- **Wait**  
  - Type: Wait node  
  - Adds delay to ensure files are ready for next steps.

---

#### 2.4 Publishing and Cleanup

**Overview:**  
Handles YouTube video upload preparation, uploads the final video, updates Google Sheets with the YouTube URL and status, and provides cleanup of temporary files from Google Drive.

**Nodes Involved:**  
- Get YouTube Title and Description  
- Prepare YouTube Upload  
- Read Video for Upload from Disk  
- YouTube Video Upload  
- Update YouTube Url  
- Temporary Files Cleanup3  
- Search Temporary Files to Delete  
- Delete Temporary Files

**Node Details:**

- **Get YouTube Title and Description**  
  - Type: Google Sheets Read  
  - Reads the latest story’s YouTube title and description to use for video upload metadata.  
  - Failure: API errors, empty data.

- **Prepare YouTube Upload**  
  - Type: HTTP Request  
  - Initiates a resumable YouTube video upload session with metadata (title, description, privacy, language, and tags).  
  - Uses YouTube OAuth2 credentials.  
  - Failure: auth errors, quota limits.

- **Read Video for Upload from Disk**  
  - Type: Read/Write File  
  - Reads the final video file (`horrorfile_short_final.mp4`) from disk for uploading.

- **YouTube Video Upload**  
  - Type: HTTP Request  
  - Uploads video binary data to YouTube using the resumable upload URL from previous node.  
  - Sends with content type `video/webm`.  
  - Failure: upload interruption, token expiration.

- **Update YouTube Url**  
  - Type: Google Sheets Update  
  - Updates Google Sheet with the YouTube Shorts URL and marks status as "published".  
  - Failure: sheet update errors.

- **Temporary Files Cleanup3**  
  - Type: Execute Command  
  - Runs shell command to remove temporary local files matching pattern `12*.png`, `12*.mp4`, etc.  
  - Failure: permissions, missing files.

- **Search Temporary Files to Delete**  
  - Type: Google Drive Search  
  - Searches Google Drive folder for temporary files matching pattern `horrorfile*.mp4` for deletion.

- **Delete Temporary Files**  
  - Type: Google Drive Delete File  
  - Deletes each found temporary file from Google Drive.  
  - Failure: permission denied, API errors.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                                    | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                               |
|------------------------------|----------------------------------|---------------------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------|
| When chat message received   | LangChain Chat Trigger            | Entry point, receives chat commands                | -                               | Switch                          |                                                                                                           |
| Switch                      | Switch                           | Routes workflow based on chat command              | When chat message received       | Story Idea Generator, Temporary Files Cleanup, Get YouTube Title and Description, Search Temporary Files to Delete |                                                                                                           |
| Story Idea Generator        | LangChain Agent                  | Generates 8-beat horror story JSON                  | Switch (Generate Story Idea)     | Story Idea Parser               |                                                                                                           |
| Story Beat Generator        | LangChain LM Chat OpenAI         | (Auxiliary LM node)                                 | -                               | Story Idea Generator, Story Idea Parser |                                                                                                           |
| Story Idea Parser           | LangChain Output Parser           | Parses story JSON                                   | Story Idea Generator             | Google Sheet Idea Log           |                                                                                                           |
| Google Sheet Idea Log       | Google Sheets Append              | Logs generated story beats                          | Story Idea Parser                | -                              |                                                                                                           |
| Get Story Idea              | Google Sheets Read                | Reads completed story beats                         | Switch (Create Uploadable Video) | Narration Prompt Generator      |                                                                                                           |
| Narration Prompt Generator  | LangChain Agent                  | Adds pacing to narration text                       | Get Story Idea                  | Narration Output Parser         |                                                                                                           |
| Narration Output Parser     | LangChain Output Parser           | Parses paced narration JSON                         | Narration Prompt Generator       | Create Beat Inputs             |                                                                                                           |
| Image Prompt Generator      | LangChain Agent                  | Extracts character traits for image generation     | Get Story Idea                  | Image Output Parser            |                                                                                                           |
| Image Output Parser         | LangChain Output Parser           | Parses image prompt JSON                            | Image Prompt Generator           | Check For Already Created Beats |                                                                                                           |
| Check For Already Created Beats | Google Drive Search             | Checks for existing beat videos to skip            | Image Output Parser             | Handle 0 Files                 |                                                                                                           |
| Handle 0 Files             | Code                             | Ensures flow continues even if no existing files   | Check For Already Created Beats  | Create Beat Inputs             |                                                                                                           |
| Create Beat Inputs         | Code                             | Prepares beat data objects with file names and skip flags | Handle 0 Files, Narration Output Parser, Image Output Parser, Get Story Idea | If Beats Remaining           |                                                                                                           |
| If Beats Remaining         | If                               | Branches if all beats are ready                     | Create Beat Inputs              | Loop Over Items, Generate Final Video Clip |                                                                                                           |
| Loop Over Items            | Split In Batches                 | Iterates over beats for processing                  | If Beats Remaining              | Generate Final Video Clip, 🎨 Image Generator |                                                                                                           |
| 🎨 Image Generator          | HTTP Request (Replicate API)     | Generates horror style images from prompts          | Loop Over Items                 | 🎨 Video Generator             |                                                                                                           |
| 🎨 Video Generator          | HTTP Request (Replicate API)     | Generates short video from images                    | 🎨 Image Generator             | HTTP Request                   |                                                                                                           |
| HTTP Request               | HTTP Request                     | Fetches generated video URL                          | 🎨 Video Generator             | Save Beat Image Locally        |                                                                                                           |
| Save Beat Image Locally    | Read/Write File                  | Saves generated image locally                        | HTTP Request                   | Wait                          |                                                                                                           |
| Wait                      | Wait                            | Delays to ensure file readiness                      | Save Beat Image Locally         | Generate Beat Audio             |                                                                                                           |
| Generate Beat Audio        | OpenAI TTS                      | Generates narration audio from paced text           | Wait                          | Save Speech Locally             |                                                                                                           |
| Save Speech Locally        | Read/Write File                  | Saves narration audio locally                        | Generate Beat Audio             | Video Audio Merge Command       |                                                                                                           |
| Video Audio Merge Command  | Code                            | Generates FFmpeg command to merge video and audio   | Save Speech Locally             | Run FFmpeg to Merge Media       |                                                                                                           |
| Run FFmpeg to Merge Media  | Execute Command                 | Executes FFmpeg to merge audio and video            | Video Audio Merge Command       | Read Beat File                 |                                                                                                           |
| Read Beat File             | Read/Write File                  | Reads merged beat video                              | Run FFmpeg to Merge Media       | Upload Beat File               |                                                                                                           |
| Upload Beat File           | Google Drive Upload              | Uploads beat video files to Google Drive            | Read Beat File                 | Loop Over Items                |                                                                                                           |
| Generate Final Video Clip  | Execute Command                 | Prepares concatenation of beat videos                | Loop Over Items                 | Search Beat Files              |                                                                                                           |
| Search Beat Files          | Google Drive Search             | Finds beat videos for final video generation         | Generate Final Video Clip       | Download Beat File             |                                                                                                           |
| Download Beat File         | Google Drive Download           | Downloads beat files from Google Drive                | Search Beat Files               | Write Beat File to Disk        |                                                                                                           |
| Write Beat File to Disk    | Read/Write File                  | Writes downloaded beat files locally                  | Download Beat File              | Generate Final Video           |                                                                                                           |
| Generate Final Video       | Execute Command                 | Concatenates all beats into final video with overlay | Write Beat File to Disk         | Read Final Video from Disk     |                                                                                                           |
| Read Final Video from Disk | Read/Write File                  | Reads final video file                               | Generate Final Video            | Upload Final Video             |                                                                                                           |
| Upload Final Video         | Google Drive Upload              | Uploads final video to Google Drive                   | Read Final Video from Disk      | Update Status to Ready         |                                                                                                           |
| Update Status to Ready     | Google Sheets Update            | Marks story status as “ready” after video creation   | Upload Final Video             | -                            |                                                                                                           |
| Get YouTube Title and Description | Google Sheets Read            | Reads YouTube metadata for upload                      | Switch (Publish to YouTube)     | Prepare YouTube Upload         |                                                                                                           |
| Prepare YouTube Upload     | HTTP Request                   | Initiates YouTube resumable upload session            | Get YouTube Title and Description | Read Video for Upload from Disk |                                                                                                           |
| Read Video for Upload from Disk | Read/Write File                  | Reads final video for upload                           | Prepare YouTube Upload          | YouTube Video Upload           |                                                                                                           |
| YouTube Video Upload       | HTTP Request                   | Uploads video binary to YouTube                        | Read Video for Upload from Disk | Update YouTube Url             |                                                                                                           |
| Update YouTube Url         | Google Sheets Update            | Updates sheet with YouTube video URL and status       | YouTube Video Upload           | Temporary Files Cleanup3       |                                                                                                           |
| Temporary Files Cleanup3   | Execute Command                 | Deletes local temporary files                           | Update YouTube Url             | -                            |                                                                                                           |
| Search Temporary Files to Delete | Google Drive Search             | Finds temporary files for deletion                      | Switch (Clean Files in Drive)   | Delete Temporary Files         |                                                                                                           |
| Delete Temporary Files     | Google Drive Delete File       | Deletes temporary files from Google Drive              | Search Temporary Files to Delete | -                            |                                                                                                           |

**Sticky Notes:**

- Step 1: Generate Story Beats  
  - Setup Google Sheets; OpenAI credentials  
  - Type `idea` to trigger story generation, status updates to `done`  
  - Edit sheet to refine story  
  - Avoid escape characters for smooth video creation  
  - Keep beats under 20 words for pacing

- Step 2: Create Video  
  - Setup Google Drive folder, Drive and AI credentials  
  - Verify 8 beats exist, only one line with `done` status  
  - Type `create` to trigger video creation, status updates to `ready`  
  - Review output, delete beats to regenerate if needed  
  - Restart on API timeout by typing `create` (previous beats saved)

- Step 3: Publish Video  
  - Setup YouTube credentials  
  - Ensure only one row with status `ready`  
  - Type `publish` to upload video, sheet status updates to `published`  
  - Video uploaded as Private, change to Public manually

- Step 4: Remove Temporary Files  
  - Type `clean drive` to delete temporary files from Google Drive after publishing

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Create a LangChain Chat Trigger node named `When chat message received`.  
   - Configure webhook with no special parameters.  
   - Connect output to a `Switch` node.

2. **Create Switch Node:**  
   - Add a Switch node named `Switch`.  
   - Add four rules to match `$json.chatInput` exactly:  
     - "idea" → Output "Generate Story Idea"  
     - "create" → Output "Create Uploadable Video"  
     - "publish" → Output "Publish to YouTube"  
     - "clean drive" → Output "Clean Files in Drive"

3. **Story Generation Block:**  
   - Add a LangChain Agent node named `Story Idea Generator`.  
     - Prompt: Request exactly 8-beat horror story JSON with YouTube title & description.  
     - System message enforcing JSON-only output, pacing, suspense, reading level 4th grade.  
   - Add a LangChain Output Parser node named `Story Idea Parser`.  
     - Use JSON schema matching the 8 beats + YouTube fields.  
   - Connect `Story Idea Generator` output to `Story Idea Parser`.  
   - Add a Google Sheets Append node named `Google Sheet Idea Log` to append results.  
     - Connect `Story Idea Parser` output to this node.  
     - Configure with Google Sheets OAuth2 credentials and target sheet.  
   - Connect Switch "Generate Story Idea" output to `Story Idea Generator`.

4. **Story Retrieval for Video Creation:**  
   - Add a Google Sheets Read node named `Get Story Idea`.  
     - Filter rows where status = "done".  
     - Connect Switch "Create Uploadable Video" output to this node.

5. **Narration Preparation:**  
   - Add LangChain Agent node `Narration Prompt Generator`.  
     - Input: beats from `Get Story Idea`.  
     - System message instructing to add pacing (pauses) without changing text.  
   - Add LangChain Output Parser node `Narration Output Parser` to parse narration JSON.  
   - Connect `Get Story Idea` → `Narration Prompt Generator` → `Narration Output Parser`.

6. **Image Prompt Preparation:**  
   - Add LangChain Agent node `Image Prompt Generator`.  
     - Input: beats from `Get Story Idea`.  
     - System message to extract character traits for image prompt consistency.  
   - Add LangChain Output Parser node `Image Output Parser`.  
   - Connect `Get Story Idea` → `Image Prompt Generator` → `Image Output Parser`.

7. **Check Existing Beat Files:**  
   - Add Google Drive Search node `Check For Already Created Beats`.  
     - Search for files in Google Drive folder matching `horrorfile*_1.mp4`.  
   - Add Code node `Handle 0 Files` to ensure flow continues if no files found.  
   - Connect `Image Output Parser` → `Check For Already Created Beats` → `Handle 0 Files`.

8. **Create Beat Inputs:**  
   - Add Code node `Create Beat Inputs`.  
     - Combine story beats, narration, image prompts, assign file names and skip flags if files exist.  
   - Connect `Handle 0 Files`, `Narration Output Parser`, `Image Output Parser`, and `Get Story Idea` as inputs.  
   - Connect output to `If Beats Remaining`.

9. **Conditional Beat Processing:**  
   - Add If node `If Beats Remaining`.  
     - Condition: number of existing beats not equal to 8.  
   - Connect `Create Beat Inputs` → `If Beats Remaining`.

10. **Loop Over Beat Inputs:**  
    - Add SplitInBatches node `Loop Over Items` to process each beat sequentially.  
    - Connect `If Beats Remaining` true output to `Loop Over Items`.

11. **Image Generation:**  
    - Add HTTP Request node `🎨 Image Generator` calling Replicate API for text-to-image.  
    - Use character description and negative prompt from `Create Beat Inputs`.  
    - Connect `Loop Over Items` → `🎨 Image Generator`.

12. **Video Generation:**  
    - Add HTTP Request node `🎨 Video Generator` calling Replicate API for video generation.  
    - Input: image from previous node, prompt and negative prompt.  
    - Connect `🎨 Image Generator` → `🎨 Video Generator`.

13. **Fetch Video URL:**  
    - Add HTTP Request node `HTTP Request` to fetch generated video URL.  
    - Connect `🎨 Video Generator` → `HTTP Request`.

14. **Save Image Locally:**  
    - Add Read/Write File node `Save Beat Image Locally` to save image for reference.  
    - Connect `HTTP Request` → `Save Beat Image Locally`.

15. **Wait for Stability:**  
    - Add Wait node `Wait` to allow file readiness.  
    - Connect `Save Beat Image Locally` → `Wait`.

16. **Generate Beat Audio:**  
    - Add OpenAI TTS node `Generate Beat Audio` with input paced narration from `Create Beat Inputs`.  
    - Connect `Wait` → `Generate Beat Audio`.

17. **Save Speech Locally:**  
    - Add Read/Write File node `Save Speech Locally` to save generated MP3 audio.  
    - Connect `Generate Beat Audio` → `Save Speech Locally`.

18. **Generate FFmpeg Command:**  
    - Add Code node `Video Audio Merge Command`.  
    - Generates command to merge video and audio files with tempo adjustment.  
    - Connect `Save Speech Locally` → `Video Audio Merge Command`.

19. **Run FFmpeg Merge:**  
    - Add Execute Command node `Run FFmpeg to Merge Media` to run the command.  
    - Connect `Video Audio Merge Command` → `Run FFmpeg to Merge Media`.

20. **Read Merged File:**  
    - Add Read/Write File node `Read Beat File`.  
    - Connect `Run FFmpeg to Merge Media` → `Read Beat File`.

21. **Upload Beat to Drive:**  
    - Add Google Drive Upload node `Upload Beat File`.  
    - Connect `Read Beat File` → `Upload Beat File`.

22. **Loop Over Items Completion:**  
    - Connect `Upload Beat File` → `Loop Over Items` to process next beat until completion.

23. **Final Video Clip Generation:**  
    - Add Execute Command node `Generate Final Video Clip` to trigger search for beat files.  
    - Connect `If Beats Remaining` false output and `Loop Over Items` completion to this node.

24. **Search Beat Files:**  
    - Add Google Drive Search node `Search Beat Files` for all beat mp4 files.

25. **Download Beat Files:**  
    - Add Google Drive Download node `Download Beat File` for each found file.

26. **Write Beat Files Locally:**  
    - Add Read/Write File node `Write Beat File to Disk`.

27. **Concatenate Final Video:**  
    - Add Execute Command node `Generate Final Video` running FFmpeg concat and overlay text.

28. **Read Final Video:**  
    - Add Read/Write File node `Read Final Video from Disk`.

29. **Upload Final Video:**  
    - Add Google Drive Upload node `Upload Final Video`.

30. **Update Google Sheet Status:**  
    - Add Google Sheets Update node `Update Status to Ready` marking story as ready.

31. **YouTube Upload Branch:**  
    - Add Google Sheets Read node `Get YouTube Title and Description`.  
    - Add HTTP Request node `Prepare YouTube Upload` to initiate resumable upload.  
    - Add Read/Write File node `Read Video for Upload from Disk`.  
    - Add HTTP Request node `YouTube Video Upload` to upload video binary.  
    - Add Google Sheets Update node `Update YouTube Url` to mark published.  
    - Connect Switch "Publish to YouTube" output to this chain.

32. **Cleanup Temporary Files Branch:**  
    - Add Google Drive Search node `Search Temporary Files to Delete` for temp files.  
    - Add Google Drive Delete node `Delete Temporary Files` to remove them.  
    - Connect Switch "Clean Files in Drive" output here.

33. **Temporary Local File Cleanup:**  
    - Add Execute Command node `Temporary Files Cleanup` and `Temporary Files Cleanup3` to remove temporary local files.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates viral horror short story creation with AI and YouTube upload                               | Main use case                                                                                       |
| Requires Google Sheets setup with columns for beats, YouTube title, description, and status                    | Google Sheets URL and sheet ID must be configured                                                  |
| Requires Google Drive folder for temporary and final video file storage                                       | Google Drive Folder ID required                                                                    |
| Requires credentials setup for OpenAI, Google Sheets OAuth2, Google Drive OAuth2, YouTube OAuth2, Replicate API | Credential IDs and names must be replaced with user-specific ones in node credential sections       |
| Stable, reproducible image and video generation using Replicate’s models                                      | Models: black-forest-labs/flux-schnell for images, wan-video/wan-2.2-i2v-fast for videos            |
| FFmpeg must be installed and accessible on the n8n host environment                                          | Used for merging audio/video and final concatenation                                              |
| Text-to-Speech uses OpenAI’s TTS model `tts-1-hd`                                                            | Requires OpenAI API key with TTS access                                                           |
| Workflow controlled by chat commands: `idea`, `create`, `publish`, `clean drive`                              | Commands trigger respective workflow branches                                                     |
| Workflow designed to handle partial failures gracefully and supports re-run without losing progress           | E.g., skipping already created beats, retry on API timeout                                        |
| Video overlay “Midnight Cuts” uses Creepster font, ensure font file path in FFmpeg command is correct        | Font file path: `/home/node/.fonts/Creepster-Regular.ttf` (adjust if needed)                       |
| YouTube videos uploaded as Private initially, manual change to Public recommended after verification          |                                                                                                    |
| For detailed AI prompt design, refer to node system messages embedded in LangChain Agent nodes                |                                                                                                    |
| Workflow sticky notes provide operational instructions and best practices                                     | Visible in workflow canvas for user guidance                                                      |

---

**Disclaimer:**  
The provided text originates exclusively from an n8n automation workflow. It complies strictly with current content policies and contains no illegal or offensive elements. All data processed is legal and publicly available.