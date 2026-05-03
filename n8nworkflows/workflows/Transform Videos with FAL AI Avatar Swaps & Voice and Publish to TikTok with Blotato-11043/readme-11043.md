Transform Videos with FAL AI Avatar Swaps & Voice and Publish to TikTok with Blotato

https://n8nworkflows.xyz/workflows/transform-videos-with-fal-ai-avatar-swaps---voice-and-publish-to-tiktok-with-blotato-11043


# Transform Videos with FAL AI Avatar Swaps & Voice and Publish to TikTok with Blotato

### 1. Workflow Overview

This workflow automates the transformation of viral videos by swapping the original character’s face with a custom AI avatar and changing the voice, then publishes the final video to TikTok via Blotato. It is designed for content creators who want to quickly generate engaging AI-enhanced videos and track their outputs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives avatar images and video URLs via Telegram messages.
- **1.2 Audio and Video Extraction:** Separates the audio from the original video for independent processing.
- **1.3 Parallel Transformation:** Simultaneously replaces the face in the video and transforms the voice audio using FAL AI APIs.
- **1.4 Merging and Finalization:** Combines the transformed video and audio into a single output file.
- **1.5 Publishing and Tracking:** Uploads the final video to Blotato for TikTok publishing and logs URLs to Google Sheets.
- **1.6 Caption Generation:** Transcribes audio and generates engaging TikTok captions using GPT-4.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for Telegram messages containing an avatar photo and a video URL in the caption. It downloads the avatar image, uploads it to a public file hosting service, and merges all input data for downstream processing.

**Nodes Involved:**  
- Telegram Trigger  
- Telegram Get File (Avatar)  
- Extract Video URL  
- Build Public Image URL  
- Workflow Configuration  
- Merge  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages with photos to start the workflow.  
  - Config: Listens for "message" updates; uses Telegram API credentials.  
  - Inputs: None (trigger node)  
  - Outputs: Sends message JSON to downstream nodes.  
  - Failures: Telegram API auth errors, webhook misconfiguration.  

- **Telegram Get File (Avatar)**  
  - Type: Telegram API node  
  - Role: Downloads the highest resolution avatar photo from the Telegram message.  
  - Config: Extracts the last photo file_id from the message JSON.  
  - Inputs: Telegram Trigger output  
  - Outputs: Binary data of the avatar image.  
  - Failures: File not found, Telegram API errors.  

- **Extract Video URL**  
  - Type: Code (JavaScript)  
  - Role: Parses the caption text from the Telegram message to extract the video URL.  
  - Config: Returns trimmed caption as `videoUrl`.  
  - Inputs: Telegram Trigger output  
  - Outputs: JSON with `videoUrl` string.  
  - Failures: Missing or malformed caption, expression errors.  

- **Build Public Image URL**  
  - Type: HTTP Request  
  - Role: Uploads the avatar image binary to tmpfiles.org to create a public URL.  
  - Config: POST multipart/form-data with avatar image binary.  
  - Inputs: Telegram Get File (Avatar) binary data  
  - Outputs: JSON with public URL of the avatar image.  
  - Failures: Upload failure, network timeout.  

- **Workflow Configuration**  
  - Type: Set  
  - Role: Holds API keys and configuration parameters (FAL API key, Replicate API key, target voice URL).  
  - Config: Static string fields for keys and URLs.  
  - Inputs: Telegram Trigger output (to synchronize flow)  
  - Outputs: JSON with configuration data.  
  - Failures: Missing or invalid keys.  

- **Merge**  
  - Type: Merge  
  - Role: Combines the outputs of Extract Video URL, Build Public Image URL, and Workflow Configuration into one JSON object for processing.  
  - Config: Choose branch mode, uses data from the last input.  
  - Inputs: Extract Video URL, Build Public Image URL, Workflow Configuration  
  - Outputs: Combined JSON with video URL, avatar URL, and config.  
  - Failures: Data mismatch or missing inputs.

---

#### 1.2 Audio and Video Extraction

**Overview:**  
Extracts the audio track from the original video URL using Replicate’s extract-audio API, waits for job completion, then downloads the extracted audio file.

**Nodes Involved:**  
- Separate Audio (Video-to-Audio)  
- Wait  
- Download Audio  

**Node Details:**

- **Separate Audio (Video-to-Audio)**  
  - Type: HTTP Request  
  - Role: Calls Replicate API to extract audio from the video URL.  
  - Config: POST JSON with video URL and audio extraction parameters (fade, trim, quality).  
  - Inputs: Merge output (contains video URL)  
  - Outputs: JSON with job status and audio extraction info.  
  - Failures: API auth errors, invalid video URL, network issues.  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 5 seconds to allow audio extraction job to process.  
  - Inputs: Separate Audio output  
  - Outputs: Passes data unchanged after delay.  
  - Failures: None typical; workflow timeout if downstream nodes fail.  

- **Download Audio**  
  - Type: HTTP Request  
  - Role: Fetches the extracted audio file URL from the completed job.  
  - Config: GET request to audio URL with Replicate API bearer token.  
  - Inputs: Wait output  
  - Outputs: Binary audio data (mp3).  
  - Failures: Job not completed, invalid URL, auth errors.

---

#### 1.3 Parallel Transformation

**Overview:**  
Transforms the video by replacing the face with the avatar image (FAL WAN API) and transforms the audio voice (FAL Chatterbox API) in parallel, each with their own wait and fetch cycles. Finally, merges the transformed URLs for final combination.

**Nodes Involved:**  
- Transform Video (FAL WAN Replace)  
- Wait Video (Job Status - WAN)  
- Get Video (Fetch WAN Output)  
- Transform Audio (FAL Chatterbox)  
- Wait Audio (Job Status - Chatterbox)  
- Get Audio (Fetch Chatterbox Output)  
- Merge urls  

**Node Details:**

- **Transform Video (FAL WAN Replace)**  
  - Type: HTTP Request  
  - Role: Sends video URL and avatar image URL to FAL WAN API to replace the character’s face.  
  - Config: POST JSON with `video_url` and `image_url` (avatar), includes FAL API key in headers.  
  - Inputs: Merge output (video URL, avatar URL, config)  
  - Outputs: JSON with job status and response URL.  
  - Failures: API key invalid, URL malformed, network errors.  

- **Wait Video (Job Status - WAN)**  
  - Type: Wait  
  - Role: Waits for the FAL WAN job to complete (polling interval handled externally).  
  - Inputs: Transform Video output  
  - Outputs: Passes job status JSON.  
  - Failures: Timeout if job takes too long.  

- **Get Video (Fetch WAN Output)**  
  - Type: HTTP Request  
  - Role: Fetches the processed video URL from FAL WAN API after job completion.  
  - Config: GET request with FAL API key.  
  - Inputs: Wait Video output  
  - Outputs: JSON with transformed video URL.  
  - Failures: Job incomplete, auth errors.  

- **Transform Audio (FAL Chatterbox)**  
  - Type: HTTP Request  
  - Role: Sends extracted audio URL and target voice audio URL to FAL Chatterbox API for voice transformation.  
  - Config: POST JSON with `source_audio_url` and `target_voice_audio_url`, includes FAL API key.  
  - Inputs: Download Audio output (audio URL), Workflow Configuration (target voice URL)  
  - Outputs: JSON with job status and response URL.  
  - Failures: API key invalid, URL errors.  

- **Wait Audio (Job Status - Chatterbox)**  
  - Type: Wait  
  - Role: Waits for the FAL Chatterbox job to complete.  
  - Inputs: Transform Audio output  
  - Outputs: Passes job status JSON.  
  - Failures: Timeout risk.  

- **Get Audio (Fetch Chatterbox Output)**  
  - Type: HTTP Request  
  - Role: Fetches the transformed audio URL after job completion.  
  - Config: GET request with FAL API key.  
  - Inputs: Wait Audio output  
  - Outputs: JSON with transformed audio URL.  
  - Failures: Job incomplete, auth errors.  

- **Merge urls**  
  - Type: Merge  
  - Role: Combines the transformed video and audio URLs into one JSON object for merging.  
  - Config: Combine by position mode.  
  - Inputs: Get Video output, Get Audio output  
  - Outputs: Combined JSON with `video.url` and `audio.url`.  
  - Failures: Missing one of the URLs.

---

#### 1.4 Merging and Finalization

**Overview:**  
Merges the transformed video and audio into a single video file using FAL FFmpeg API, waits for job completion, fetches the final video URL, and writes the original and final URLs to Google Sheets.

**Nodes Involved:**  
- Combine (FAL FFmpeg Merge)  
- Wait Final (Job Status - FFmpeg)  
- Get Final (Fetch Final Video)  
- OUTPUT (Google Sheet Write)  

**Node Details:**

- **Combine (FAL FFmpeg Merge)**  
  - Type: HTTP Request  
  - Role: Sends transformed video and audio URLs to FAL FFmpeg API to merge into one video file.  
  - Config: POST JSON with `video_url` and `audio_url`, includes FAL API key.  
  - Inputs: Merge urls output  
  - Outputs: JSON with job status and response URL.  
  - Failures: API errors, invalid URLs.  

- **Wait Final (Job Status - FFmpeg)**  
  - Type: Wait  
  - Role: Waits for the FFmpeg merge job to complete.  
  - Inputs: Combine output  
  - Outputs: Passes job status JSON.  
  - Failures: Timeout risk.  

- **Get Final (Fetch Final Video)**  
  - Type: HTTP Request  
  - Role: Fetches the final merged video URL after job completion.  
  - Config: GET request with FAL API key.  
  - Inputs: Wait Final output  
  - Outputs: JSON with final video URL.  
  - Failures: Job incomplete, auth errors.  

- **OUTPUT (Google Sheet Write)**  
  - Type: Google Sheets  
  - Role: Appends a new row with original and final video URLs for tracking.  
  - Config: Maps `url original` to original video URL, `url output` to final video URL; uses OAuth2 credentials.  
  - Inputs: Get Final output  
  - Outputs: Confirmation of sheet write.  
  - Failures: Credential errors, sheet access issues, missing columns.

---

#### 1.5 Publishing and Tracking

**Overview:**  
Uploads the final video to Blotato for TikTok publishing and optionally other social platforms. Sends a Telegram message confirming publishing.

**Nodes Involved:**  
- Generate Caption with GPT-4  
- Upload Video to BLOTATO  
- Tiktok (Blotato node)  
- Send a text message (Telegram)  
- Merge1  
- Other social media nodes (disabled): Youtube, Facebook, Linkedin, Instagram, Twitter (X)  

**Node Details:**

- **Generate Caption with GPT-4**  
  - Type: OpenAI (Langchain)  
  - Role: Generates an engaging TikTok caption and title based on audio transcription.  
  - Config: Uses GPT-4o-mini model with prompt requesting catchy hook, hashtags, concise text.  
  - Inputs: Transcribe a recording output (transcription text)  
  - Outputs: JSON with caption and title.  
  - Failures: API quota, network errors.  

- **Upload Video to BLOTATO**  
  - Type: Blotato node (community)  
  - Role: Uploads the final video URL to Blotato media library.  
  - Config: Uses URL from Google Sheets output; requires Blotato API credentials.  
  - Inputs: Generate Caption output  
  - Outputs: Media upload confirmation with URL.  
  - Failures: API auth errors, invalid URL.  

- **Tiktok (Blotato node)**  
  - Type: Blotato node  
  - Role: Publishes the uploaded video to TikTok with generated caption.  
  - Config: Uses Blotato API key, platform set to TikTok, caption from GPT-4 output, privacy public.  
  - Inputs: Upload Video to BLOTATO output  
  - Outputs: Publishing confirmation.  
  - Failures: API errors, account issues.  
  - Note: This node is disabled in the workflow (can be enabled as needed).  

- **Send a text message (Telegram)**  
  - Type: Telegram  
  - Role: Sends a confirmation message to the Telegram chat that the video was published.  
  - Config: Sends text "Published" to the chat ID from Telegram Trigger.  
  - Inputs: Merge1 output (aggregated publishing results)  
  - Outputs: Confirmation of message sent.  
  - Failures: Telegram API errors.  

- **Merge1**  
  - Type: Merge  
  - Role: Combines outputs from multiple social media publishing nodes and the Telegram message node.  
  - Config: Choose branch mode with 6 inputs.  
  - Inputs: Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, Send a text message  
  - Outputs: Aggregated publishing results.  
  - Failures: Missing inputs if nodes disabled.

- **Other social media nodes (Youtube, Facebook, Linkedin, Instagram, Twitter (X))**  
  - Type: Blotato nodes  
  - Role: Publish to respective platforms with same caption and video URL.  
  - Config: Disabled by default; can be enabled for multi-platform publishing.  
  - Inputs: Upload Video to BLOTATO output  
  - Outputs: Publishing confirmations.  
  - Failures: API errors, account issues.

---

#### 1.6 Caption Generation

**Overview:**  
Transcribes the transformed audio to text and generates a TikTok caption using GPT-4.

**Nodes Involved:**  
- Convert mp3 to data  
- Transcribe a recording  
- Generate Caption with GPT-4  

**Node Details:**

- **Convert mp3 to data**  
  - Type: HTTP Request  
  - Role: Downloads the transformed audio mp3 file as binary data for transcription.  
  - Config: GET request to audio URL from Get Audio node.  
  - Inputs: Send Video to Telegram output (video URL)  
  - Outputs: Binary audio data.  
  - Failures: URL invalid, network errors.  

- **Transcribe a recording**  
  - Type: OpenAI (Langchain)  
  - Role: Transcribes audio binary data into text.  
  - Config: Uses OpenAI audio transcription operation with binary property set.  
  - Inputs: Convert mp3 to data output (binary audio)  
  - Outputs: JSON with transcription text.  
  - Failures: API quota, audio format issues.  

- **Generate Caption with GPT-4**  
  - See above in 1.5.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                             | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                      |
|-------------------------------|----------------------------------|--------------------------------------------|--------------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger               | Telegram Trigger                 | Receives Telegram messages with avatar    | None                                 | Telegram Get File (Avatar), Extract Video URL, Workflow Configuration | INPUT: TELEGRAM block description                                                                |
| Telegram Get File (Avatar)     | Telegram API                    | Downloads avatar image from Telegram       | Telegram Trigger                     | Build Public Image URL                  | INPUT: TELEGRAM block description                                                                |
| Extract Video URL             | Code (JavaScript)                | Extracts video URL from Telegram caption   | Telegram Trigger                     | Merge                                  | INPUT: TELEGRAM block description                                                                |
| Build Public Image URL        | HTTP Request                    | Uploads avatar image to tmpfiles.org       | Telegram Get File (Avatar)           | Merge                                  | INPUT: TELEGRAM block description                                                                |
| Workflow Configuration        | Set                            | Holds API keys and config parameters       | Telegram Trigger                     | Merge                                  | INPUT: TELEGRAM block description                                                                |
| Merge                        | Merge                          | Combines video URL, avatar URL, and config| Extract Video URL, Build Public Image URL, Workflow Configuration | Separate Audio (Video-to-Audio), Transform Video, Transform Audio | INPUT: TELEGRAM block description                                                                |
| Separate Audio (Video-to-Audio)| HTTP Request                   | Extracts audio track from video             | Merge                               | Wait                                   | STEP 1: SEPARATE AUDIO & VIDEO block description                                                 |
| Wait                         | Wait                           | Waits 5 seconds for audio extraction job   | Separate Audio (Video-to-Audio)     | Download Audio                         | STEP 1: SEPARATE AUDIO & VIDEO block description                                                 |
| Download Audio               | HTTP Request                    | Downloads extracted audio file              | Wait                               | Transform Video, Transform Audio       | STEP 1: SEPARATE AUDIO & VIDEO block description                                                 |
| Transform Video (FAL WAN Replace)| HTTP Request                | Replaces face in video with avatar          | Merge, Download Audio               | Wait Video (Job Status - WAN)           | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Wait Video (Job Status - WAN) | Wait                           | Waits for video transformation job         | Transform Video                    | Get Video (Fetch WAN Output)            | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Get Video (Fetch WAN Output)  | HTTP Request                   | Fetches transformed video URL               | Wait Video                       | Merge urls                             | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Transform Audio (FAL Chatterbox)| HTTP Request                 | Transforms voice audio                       | Download Audio, Workflow Configuration | Wait Audio (Job Status - Chatterbox) | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Wait Audio (Job Status - Chatterbox)| Wait                    | Waits for audio transformation job          | Transform Audio                  | Get Audio (Fetch Chatterbox Output)     | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Get Audio (Fetch Chatterbox Output)| HTTP Request              | Fetches transformed audio URL                | Wait Audio                     | Merge urls                             | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Merge urls                   | Merge                          | Combines transformed video and audio URLs  | Get Video, Get Audio              | Combine (FAL FFmpeg Merge)              | STEP 2: TRANSFORM (PARALLEL) block description                                                   |
| Combine (FAL FFmpeg Merge)    | HTTP Request                   | Merges transformed video and audio          | Merge urls                      | Wait Final (Job Status - FFmpeg)        | STEP 3: COMBINE & OUTPUT block description                                                       |
| Wait Final (Job Status - FFmpeg)| Wait                        | Waits for final merge job completion        | Combine                        | Get Final (Fetch Final Video)            | STEP 3: COMBINE & OUTPUT block description                                                       |
| Get Final (Fetch Final Video) | HTTP Request                   | Fetches final merged video URL               | Wait Final                    | OUTPUT (Google Sheet Write)              | STEP 3: COMBINE & OUTPUT block description                                                       |
| OUTPUT (Google Sheet Write)   | Google Sheets                  | Logs original and final video URLs           | Get Final                     | Send Video to Telegram                   | STEP 3: COMBINE & OUTPUT block description                                                       |
| Send Video to Telegram        | Telegram                       | Sends final video back to Telegram chat      | OUTPUT (Google Sheet Write)    | Convert mp3 to data                      | STEP 4: PUBLISHING & TRACKING block description                                                  |
| Convert mp3 to data           | HTTP Request                   | Downloads transformed audio as binary        | Send Video to Telegram         | Transcribe a recording                   | STEP 4: PUBLISHING & TRACKING block description                                                  |
| Transcribe a recording        | OpenAI (Langchain)             | Transcribes audio to text                     | Convert mp3 to data            | Generate Caption with GPT-4              | STEP 4: PUBLISHING & TRACKING block description                                                  |
| Generate Caption with GPT-4   | OpenAI (Langchain)             | Generates TikTok caption and title           | Transcribe a recording        | Upload Video to BLOTATO                  | STEP 4: PUBLISHING & TRACKING block description                                                  |
| Upload Video to BLOTATO       | Blotato                       | Uploads final video to Blotato media          | Generate Caption with GPT-4   | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube | STEP 4: PUBLISHING & TRACKING block description                                                  |
| Tiktok                       | Blotato                       | Publishes video to TikTok                      | Upload Video to BLOTATO       | Merge1                                  | STEP 4: PUBLISHING & TRACKING block description (disabled by default)                            |
| Linkedin                     | Blotato                       | Publishes video to LinkedIn                     | Upload Video to BLOTATO       | Merge1                                  | STEP 4: PUBLISHING & TRACKING block description (disabled by default)                            |
| Facebook                     | Blotato                       | Publishes video to Facebook                     | Upload Video to BLOTATO       | Merge1                                  | STEP 4: PUBLISHING & TRACKING block description (disabled by default)                            |
| Instagram                    | Blotato                       | Publishes video to Instagram                    | Upload Video to BLOTATO       | Merge1                                  | STEP 4: PUBLISHING & TRACKING block description (disabled by default)                            |
| Twitter (X)                  | Blotato                       | Publishes video to Twitter (X)                  | Upload Video to BLOTATO       | Merge1                                  | STEP 4: PUBLISHING & TRACKING block description (disabled by default)                            |
| Youtube                      | Blotato                       | Publishes video to YouTube                       | Upload Video to BLOTATO       | Merge1                                  | STEP 4: PUBLISHING & TRACKING block description (disabled by default)                            |
| Merge1                       | Merge                         | Aggregates publishing results                   | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, Send a text message | Send a text message                      | STEP 4: PUBLISHING & TRACKING block description                                                  |
| Send a text message          | Telegram                      | Sends publishing confirmation to Telegram chat | Merge1                       | None                                    | STEP 4: PUBLISHING & TRACKING block description                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot credentials  
   - Set to listen for "message" updates  

2. **Create Telegram Get File (Avatar) node**  
   - Type: Telegram  
   - Set resource to "file"  
   - File ID expression: `{{$json.message.photo[$json.message.photo.length - 1].file_id}}`  
   - Connect input from Telegram Trigger  

3. **Create Extract Video URL node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const caption = $input.first().json.message.caption || '';
     return [{ json: { videoUrl: caption.trim() } }];
     ```  
   - Connect input from Telegram Trigger  

4. **Create Build Public Image URL node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://tmpfiles.org/api/v1/upload`  
   - Content-Type: multipart/form-data  
   - Body parameter: file (binary data from Telegram Get File)  
   - Connect input from Telegram Get File (Avatar)  

5. **Create Workflow Configuration node**  
   - Type: Set  
   - Add string fields:  
     - falApiKey (your FAL API key)  
     - replicateApiKey (your Replicate API key)  
     - targetVoiceAudioUrl (URL of target voice audio)  
   - Connect input from Telegram Trigger  

6. **Create Merge node**  
   - Type: Merge  
   - Mode: Choose branch  
   - Number of inputs: 3  
   - Use data of input 2  
   - Connect inputs from Extract Video URL, Build Public Image URL, Workflow Configuration  

7. **Create Separate Audio (Video-to-Audio) node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/models/lucataco/extract-audio/predictions`  
   - Headers: Authorization: `bearer {{ $json.replicateApiKey }}`, Content-Type: application/json  
   - JSON Body:  
     ```json
     {
       "input": {
         "video": "{{ $json.videoUrl }}",
         "fade_in": 0,
         "fade_out": 0,
         "trim_end": 0,
         "trim_start": 0,
         "volume_boost": 1,
         "audio_quality": "high",
         "output_format": "mp3",
         "noise_reduction": false,
         "normalize_audio": false
       }
     }
     ```  
   - Connect input from Merge  

8. **Create Wait node**  
   - Type: Wait  
   - Duration: 5 seconds  
   - Connect input from Separate Audio  

9. **Create Download Audio node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{$json.urls.get}}` (from Separate Audio output)  
   - Headers: Authorization: `bearer {{ $json.replicateApiKey }}`  
   - Response format: File (binary)  
   - Connect input from Wait  

10. **Create Transform Video (FAL WAN Replace) node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/wan/v2.2-14b/animate/replace`  
    - Headers: Content-Type: application/json, Authorization: `Key {{ $json.falApiKey }}`  
    - JSON Body:  
      ```json
      {
        "video_url": "{{ $json.videoUrl }}",
        "image_url": "{{ $json.data.url.replace(/^http:\/\/tmpfiles\.org\/(\d+)\/(.*)$/i, 'https://tmpfiles.org/dl/$1/$2') }}"
      }
      ```  
    - Connect input from Merge and Download Audio (binary data)  

11. **Create Wait Video (Job Status - WAN) node**  
    - Type: Wait  
    - Connect input from Transform Video  

12. **Create Get Video (Fetch WAN Output) node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$json.response_url}}` (from Wait Video output)  
    - Headers: Authorization: `Key {{ $json.replicateApiKey }}`  
    - Connect input from Wait Video  

13. **Create Transform Audio (FAL Chatterbox) node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/chatterbox/speech-to-speech`  
    - Headers: Content-Type: application/json, Authorization: `Key {{ $json.falApiKey }}`  
    - JSON Body:  
      ```json
      {
        "source_audio_url": "{{ $json.output }}",
        "target_voice_audio_url": "{{ $json.targetVoiceAudioUrl }}"
      }
      ```  
    - Connect input from Download Audio and Workflow Configuration  

14. **Create Wait Audio (Job Status - Chatterbox) node**  
    - Type: Wait  
    - Connect input from Transform Audio  

15. **Create Get Audio (Fetch Chatterbox Output) node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$json.response_url}}` (from Wait Audio output)  
    - Headers: Authorization: `Key {{ $json.falApiKey }}`  
    - Connect input from Wait Audio  

16. **Create Merge urls node**  
    - Type: Merge  
    - Mode: Combine by position  
    - Inputs: 2  
    - Connect inputs from Get Video and Get Audio  

17. **Create Combine (FAL FFmpeg Merge) node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/merge-audio-video`  
    - Headers: Content-Type: application/json, Authorization: `Key {{ $json.falApiKey }}`  
    - JSON Body:  
      ```json
      {
        "video_url": "{{ $json.video.url }}",
        "audio_url": "{{ $json.audio.url }}"
      }
      ```  
    - Connect input from Merge urls  

18. **Create Wait Final (Job Status - FFmpeg) node**  
    - Type: Wait  
    - Connect input from Combine  

19. **Create Get Final (Fetch Final Video) node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$json.response_url}}` (from Wait Final output)  
    - Headers: Authorization: `Key {{ $json.falApiKey }}`  
    - Connect input from Wait Final  

20. **Create OUTPUT (Google Sheet Write) node**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document and Sheet: Select your target Google Sheet and sheet with columns `url original` and `url output`  
    - Map columns:  
      - `url original` → `{{$json.videoUrl}}`  
      - `url output` → `{{$json.video.url}}` (final video URL)  
    - Use Google Sheets OAuth2 credentials  
    - Connect input from Get Final  

21. **Create Send Video to Telegram node**  
    - Type: Telegram  
    - Operation: sendVideo  
    - Chat ID: `{{$json.message.chat.id}}` (from Telegram Trigger)  
    - File: `{{$json.videoUrl}}` (original video URL) or final video URL as needed  
    - Caption: "Your video is ready! 🎥 {{$json.videoUrl}}"  
    - Connect input from OUTPUT (Google Sheet Write)  

22. **Create Convert mp3 to data node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$json.audio.url}}` (from Get Audio)  
    - Response format: File (binary)  
    - Connect input from Send Video to Telegram  

23. **Create Transcribe a recording node**  
    - Type: OpenAI (Langchain)  
    - Operation: Transcribe audio  
    - Binary property: `data` (from Convert mp3 to data)  
    - Connect input from Convert mp3 to data  

24. **Create Generate Caption with GPT-4 node**  
    - Type: OpenAI (Langchain)  
    - Model: GPT-4o-mini  
    - Prompt: Create engaging TikTok caption with hashtags and title based on transcription text  
    - Connect input from Transcribe a recording  

25. **Create Upload Video to BLOTATO node**  
    - Type: Blotato  
    - Resource: media  
    - Media URL: `{{$json['url output']}}` (from Google Sheets output)  
    - Use Blotato API credentials  
    - Connect input from Generate Caption with GPT-4  

26. **Create Tiktok (Blotato) node**  
    - Type: Blotato  
    - Platform: TikTok  
    - Account: Select your TikTok account  
    - Caption: `{{$json.message.content.caption}}` (from GPT-4 output)  
    - Media URLs: `{{$json.url}}` (uploaded video URL)  
    - Use Blotato API credentials  
    - Connect input from Upload Video to BLOTATO  
    - (Optional) Enable this node to publish  

27. **Create Send a text message node**  
    - Type: Telegram  
    - Text: "Published"  
    - Chat ID: `{{$json.message.chat.id}}` (from Telegram Trigger)  
    - Connect input from Merge1 (aggregated publishing results)  

28. **Create Merge1 node**  
    - Type: Merge  
    - Mode: Choose branch  
    - Number of inputs: 6 (for social media nodes and Send a text message)  
    - Connect inputs from Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, and Send a text message nodes  

---

**Note:**  
- Ensure all API keys and credentials are correctly configured in the Workflow Configuration and credential nodes.  
- The workflow uses multiple wait nodes to poll asynchronous jobs; adjust wait times if needed.  
- The TikTok publishing node is disabled by default; enable and configure as needed.  
- Google Sheets must have columns `url original` and `url output` for proper logging.  
- The Blotato community nodes require installation and API key setup as described in the sticky notes.  

---

This completes the detailed reference documentation for the "Transform Videos with FAL AI Avatar Swaps & Voice and Publish to TikTok with Blotato" workflow.