🎙️ Convert Voice Notes to X Posts with Google Drive and AssemblyAI

https://n8nworkflows.xyz/workflows/----convert-voice-notes-to-x-posts-with-google-drive-and-assemblyai-8184


# 🎙️ Convert Voice Notes to X Posts with Google Drive and AssemblyAI

### 1. Workflow Overview

This workflow automates the conversion of voice notes stored in a Google Drive folder into posts on X (formerly Twitter). It is designed for content creators or social media managers who want to streamline sharing audio insights as text posts without manual transcription. The workflow includes three main logical blocks:

- **1.1 Input Reception:** Monitoring a Google Drive folder for new audio files in supported formats (.mp3, .m4a, .wav).
- **1.2 AI Processing:** Sending the audio files to AssemblyAI for transcription and keyword extraction.
- **1.3 Social Posting:** Publishing the transcribed text as tweets on X, with options for customizing content length and format.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block waits for new audio files to be uploaded into a designated Google Drive folder. It filters and triggers the workflow upon detection of supported audio formats.

**Nodes Involved:**  
- Watch Google Drive Folder

**Node Details:**

- **Watch Google Drive Folder**  
  - Type: Google Drive Trigger  
  - Role: Detects new files uploaded to a specific Drive folder, initiating the workflow.  
  - Configuration:  
    - Monitors a configured Google Drive folder (folder ID must be set in parameters).  
    - Supports audio files with extensions: .mp3, .m4a, .wav.  
    - Does not require manual polling interval setup as it uses Drive’s webhook feature (version 2).  
  - Key Expressions/Variables: Uses the file metadata such as name and link for downstream processing.  
  - Input: None (trigger node)  
  - Output: New file metadata (including public URL or file ID).  
  - Version Requirements: Requires Google Drive credentials with read access.  
  - Potential Failures:  
    - Authentication failure if OAuth2 token expires.  
    - File permission issues if the Drive folder is private or sharing settings prevent access.  
    - Unsupported file types or corrupted files might cause downstream failures.  
  - Sub-Workflow: None

#### 2.2 AI Processing

**Overview:**  
This block takes the audio file detected by the previous node and sends it to AssemblyAI’s API for transcription. The response includes the text transcript and extracted key phrases.

**Nodes Involved:**  
- Transcribe with AssemblyAI

**Node Details:**

- **Transcribe with AssemblyAI**  
  - Type: HTTP Request  
  - Role: Interfaces with AssemblyAI’s speech-to-text API to convert audio to text.  
  - Configuration:  
    - Sends a publicly accessible Google Drive file link as input to AssemblyAI.  
    - Requires AssemblyAI API key set in credentials or HTTP headers.  
    - Waits for transcription completion (note: workflow currently lacks delay or webhook to confirm completion).  
  - Key Expressions/Variables:  
    - Uses the Google Drive file’s public URL or a direct download link as the audio source.  
    - Extracts transcript and key phrases from the JSON response.  
  - Input: File metadata from the Google Drive node.  
  - Output: JSON containing transcription text, key phrases, and metadata.  
  - Version Requirements: None specific; must comply with AssemblyAI API version and authentication.  
  - Potential Failures:  
    - Invalid or inaccessible audio URL causing API errors.  
    - API key missing or invalid leading to authentication errors.  
    - Network timeouts or slow response from AssemblyAI.  
    - Partial or failed transcriptions if audio quality is poor.  
  - Sub-Workflow: None

#### 2.3 Social Posting

**Overview:**  
This block publishes the transcribed text as a post on X. It allows customization such as adding hashtags, trimming text to the 280-character limit, and threading longer content.

**Nodes Involved:**  
- Post to X (Twitter)

**Node Details:**

- **Post to X (Twitter)**  
  - Type: X (Twitter) Node  
  - Role: Posts the processed transcription text to the user’s X account as a tweet.  
  - Configuration:  
    - Uses OAuth2 credentials for X account authentication.  
    - Supports text customization via prior Function nodes (not included here) to clean text, remove filler words, or add emojis.  
    - Handles text trimming to 280 characters and can split long text into threads.  
  - Key Expressions/Variables:  
    - Accepts the transcription text, possibly enhanced or cleaned, as input for the tweet content.  
  - Input: Transcription output JSON from AssemblyAI node (or processed text).  
  - Output: Posting confirmation with tweet ID and metadata.  
  - Version Requirements: Requires valid OAuth2 credentials for X API v2.  
  - Potential Failures:  
    - Authentication failure due to expired or invalid tokens.  
    - Exceeding rate limits or posting restrictions on X.  
    - Text encoding issues or disallowed content causing rejection.  
  - Sub-Workflow: None

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                  | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                             |
|--------------------------|-----------------------|---------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Watch Google Drive Folder | Google Drive Trigger  | Detects new audio files          | None                     | Transcribe with AssemblyAI | 📁 Watch for New Voice Notes<br>Monitors a specific Google Drive folder for uploaded audio files.<br>🔹 Supported formats: .mp3, .m4a, .wav<br>💡 Tip: Name files clearly (e.g., 'Tip-about-boundaries.m4a')<br>Triggers when a new audio file is added. |
| Transcribe with AssemblyAI | HTTP Request           | Converts audio to text via AI    | Watch Google Drive Folder | Post to X (Twitter)      | 🎙️ Transcribe Audio to Text<br>Sends the voice note to AssemblyAI for speech-to-text conversion.<br>🔐 Requires AssemblyAI API key and publicly accessible file link.<br>💡 Response includes transcript + key phrases.<br>Wait for completion (add delay or webhook if needed). |
| Post to X (Twitter)       | X (Twitter) Node       | Posts transcribed text as tweet | Transcribe with AssemblyAI | None                    | 🐦 Post to X (Twitter)<br>Publishes the transcribed text as a tweet.<br>🔧 Customize:<br>- Add hashtags<br>- Trim to 280 chars<br>- Thread long content<br>💡 Use a Function node before to clean up text (remove filler words, add emojis). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Name: `Watch Google Drive Folder`  
   - Type: Google Drive Trigger  
   - Configure OAuth2 credentials with read access to the target Drive account.  
   - Set folder ID to the specific folder to monitor for new audio files.  
   - Enable file type filter for `.mp3`, `.m4a`, and `.wav` (if available).  
   - Position the node as the workflow entry point.

2. **Create HTTP Request Node for AssemblyAI Transcription**  
   - Name: `Transcribe with AssemblyAI`  
   - Type: HTTP Request  
   - Set HTTP method to POST.  
   - URL: AssemblyAI transcription endpoint (e.g., https://api.assemblyai.com/v2/transcript).  
   - Headers: Include `authorization` with AssemblyAI API key.  
   - Body Parameters: Pass the publicly accessible audio URL obtained from the Google Drive trigger node.  
   - Configure to parse JSON response.  
   - Position this node downstream of the Google Drive trigger node and connect the output data accordingly.  
   - Note: Add an explicit wait or polling mechanism to ensure transcription completes before proceeding (optional enhancement).

3. **Create X (Twitter) Node for Posting**  
   - Name: `Post to X (Twitter)`  
   - Type: X (Twitter) Node  
   - Configure OAuth2 credentials for the Twitter account.  
   - Set action to "Create Tweet" or equivalent.  
   - Map the input text field to the transcription text from the AssemblyAI node’s output.  
   - Optionally, add pre-processing Function nodes before this step to clean text, add hashtags, emojis, and ensure character limit compliance.  
   - Connect this node downstream from the AssemblyAI transcription node.

4. **Connect Nodes**  
   - Connect `Watch Google Drive Folder` output to `Transcribe with AssemblyAI` input.  
   - Connect `Transcribe with AssemblyAI` output to `Post to X (Twitter)` input.

5. **Credential Setup**  
   - Google Drive: OAuth2 with Drive API access.  
   - AssemblyAI: API key stored securely in HTTP Request credentials or headers.  
   - X (Twitter): OAuth2 with required tweet-posting scopes.

6. **Default Values and Constraints**  
   - Ensure audio files are named clearly and uploaded in supported formats.  
   - Tweets must be trimmed to 280 characters or split into threads (handled externally or via additional nodes).  
   - Monitor execution logs for authentication errors or API rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                            |
|--------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Tip: Name audio files clearly in Google Drive for easy identification (e.g., 'Tip-about-boundaries.m4a').                            | Included in Drive trigger node sticky note                  |
| AssemblyAI transcription API requires publicly accessible file URLs; consider sharing folder or generating public links.             | AssemblyAI node sticky note                                 |
| Use a Function node before posting to clean transcription text: remove filler words, add relevant emojis, and format hashtags.       | Post to X node sticky note                                  |
| AssemblyAI transcription may require additional handling for asynchronous completion (e.g., delays, polling, or webhook triggers).  | AssemblyAI node sticky note                                 |
| Twitter (X) API v2 requires OAuth2 authentication setup with proper scopes for posting tweets.                                        | Post to X node sticky note                                  |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow built with n8n, a no-code automation tool. All processing respects applicable content policies and contains no illegal or protected material. Data handled is legal and public.