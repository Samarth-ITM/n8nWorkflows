Generate & Translate Video Subtitles with OpenAI Whisper and LibreTranslate

https://n8nworkflows.xyz/workflows/generate---translate-video-subtitles-with-openai-whisper-and-libretranslate-9301


# Generate & Translate Video Subtitles with OpenAI Whisper and LibreTranslate

---

### 1. Workflow Overview

This workflow automates the process of generating subtitles for videos and optionally translating those subtitles into another language. It is designed for users who want accurate transcription and translation of video content using open-source and free tools combined with n8n's automation capabilities.

The workflow logically divides into the following blocks:

- **1.1 Input Reception & Video Download:** Accepts a video URL via a webhook and downloads the video file.

- **1.2 Audio Extraction & Transcription:** Extracts audio from the downloaded video using FFmpeg, then transcribes the audio to subtitles using the OpenAI Whisper model running locally, generating an SRT subtitle file.

- **1.3 Subtitle Translation:** Optionally translates the generated subtitles using LibreTranslate, producing a translated SRT file.

- **1.4 Output & Notification:** Merges the original and translated subtitle paths and sends an email notification with the subtitle files attached.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Video Download

**Overview:**  
Receives a video URL via an HTTP webhook, acknowledges the request, and downloads the video file for further processing.

**Nodes Involved:**  
- Webhook Trigger  
- Download Video  
- Sticky Note (explaining input block)

**Node Details:**

- **Webhook Trigger**  
  - *Type:* Webhook Trigger  
  - *Role:* Entry point; listens for POST requests on path `/generate-subtitles`.  
  - *Configuration:* Returns immediate JSON response `{ "Processing started" }` to acknowledge receipt.  
  - *Key Expression:* Uses incoming JSON body to extract `video_url`.  
  - *Inputs:* External HTTP request.  
  - *Outputs:* Passes JSON containing video URL to Download Video node.  
  - *Failures:* Missing or invalid URL in payload; malformed requests.

- **Download Video**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the video file from the URL provided.  
  - *Configuration:* URL derived dynamically from `{{$json["body"]["video_url"]}}`; response format set to "file" to save binary video data.  
  - *Inputs:* JSON from Webhook Trigger.  
  - *Outputs:* Binary video file for next step.  
  - *Failures:* Invalid URL, network errors, HTTP errors, unsupported video formats.

- **Sticky Note**  
  - *Content:* Describes this block as input reception, video URL intake, and preparation for audio extraction.

---

#### 2.2 Audio Extraction & Transcription

**Overview:**  
Extracts audio from the downloaded video file using FFmpeg, then transcribes the audio into subtitles using a local installation of OpenAI Whisper. Reads the resulting subtitle file to prepare for translation.

**Nodes Involved:**  
- Extract Audio (FFmpeg)  
- Run Whisper (Local)  
- Read SRT File  
- Sticky Note (explaining audio and transcription block)

**Node Details:**

- **Extract Audio (FFmpeg)**  
  - *Type:* Execute Command  
  - *Role:* Runs FFmpeg CLI to extract audio track from the video file, saving as WAV.  
  - *Configuration:* Command: `ffmpeg -i /data/video.mp4 -q:a 0 -map a /data/audio.wav`  
  - *Inputs:* Binary video file from Download Video node (implicit file `/data/video.mp4`).  
  - *Outputs:* Audio file `/data/audio.wav` for transcription.  
  - *Failures:* FFmpeg not installed or not in PATH; file not found; corrupt input video.

- **Run Whisper (Local)**  
  - *Type:* Execute Command  
  - *Role:* Runs the Whisper CLI locally to transcribe audio into SRT subtitle format.  
  - *Configuration:* Command: `whisper /data/audio.wav --model base --output_format srt --output_dir /data/`  
  - *Inputs:* Audio file `/data/audio.wav` from previous node.  
  - *Outputs:* Subtitle file `/data/audio.srt`.  
  - *Failures:* Whisper CLI not installed; missing model files; incorrect audio format; runtime errors.

- **Read SRT File**  
  - *Type:* Read Binary File  
  - *Role:* Reads the generated subtitle file from disk for further processing.  
  - *Configuration:* File path set to `/data/audio.srt`.  
  - *Inputs:* Output of Whisper transcription.  
  - *Outputs:* Binary subtitle file content.  
  - *Failures:* File not found; file read permission errors.

- **Sticky Note1**  
  - *Content:* Describes this block’s focus on extracting audio and transcribing it to subtitles using Whisper.

---

#### 2.3 Subtitle Translation

**Overview:**  
Optionally translates the original subtitles into a target language using LibreTranslate API and writes the translated subtitles as a new SRT file.

**Nodes Involved:**  
- Translate Subtitles (LibreTranslate)  
- Write Translated SRT  
- Sticky Note (explaining translation block)

**Node Details:**

- **Translate Subtitles (LibreTranslate)**  
  - *Type:* HTTP Request  
  - *Role:* Sends subtitle text to LibreTranslate API for language translation via authenticated request.  
  - *Configuration:*  
    - URL: `https://libretranslate.com/translate`  
    - Authentication: Uses generic credential (configured separately).  
    - JSON parameters enabled to send translation request payload.  
  - *Inputs:* Subtitle file content from Read SRT File node.  
  - *Outputs:* Translated subtitle text.  
  - *Failures:* API authentication errors; network problems; rate limits; invalid input format.

- **Write Translated SRT**  
  - *Type:* Write Binary File  
  - *Role:* Writes the translated subtitle text to disk as `audio_translated.srt`.  
  - *Configuration:* Filename configured as `audio_translated.srt`.  
  - *Inputs:* Translated subtitle content.  
  - *Outputs:* Binary file for downstream use.  
  - *Failures:* File write permission errors; disk space issues.

- **Sticky Note2**  
  - *Content:* Explains translation integration, combining original and translated subtitle paths.

---

#### 2.4 Output & Notification

**Overview:**  
Combines the original and translated subtitle streams and sends an email with the generated subtitle files attached.

**Nodes Involved:**  
- Merge Paths  
- Send a message  
- Sticky Notes describing output flow

**Node Details:**

- **Merge Paths**  
  - *Type:* Merge  
  - *Role:* Passes through both original and translated subtitle outputs to a common endpoint.  
  - *Configuration:* Mode set to "passThrough" to forward all incoming data unaltered.  
  - *Inputs:* Receives two inputs — original SRT from Read SRT File and translated SRT from Write Translated SRT.  
  - *Outputs:* Sends merged data downstream.  
  - *Failures:* Data format mismatch; synchronization issues.

- **Send a message**  
  - *Type:* Gmail  
  - *Role:* Sends an email with subtitles attached to a configured recipient.  
  - *Configuration:*  
    - Recipient email: `your-email@gmail.com` (to be customized).  
    - Subject: "Video Subtitles Generated".  
    - Message body text specifying attached subtitles.  
    - Credentials: Uses OAuth2 Gmail account configured in n8n.  
  - *Inputs:* Merged subtitle files.  
  - *Outputs:* None (final node).  
  - *Failures:* Authentication errors; email sending failures; attachment size limits.

- **Sticky Note3**  
  - *Content:* Contains full workflow instructions, prerequisites (self-hosting, FFmpeg, Whisper, translation credentials), and overview.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                            | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                      |
|---------------------------|---------------------|------------------------------------------|-------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| Webhook Trigger           | Webhook Trigger     | Entry point; receives video URL          | —                       | Download Video            | ## Input - Receives a video URL via webhook. Acts as the workflow’s entry point.               |
| Download Video            | HTTP Request        | Downloads video from the provided URL    | Webhook Trigger         | Extract Audio (FFmpeg)    | ## Input - Downloads the video from the provided URL.                                          |
| Extract Audio (FFmpeg)    | Execute Command     | Extracts audio track from video           | Download Video          | Run Whisper (Local)       | ## Audio Extraction & Transcribing - Uses FFmpeg to separate audio from video.                 |
| Run Whisper (Local)       | Execute Command     | Transcribes audio to subtitles (SRT)     | Extract Audio (FFmpeg)  | Read SRT File             | ## Audio Extraction & Transcribing - Runs Whisper locally to generate subtitles.               |
| Read SRT File             | Read Binary File    | Reads generated subtitle file             | Run Whisper (Local)     | Merge Paths               | ## Audio Extraction & Transcribing - Reads the generated subtitle file (.srt).                 |
| Translate Subtitles (LibreTranslate) | HTTP Request        | Translates subtitle text using API        | Read SRT File           | Write Translated SRT      | ## Translation - Sends .srt file text to LibreTranslate for language translation.              |
| Write Translated SRT      | Write Binary File   | Writes translated subtitles to new SRT    | Translate Subtitles     | Merge Paths               | ## Translation - Outputs a new .srt file containing translated subtitles.                       |
| Merge Paths               | Merge               | Merges original and translated subtitles | Read SRT File, Write Translated SRT | Send a message            | ## Translation - Combines transcription and translation paths.                                 |
| Send a message            | Gmail               | Sends email notification with subtitles   | Merge Paths             | —                         | ## Output & Notification - Sends email with subtitle attachments.                              |
| Sticky Note               | Sticky Note         | Documentation note                        | —                       | —                         | ## Input block note                                                                             |
| Sticky Note1              | Sticky Note         | Documentation note                        | —                       | —                         | ## Audio Extraction & Transcribing block note                                                  |
| Sticky Note2              | Sticky Note         | Documentation note                        | —                       | —                         | ## Translation block note                                                                      |
| Sticky Note3              | Sticky Note         | Documentation note                        | —                       | —                         | Full workflow prerequisites and instructions                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `generate-subtitles`  
   - Response: JSON body `{ "Processing started" }` immediately after receiving request.

2. **Create Download Video Node**  
   - Type: HTTP Request  
   - URL: Expression `{{$json["body"]["video_url"]}}`  
   - Response Format: File (binary)  
   - Connect input from Webhook Trigger output.

3. **Create Extract Audio (FFmpeg) Node**  
   - Type: Execute Command  
   - Command: `ffmpeg -i /data/video.mp4 -q:a 0 -map a /data/audio.wav`  
   - Note: Ensure FFmpeg is installed and accessible on the server running n8n.  
   - Connect input from Download Video.

4. **Create Run Whisper (Local) Node**  
   - Type: Execute Command  
   - Command: `whisper /data/audio.wav --model base --output_format srt --output_dir /data/`  
   - Note: Ensure Whisper CLI (Python package) is installed and accessible on the server.  
   - Connect input from Extract Audio (FFmpeg).

5. **Create Read SRT File Node**  
   - Type: Read Binary File  
   - File Path: `/data/audio.srt`  
   - Connect input from Run Whisper (Local).

6. **Create Translate Subtitles (LibreTranslate) Node**  
   - Type: HTTP Request  
   - URL: `https://libretranslate.com/translate`  
   - Authentication: Configure generic credential with LibreTranslate API key if required.  
   - JSON Parameters: Enabled  
   - Connect input from Read SRT File.

7. **Create Write Translated SRT Node**  
   - Type: Write Binary File  
   - File Name: `audio_translated.srt`  
   - Connect input from Translate Subtitles.

8. **Create Merge Paths Node**  
   - Type: Merge  
   - Mode: passThrough  
   - Connect two inputs:  
     - First input from Read SRT File (original subtitles)  
     - Second input from Write Translated SRT (translated subtitles)

9. **Create Send a message Node**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - To: `your-email@gmail.com` (replace with actual recipient)  
   - Subject: `Video Subtitles Generated`  
   - Message: `Attached are the subtitles (original and translated, if requested).`  
   - Connect input from Merge Paths.

10. **Add Sticky Notes (Optional)**  
    - Add four sticky notes with the content from the workflow to document input, audio extraction, translation, and overall instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                      | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow requires a self-hosted n8n instance because it uses Execute Command nodes for FFmpeg and Whisper CLI execution.                                                                                                                     | Sticky Note3 content                                                                                          |
| Install FFmpeg and ensure it is accessible in the shell environment where n8n runs (`ffmpeg` command).                                                                                                                                             | Prerequisites section in Sticky Note3                                                                        |
| Install OpenAI Whisper CLI (Python package) and verify accessibility in the shell environment (`whisper` command).                                                                                                                                | Prerequisites section in Sticky Note3                                                                        |
| Setup LibreTranslate credentials as a generic credential in n8n for authenticated translation requests.                                                                                                                                            | Translate Subtitles node configuration                                                                        |
| Gmail OAuth2 credentials must be configured in n8n to send emails via Gmail API securely.                                                                                                                                                          | Send a message node configuration                                                                             |
| For more on OpenAI Whisper see: https://github.com/openai/whisper                                                                                                                                                                                 | External resource                                                                                              |
| LibreTranslate API public instance used here is https://libretranslate.com/ but self-hosted or other instances are recommended for heavy use to avoid rate limits.                                                                               | Translation block context                                                                                      |
| Workflow handles only videos accessible via direct URL; private or protected videos require additional authentication or pre-processing not covered here.                                                                                        | Input block context                                                                                             |
| Output email attachments size may be limited by Gmail; large subtitle files or multiple attachments may require alternative delivery (e.g., cloud storage links).                                                                                 | Output & Notification context                                                                                  |

---

**Disclaimer:** The provided text is extracted from an automated n8n workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---