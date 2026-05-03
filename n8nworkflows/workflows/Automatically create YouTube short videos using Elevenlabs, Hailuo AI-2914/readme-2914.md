Automatically create YouTube short videos using Elevenlabs, Hailuo AI

https://n8nworkflows.xyz/workflows/automatically-create-youtube-short-videos-using-elevenlabs--hailuo-ai-2914


# Automatically create YouTube short videos using Elevenlabs, Hailuo AI

### 1. Workflow Overview

This workflow automates the creation of YouTube short videos by leveraging multiple AI services and cloud integrations. It is designed for marketers, content creators, and businesses aiming to streamline video production from script generation to final video upload and tracking.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Script Generation:** Starts with manual trigger or input, defines video category, and generates a transcript script using AI.
- **1.2 Voiceover Creation:** Converts the generated script into natural-sounding audio using ElevenLabs and manages audio storage.
- **1.3 Image Generation & Processing:** Creates image prompts, generates images, converts them to base64, and stores them.
- **1.4 Video Creation & Animation:** Uses Hailuo AI to animate images into video, monitors processing status, downloads, and stores the video.
- **1.5 Video Assembly & Captioning:** Combines audio and video, monitors video creation progress, retrieves captions, and prepares final video metadata.
- **1.6 Data Management & Distribution:** Uploads video metadata and URLs to Google Sheets for tracking and facilitates multi-platform sharing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Script Generation

- **Overview:** This block initiates the workflow, sets the video category, and generates the video transcript using AI services.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Video Catagory (Set)  
  - Generate Transcript (HTTP Request)  
  - Fetch Elevenlabs (HTTP Request)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution during testing  
    - Config: Disabled by default; triggers the workflow manually  
    - Inputs: None  
    - Outputs: Video Catagory  
    - Edge Cases: None; disabled by default

  - **Video Catagory**  
    - Type: Set  
    - Role: Defines or sets the video category or topic parameters for the script generation  
    - Config: Sets static or dynamic values for video category  
    - Inputs: Manual Trigger  
    - Outputs: Generate Transcript  
    - Edge Cases: Missing or invalid category could affect transcript relevance

  - **Generate Transcript**  
    - Type: HTTP Request  
    - Role: Calls an AI API (likely OpenAI or Together AI) to generate a video script/transcript based on category  
    - Config: HTTP POST with prompt parameters, API keys configured in credentials  
    - Inputs: Video Catagory  
    - Outputs: Fetch Elevenlabs  
    - Edge Cases: API rate limits, network errors, invalid prompts

  - **Fetch Elevenlabs**  
    - Type: HTTP Request  
    - Role: Sends the generated transcript to ElevenLabs API to create voiceover audio  
    - Config: HTTP POST with ElevenLabs API key, voice parameters  
    - Inputs: Generate Transcript  
    - Outputs: Save Audio  
    - Edge Cases: Authentication errors, audio generation failures

---

#### 2.2 Voiceover Creation

- **Overview:** Converts text transcript into audio, downloads and saves audio to cloud storage, and maps public URLs for further processing.
- **Nodes Involved:**  
  - Save Audio (Google Cloud Storage)  
  - Map Public Link (Set)  
  - Download Audio (HTTP Request)  
  - Open AI Whisper (HTTP Request)  
  - Combine Transcript (Code)  
  - Create a list of Image Text (Code)  
  - Split Image (SplitOut)

- **Node Details:**

  - **Save Audio**  
    - Type: Google Cloud Storage  
    - Role: Uploads generated audio file to Google Cloud Storage bucket  
    - Config: Bucket name, file path, credentials for GCP  
    - Inputs: Fetch Elevenlabs  
    - Outputs: Map Public Link  
    - Edge Cases: Storage permission errors, upload failures

  - **Map Public Link**  
    - Type: Set  
    - Role: Creates a publicly accessible URL for the saved audio file  
    - Config: Sets URL based on storage bucket and file path  
    - Inputs: Save Audio  
    - Outputs: Download Audio  
    - Edge Cases: Incorrect URL formatting

  - **Download Audio**  
    - Type: HTTP Request  
    - Role: Downloads the audio file from the public URL for further processing  
    - Config: HTTP GET request to public audio URL  
    - Inputs: Map Public Link  
    - Outputs: Open AI Whisper  
    - Edge Cases: URL expiration, network errors

  - **Open AI Whisper**  
    - Type: HTTP Request  
    - Role: Sends audio to OpenAI Whisper API for transcription or enhancement  
    - Config: HTTP POST with audio data, API key  
    - Inputs: Download Audio  
    - Outputs: Combine Transcript, Create a list of Image Text  
    - Edge Cases: API errors, transcription inaccuracies

  - **Combine Transcript**  
    - Type: Code  
    - Role: Merges or processes transcript data for subsequent steps  
    - Config: Custom JavaScript code to combine text segments  
    - Inputs: Open AI Whisper  
    - Outputs: Merge1  
    - Edge Cases: Code errors, malformed input data

  - **Create a list of Image Text**  
    - Type: Code  
    - Role: Extracts or formats text snippets to be used as image prompts  
    - Config: Custom JavaScript code  
    - Inputs: Open AI Whisper  
    - Outputs: Split Image  
    - Edge Cases: Empty or invalid text arrays

  - **Split Image**  
    - Type: SplitOut  
    - Role: Splits the list of image text prompts into individual items for parallel processing  
    - Config: Default split configuration  
    - Inputs: Create a list of Image Text  
    - Outputs: Generate Image Prompt  
    - Edge Cases: Empty input array

---

#### 2.3 Image Generation & Processing

- **Overview:** Generates image prompts, creates images, converts images to base64, and saves them to cloud storage.
- **Nodes Involved:**  
  - Generate Image Prompt (HTTP Request)  
  - Split Prompt (SplitOut)  
  - Get Image Base 64 (HTTP Request)  
  - Convert Base64 to Data (ConvertToFile)  
  - Save Image (Google Cloud Storage)  
  - Limit only first Image to video (Limit)

- **Node Details:**

  - **Generate Image Prompt**  
    - Type: HTTP Request  
    - Role: Calls AI service (Together AI or similar) to generate image prompts based on text snippets  
    - Config: HTTP POST with prompt data, API key  
    - Inputs: Split Image  
    - Outputs: Split Prompt  
    - Edge Cases: API failures, invalid prompt generation

  - **Split Prompt**  
    - Type: SplitOut  
    - Role: Splits generated prompts for individual image generation requests  
    - Config: Default split  
    - Inputs: Generate Image Prompt  
    - Outputs: Get Image Base 64  
    - Edge Cases: Empty or malformed prompt list

  - **Get Image Base 64**  
    - Type: HTTP Request  
    - Role: Requests image generation API to create images and return base64 encoded images  
    - Config: HTTP POST with prompt, API key  
    - Inputs: Split Prompt  
    - Outputs: Convert Base64 to Data  
    - Edge Cases: API errors, image generation failures

  - **Convert Base64 to Data**  
    - Type: ConvertToFile  
    - Role: Converts base64 image data into file format suitable for storage  
    - Config: Output file type (e.g., PNG or JPEG)  
    - Inputs: Get Image Base 64  
    - Outputs: Save Image  
    - Edge Cases: Conversion errors, invalid base64 data

  - **Save Image**  
    - Type: Google Cloud Storage  
    - Role: Uploads image files to cloud storage bucket  
    - Config: Bucket name, path, credentials  
    - Inputs: Convert Base64 to Data  
    - Outputs: Limit only first Image to video  
    - Edge Cases: Upload failures, permission issues

  - **Limit only first Image to video**  
    - Type: Limit  
    - Role: Restricts processing to only the first image for video creation  
    - Config: Limit count = 1  
    - Inputs: Save Image  
    - Outputs: Hailuo AI  
    - Edge Cases: No images available

---

#### 2.4 Video Creation & Animation

- **Overview:** Animates the selected image into a video using Hailuo AI, monitors processing status, downloads, and saves the video.
- **Nodes Involved:**  
  - Hailuo AI (HTTP Request)  
  - Wait Hailou done process (Wait)  
  - Get Hailuo Status (HTTP Request)  
  - If1 (If)  
  - Get Hailuo Download Link (HTTP Request)  
  - Download Video (HTTP Request)  
  - Save Video (Google Cloud Storage)

- **Node Details:**

  - **Hailuo AI**  
    - Type: HTTP Request  
    - Role: Sends image and parameters to Hailuo AI API to create animated video  
    - Config: HTTP POST with image reference and animation parameters, API key  
    - Inputs: Limit only first Image to video  
    - Outputs: Wait Hailou done process  
    - Edge Cases: API errors, invalid image data

  - **Wait Hailou done process**  
    - Type: Wait  
    - Role: Pauses workflow to allow Hailuo AI to process video animation asynchronously  
    - Config: Waits for webhook or fixed delay  
    - Inputs: Hailuo AI, If1 (false branch)  
    - Outputs: Get Hailuo Status  
    - Edge Cases: Timeout, webhook failures

  - **Get Hailuo Status**  
    - Type: HTTP Request  
    - Role: Polls Hailuo AI API to check video processing status  
    - Config: HTTP GET with job ID, API key  
    - Inputs: Wait Hailou done process  
    - Outputs: If1  
    - Edge Cases: API errors, job not found

  - **If1**  
    - Type: If  
    - Role: Branches workflow based on whether Hailuo video processing is complete  
    - Config: Condition on status field (e.g., "completed")  
    - Inputs: Get Hailuo Status  
    - Outputs:  
      - True: Get Hailuo Download Link  
      - False: Wait Hailou done process (loop)  
    - Edge Cases: Incorrect status parsing

  - **Get Hailuo Download Link**  
    - Type: HTTP Request  
    - Role: Retrieves the download URL for the completed video from Hailuo AI  
    - Config: HTTP GET with job ID, API key  
    - Inputs: If1 (true branch)  
    - Outputs: Download Video  
    - Edge Cases: Link expiration, API errors

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads the video file from the Hailuo AI download URL  
    - Config: HTTP GET request  
    - Inputs: Get Hailuo Download Link  
    - Outputs: Save Video  
    - Edge Cases: Network errors, incomplete downloads

  - **Save Video**  
    - Type: Google Cloud Storage  
    - Role: Uploads the downloaded video file to Google Cloud Storage  
    - Config: Bucket name, path, credentials  
    - Inputs: Download Video  
    - Outputs: Merge1  
    - Edge Cases: Upload failures, permission issues

---

#### 2.5 Video Assembly & Captioning

- **Overview:** Combines audio and video, monitors video creation progress, retrieves captions, and prepares final video metadata.
- **Nodes Involved:**  
  - Merge1 (Merge)  
  - Map Music and Video (Set)  
  - Create Video (HTTP Request)  
  - Wait1 andynocode combine video (Wait)  
  - Get Video Progress (HTTP Request)  
  - If (If)  
  - Set Final Video URL (Set)  
  - Get Captions (HTTP Request)  
  - JSON to Object (Code)  
  - Google Sheets (Google Sheets)

- **Node Details:**

  - **Merge1**  
    - Type: Merge  
    - Role: Combines audio and video data streams for final video creation  
    - Config: Default merge mode (e.g., merge by index)  
    - Inputs: Combine Transcript, Save Video  
    - Outputs: Map Music and Video  
    - Edge Cases: Data mismatch, missing inputs

  - **Map Music and Video**  
    - Type: Set  
    - Role: Prepares payload combining audio and video references for final video creation API  
    - Config: Sets parameters such as audio URL, video URL, metadata  
    - Inputs: Merge1  
    - Outputs: Create Video  
    - Edge Cases: Missing or invalid URLs

  - **Create Video**  
    - Type: HTTP Request  
    - Role: Calls video assembly API (e.g., andynocode or similar) to combine audio and video into final product  
    - Config: HTTP POST with combined media URLs, API key  
    - Inputs: Map Music and Video  
    - Outputs: Wait1 andynocode combine video  
    - Edge Cases: API errors, invalid media URLs

  - **Wait1 andynocode combine video**  
    - Type: Wait  
    - Role: Waits for asynchronous video assembly process to complete  
    - Config: Waits for webhook or fixed delay  
    - Inputs: Create Video, If (false branch)  
    - Outputs: Get Video Progress  
    - Edge Cases: Timeout, webhook failures

  - **Get Video Progress**  
    - Type: HTTP Request  
    - Role: Polls video assembly API for processing status  
    - Config: HTTP GET with job ID, API key  
    - Inputs: Wait1 andynocode combine video  
    - Outputs: If  
    - Edge Cases: API errors, job not found

  - **If**  
    - Type: If  
    - Role: Checks if video assembly is complete  
    - Config: Condition on status field (e.g., "completed")  
    - Inputs: Get Video Progress  
    - Outputs:  
      - True: Set Final Video URL  
      - False: Wait1 andynocode combine video (loop)  
    - Edge Cases: Incorrect status parsing

  - **Set Final Video URL**  
    - Type: Set  
    - Role: Sets the final video URL and metadata for downstream use  
    - Config: Sets fields like final video URL, status, title  
    - Inputs: If (true branch)  
    - Outputs: Get Captions  
    - Edge Cases: Missing URL or metadata

  - **Get Captions**  
    - Type: HTTP Request  
    - Role: Retrieves captions or subtitles for the final video  
    - Config: HTTP GET with video ID or URL, API key  
    - Inputs: Set Final Video URL  
    - Outputs: JSON to Object  
    - Edge Cases: Captions not available, API errors

  - **JSON to Object**  
    - Type: Code  
    - Role: Parses JSON captions into structured object for Google Sheets  
    - Config: JavaScript JSON.parse or equivalent  
    - Inputs: Get Captions  
    - Outputs: Google Sheets  
    - Edge Cases: Malformed JSON

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Uploads video metadata, URLs, and status to a Google Sheets document for tracking  
    - Config: Spreadsheet ID, sheet name, credentials  
    - Inputs: JSON to Object  
    - Outputs: None  
    - Edge Cases: Permission errors, API limits

---

#### 2.6 Data Management & Distribution

- **Overview:** Finalizes data storage and enables multi-platform video distribution.
- **Nodes Involved:**  
  - (Implicit in Google Sheets node and final outputs; no explicit nodes for distribution in this workflow)

- **Node Details:**  
  - Google Sheets node manages centralized tracking of video assets with columns for title, URL, and status.  
  - Users can manually download or upload videos to social platforms using URLs provided.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note |
|-----------------------------|-------------------------|----------------------------------------------|-------------------------------|--------------------------------|-------------|
| When clicking ‘Test workflow’| Manual Trigger          | Manual start trigger for testing             | None                          | Video Catagory                 |             |
| Video Catagory              | Set                     | Defines video category parameters             | When clicking ‘Test workflow’ | Generate Transcript            |             |
| Generate Transcript         | HTTP Request            | Generates video script/transcript via AI     | Video Catagory                | Fetch Elevenlabs              |             |
| Fetch Elevenlabs            | HTTP Request            | Converts script to voiceover audio            | Generate Transcript           | Save Audio                    |             |
| Save Audio                  | Google Cloud Storage    | Uploads audio file to cloud storage           | Fetch Elevenlabs              | Map Public Link               |             |
| Map Public Link             | Set                     | Creates public URL for audio file             | Save Audio                   | Download Audio                |             |
| Download Audio              | HTTP Request            | Downloads audio file from public URL          | Map Public Link              | Open AI Whisper              |             |
| Open AI Whisper             | HTTP Request            | Transcribes or processes audio                 | Download Audio               | Combine Transcript, Create a list of Image Text |             |
| Combine Transcript          | Code                    | Merges transcript segments                     | Open AI Whisper              | Merge1                       |             |
| Create a list of Image Text | Code                    | Extracts text snippets for image prompts      | Open AI Whisper              | Split Image                  |             |
| Split Image                | SplitOut                 | Splits image text list into individual items  | Create a list of Image Text  | Generate Image Prompt         |             |
| Generate Image Prompt       | HTTP Request            | Generates image prompts via AI                 | Split Image                  | Split Prompt                 |             |
| Split Prompt               | SplitOut                 | Splits image prompts for individual requests  | Generate Image Prompt        | Get Image Base 64            |             |
| Get Image Base 64          | HTTP Request            | Generates images and returns base64 data      | Split Prompt                | Convert Base64 to Data       |             |
| Convert Base64 to Data     | ConvertToFile            | Converts base64 image data to file format     | Get Image Base 64           | Save Image                  |             |
| Save Image                 | Google Cloud Storage     | Uploads image files to cloud storage           | Convert Base64 to Data       | Limit only first Image to video |             |
| Limit only first Image to video | Limit               | Limits processing to first image only          | Save Image                  | Hailuo AI                   |             |
| Hailuo AI                 | HTTP Request             | Sends image to Hailuo AI for video animation  | Limit only first Image to video | Wait Hailou done process   |             |
| Wait Hailou done process  | Wait                     | Waits for Hailuo AI video processing           | Hailuo AI, If1 (false branch) | Get Hailuo Status           |             |
| Get Hailuo Status         | HTTP Request             | Polls Hailuo AI for video processing status    | Wait Hailou done process     | If1                         |             |
| If1                       | If                       | Checks if Hailuo video processing is complete | Get Hailuo Status            | Get Hailuo Download Link (true), Wait Hailou done process (false) |             |
| Get Hailuo Download Link  | HTTP Request             | Retrieves download link for completed video    | If1 (true branch)            | Download Video              |             |
| Download Video            | HTTP Request             | Downloads video file from Hailuo AI            | Get Hailuo Download Link     | Save Video                  |             |
| Save Video                | Google Cloud Storage     | Uploads video file to cloud storage             | Download Video               | Merge1                      |             |
| Merge1                    | Merge                    | Combines audio and video data streams           | Combine Transcript, Save Video | Map Music and Video         |             |
| Map Music and Video       | Set                      | Prepares payload for final video creation       | Merge1                       | Create Video                |             |
| Create Video              | HTTP Request             | Calls API to assemble final video                | Map Music and Video          | Wait1 andynocode combine video |             |
| Wait1 andynocode combine video | Wait                 | Waits for final video assembly process          | Create Video, If (false branch) | Get Video Progress         |             |
| Get Video Progress        | HTTP Request             | Polls final video assembly status                | Wait1 andynocode combine video | If                        |             |
| If                        | If                       | Checks if final video assembly is complete      | Get Video Progress           | Set Final Video URL (true), Wait1 andynocode combine video (false) |             |
| Set Final Video URL       | Set                      | Sets final video URL and metadata                 | If (true branch)             | Get Captions                |             |
| Get Captions              | HTTP Request             | Retrieves captions/subtitles for final video     | Set Final Video URL          | JSON to Object              |             |
| JSON to Object            | Code                     | Parses captions JSON into structured object       | Get Captions                 | Google Sheets               |             |
| Google Sheets             | Google Sheets            | Uploads video metadata and URLs for tracking      | JSON to Object               | None                       |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - Disabled by default (optional for testing)  

2. **Create Set Node for Video Category**  
   - Type: Set  
   - Name: "Video Catagory"  
   - Configure parameters to define video category or topic  

3. **Create HTTP Request Node for Transcript Generation**  
   - Type: HTTP Request  
   - Name: "Generate Transcript"  
   - Configure POST request to AI API (OpenAI or Together AI) with prompt based on video category  
   - Set credentials for API access  

4. **Create HTTP Request Node for ElevenLabs Audio Generation**  
   - Type: HTTP Request  
   - Name: "Fetch Elevenlabs"  
   - Configure POST request to ElevenLabs API with transcript text  
   - Set ElevenLabs API credentials  

5. **Create Google Cloud Storage Node to Save Audio**  
   - Type: Google Cloud Storage  
   - Name: "Save Audio"  
   - Configure bucket, path, and credentials for audio file upload  

6. **Create Set Node to Map Public Audio URL**  
   - Type: Set  
   - Name: "Map Public Link"  
   - Configure to construct public URL from storage bucket and file path  

7. **Create HTTP Request Node to Download Audio**  
   - Type: HTTP Request  
   - Name: "Download Audio"  
   - Configure GET request to public audio URL  

8. **Create HTTP Request Node for OpenAI Whisper Transcription**  
   - Type: HTTP Request  
   - Name: "Open AI Whisper"  
   - Configure POST request to OpenAI Whisper API with audio data  
   - Set OpenAI API credentials  

9. **Create Code Node to Combine Transcript**  
   - Type: Code  
   - Name: "Combine Transcript"  
   - Implement JavaScript to merge transcript segments  

10. **Create Code Node to Create List of Image Text**  
    - Type: Code  
    - Name: "Create a list of Image Text"  
    - Implement JavaScript to extract text snippets for image prompts  

11. **Create SplitOut Node to Split Image Text List**  
    - Type: SplitOut  
    - Name: "Split Image"  

12. **Create HTTP Request Node to Generate Image Prompts**  
    - Type: HTTP Request  
    - Name: "Generate Image Prompt"  
    - Configure POST request to AI image prompt generation API  
    - Set credentials  

13. **Create SplitOut Node to Split Image Prompts**  
    - Type: SplitOut  
    - Name: "Split Prompt"  

14. **Create HTTP Request Node to Generate Images (Base64)**  
    - Type: HTTP Request  
    - Name: "Get Image Base 64"  
    - Configure POST request to image generation API returning base64 images  
    - Set credentials  

15. **Create ConvertToFile Node to Convert Base64 to File**  
    - Type: ConvertToFile  
    - Name: "Convert Base64 to Data"  
    - Configure output file type (e.g., PNG)  

16. **Create Google Cloud Storage Node to Save Images**  
    - Type: Google Cloud Storage  
    - Name: "Save Image"  
    - Configure bucket, path, and credentials  

17. **Create Limit Node to Restrict to First Image**  
    - Type: Limit  
    - Name: "Limit only first Image to video"  
    - Set limit count to 1  

18. **Create HTTP Request Node to Send Image to Hailuo AI**  
    - Type: HTTP Request  
    - Name: "Hailuo AI"  
    - Configure POST request to Hailuo AI animation API with image reference  
    - Set credentials  

19. **Create Wait Node to Wait for Hailuo Processing**  
    - Type: Wait  
    - Name: "Wait Hailou done process"  
    - Configure wait for webhook or fixed delay  

20. **Create HTTP Request Node to Get Hailuo Status**  
    - Type: HTTP Request  
    - Name: "Get Hailuo Status"  
    - Configure GET request to poll video processing status  

21. **Create If Node to Check Hailuo Completion**  
    - Type: If  
    - Name: "If1"  
    - Condition: status == "completed"  

22. **Create HTTP Request Node to Get Hailuo Download Link**  
    - Type: HTTP Request  
    - Name: "Get Hailuo Download Link"  

23. **Create HTTP Request Node to Download Video**  
    - Type: HTTP Request  
    - Name: "Download Video"  

24. **Create Google Cloud Storage Node to Save Video**  
    - Type: Google Cloud Storage  
    - Name: "Save Video"  

25. **Create Merge Node to Combine Transcript and Video**  
    - Type: Merge  
    - Name: "Merge1"  

26. **Create Set Node to Map Music and Video Data**  
    - Type: Set  
    - Name: "Map Music and Video"  
    - Configure payload for final video creation  

27. **Create HTTP Request Node to Create Final Video**  
    - Type: HTTP Request  
    - Name: "Create Video"  

28. **Create Wait Node to Wait for Final Video Assembly**  
    - Type: Wait  
    - Name: "Wait1 andynocode combine video"  

29. **Create HTTP Request Node to Get Video Progress**  
    - Type: HTTP Request  
    - Name: "Get Video Progress"  

30. **Create If Node to Check Final Video Completion**  
    - Type: If  
    - Name: "If"  
    - Condition: status == "completed"  

31. **Create Set Node to Set Final Video URL**  
    - Type: Set  
    - Name: "Set Final Video URL"  

32. **Create HTTP Request Node to Get Captions**  
    - Type: HTTP Request  
    - Name: "Get Captions"  

33. **Create Code Node to Parse Captions JSON**  
    - Type: Code  
    - Name: "JSON to Object"  

34. **Create Google Sheets Node to Upload Metadata**  
    - Type: Google Sheets  
    - Name: "Google Sheets"  
    - Configure spreadsheet ID, sheet name, and credentials  

35. **Connect Nodes According to the workflow connections described in Section 1 and 2.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow integrates multiple AI services: ElevenLabs for voiceover, Together AI for prompt engineering, Hailuo AI for video animation, and OpenAI for transcription and text generation. | Overview section |
| Google Sheets is used for centralized tracking of video metadata including title, URL, and status.  | Data management |
| The workflow supports asynchronous processing with Wait nodes and polling via If nodes to handle AI service delays. | Workflow design pattern |
| Credentials required: ElevenLabs API key, OpenAI API key, Hailuo AI API key, Google Cloud Storage credentials, Google Sheets OAuth2 credentials. | Credential setup |
| Users can extend this workflow to automate uploads to social media platforms manually using the generated video URLs. | Distribution note |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the automated YouTube short video creation workflow using ElevenLabs, Hailuo AI, and other AI services.