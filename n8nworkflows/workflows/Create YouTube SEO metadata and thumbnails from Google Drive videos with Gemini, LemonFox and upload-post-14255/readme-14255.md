Create YouTube SEO metadata and thumbnails from Google Drive videos with Gemini, LemonFox and upload-post

https://n8nworkflows.xyz/workflows/create-youtube-seo-metadata-and-thumbnails-from-google-drive-videos-with-gemini--lemonfox-and-upload-post-14255


# Create YouTube SEO metadata and thumbnails from Google Drive videos with Gemini, LemonFox and upload-post

# 1. Workflow Overview

This workflow automates a full post-production pipeline for videos added to a Google Drive folder. When a new video appears, it makes the file temporarily public, sends it to LemonFox for transcription, cleans and stores the transcript in Google Drive, optionally retrieves a related sponsor text file, uses Gemini to generate YouTube SEO metadata plus a thumbnail prompt, generates a thumbnail image, uploads the final package through the upload-post API, and finally removes the temporary public file permission.

Typical use cases:
- Automating YouTube publishing prep from raw uploaded videos
- Generating titles, descriptions, tags, hashtags, and thumbnails from transcript content
- Enriching videos with sponsor metadata stored as companion files in Google Drive
- Reducing manual content operations for creators or media teams

## 1.1 Trigger and File Qualification
The workflow starts from a Google Drive folder watch. It only continues if the created file has a MIME type containing `video`.

## 1.2 Temporary File Access and Transcription
The video is made publicly readable through a temporary Google Drive permission, then sent to LemonFox transcription using a direct Google Drive download URL.

## 1.3 Transcript Cleanup and Storage
The raw SRT-like transcription response is normalized in a Code node and saved back into Google Drive as a text file.

## 1.4 Sponsor File Discovery and Extraction
The workflow searches Drive for a companion file named after the original video with the `_sponsors.txt` suffix, downloads it, and extracts text content from it.

## 1.5 AI Metadata Generation
A Gemini-powered AI agent receives the transcript plus sponsor text and returns structured JSON containing title, description, tags, hashtags, thumbnail prompt, and extracted sponsor entries.

## 1.6 Thumbnail Generation and Final Upload
Gemini image generation creates a thumbnail image from the prompt. The workflow then uploads the video, metadata, and thumbnail to the external upload-post API and removes the temporary Drive sharing permission.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and File Qualification

### Overview
This block detects new files in a specific Google Drive folder and filters execution so only video files continue. It prevents unnecessary API usage for non-video uploads.

### Nodes Involved
- When File Added in Drive
- If Video Mime Type

### Node Details

#### When File Added in Drive
- **Type / role:** `n8n-nodes-base.googleDriveTrigger` — polling trigger for newly created Drive files
- **Configuration choices:**
  - Event: `fileCreated`
  - Trigger scope: specific folder
  - Polling frequency: every minute
  - Folder selection is required but not populated in the JSON export
- **Key expressions / variables used:** none internally, but downstream nodes use:
  - `$('When File Added in Drive').item.json.id`
  - `$('When File Added in Drive').item.json.name`
  - `$('When File Added in Drive').item.json.mimeType`
- **Input / output connections:**
  - Entry point node
  - Outputs to `If Video Mime Type`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Google Drive OAuth credential issues
  - Incorrect or missing watched folder configuration
  - Polling delays or duplicate handling concerns if files are edited/recreated externally
  - Shared Drive behavior may differ from My Drive depending on credential scope
- **Sub-workflow reference:** none

#### If Video Mime Type
- **Type / role:** `n8n-nodes-base.if` — conditional filter
- **Configuration choices:**
  - Checks whether `mimeType` contains the string `video`
  - Strict type validation enabled
- **Key expressions / variables used:**
  - Left value: `{{ $json.mimeType }}`
  - Right value: `video`
- **Input / output connections:**
  - Input from `When File Added in Drive`
  - True branch goes to `Grant Temp File Access`
  - No false branch connected
- **Version-specific requirements:** typeVersion `2.3`, conditions format version `3`
- **Edge cases / failures:**
  - Some uncommon video MIME types may still pass correctly because of `contains`
  - Missing `mimeType` would cause false evaluation
  - Non-video files are silently ignored because the false path is unused
- **Sub-workflow reference:** none

---

## 2.2 Temporary File Access and Transcription

### Overview
This block makes the Drive file publicly readable, then submits it to LemonFox for transcription by referencing a public download URL. This design avoids downloading the binary into n8n before transcription.

### Nodes Involved
- Grant Temp File Access
- Post Audio to Lemonfox

### Node Details

#### Grant Temp File Access
- **Type / role:** `n8n-nodes-base.httpRequest` — direct call to Google Drive permissions API
- **Configuration choices:**
  - Method: `POST`
  - URL: Google Drive permissions endpoint for the triggered file
  - Authentication: predefined Google Drive OAuth2 credential
  - Body sets:
    - `role = reader`
    - `type = anyone`
- **Key expressions / variables used:**
  - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}/permissions`
- **Input / output connections:**
  - Input from `If Video Mime Type`
  - Output to `Post Audio to Lemonfox`
- **Version-specific requirements:** typeVersion `4.4`
- **Edge cases / failures:**
  - Insufficient Drive permission scopes
  - Organization-level sharing restrictions may block public access
  - Existing sharing policies may reject `anyone` permissions
  - API quota or rate-limit errors
- **Sub-workflow reference:** none

#### Post Audio to Lemonfox
- **Type / role:** `n8n-nodes-base.httpRequest` — submits transcription request to LemonFox
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.lemonfox.ai/v1/audio/transcriptions`
  - Auth: generic credential type using bearer authentication
  - Body format: form-urlencoded
  - Parameters:
    - `file`: public Google Drive download URL
    - `language`: `english`
    - `response_format`: `srt`
- **Key expressions / variables used:**
  - File URL: `https://drive.google.com/uc?id={{ $('When File Added in Drive').item.json.id }}&export=download`
- **Input / output connections:**
  - Input from `Grant Temp File Access`
  - Output to `Clean SRT Content`
- **Version-specific requirements:** typeVersion `4.4`
- **Edge cases / failures:**
  - LemonFox authentication failure
  - File URL inaccessible if permission propagation is delayed after sharing
  - Large files may exceed service limits or time out
  - Language fixed to English, causing lower quality for non-English videos
  - Response structure assumptions: downstream Code node expects `json.data`
- **Sub-workflow reference:** none

---

## 2.3 Transcript Cleanup and Storage

### Overview
This block cleans the transcription text returned by LemonFox and stores it in Google Drive as an `.srt` text file. It creates a reusable transcript artifact for later processing or auditing.

### Nodes Involved
- Clean SRT Content
- Create Text File in Drive

### Node Details

#### Clean SRT Content
- **Type / role:** `n8n-nodes-base.code` — JavaScript normalization of transcription payload
- **Configuration choices:**
  - Reads `data` from input JSON
  - Removes wrapping quotes if present
  - Unescapes `\"`
  - Converts escaped newlines `\\n` to actual newlines
  - Compresses triple-or-more blank lines into double blank lines
  - Returns cleaned text in `json.srt`
- **Key expressions / variables used:**
  - Input: `$json.data`
  - Output: `json.srt`
- **Input / output connections:**
  - Input from `Post Audio to Lemonfox`
  - Output to `Create Text File in Drive`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**
  - If `data` is missing or not a string, the code will fail
  - If LemonFox already returns plain text rather than an escaped string, cleanup may still work but assumptions are fragile
  - No defensive checks for null/undefined values
- **Sub-workflow reference:** none

#### Create Text File in Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — creates transcript file from text content
- **Configuration choices:**
  - Operation: `createFromText`
  - File name: original video base name with `.srt`
  - Content: cleaned transcript from previous node
  - Drive ID and folder ID are required but blank in the exported JSON
- **Key expressions / variables used:**
  - Name: `{{ $('When File Added in Drive').item.json.name.replace(/\.[^/.]+$/, '') }}.srt`
  - Content: `{{ $json.srt }}`
- **Input / output connections:**
  - Input from `Clean SRT Content`
  - Output to `Search Drive Files`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases / failures:**
  - Missing target Drive or folder configuration
  - Duplicate names may create multiple transcript files depending on Drive behavior
  - Insufficient write permission in destination folder
- **Sub-workflow reference:** none

---

## 2.4 Sponsor File Discovery and Extraction

### Overview
This block looks for a sponsor companion file in Google Drive, downloads it as binary content, and extracts plain text. That sponsor text is then supplied to the AI agent as a separate input.

### Nodes Involved
- Search Drive Files
- Fetch File Details
- Extract Content from File

### Node Details

#### Search Drive Files
- **Type / role:** `n8n-nodes-base.googleDrive` — searches Drive for a sponsor metadata file
- **Configuration choices:**
  - Resource: file/folder
  - Search scope:
    - Drive: My Drive
    - Folder: root
    - Search type: all
  - Result limit: 1
  - Query string searches for `<original video name without extension>_sponsors.txt`
- **Key expressions / variables used:**
  - Query: `{{ $('When File Added in Drive').item.json.name.replace(/\.[^/.]+$/, '') }}_sponsors.txt`
- **Input / output connections:**
  - Input from `Create Text File in Drive`
  - Output to `Fetch File Details`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases / failures:**
  - Search only targets root-level context and may miss files in other folders depending on Drive behavior
  - No explicit branch for “file not found”
  - If multiple matching files exist, only one is used
- **Sub-workflow reference:** none

#### Fetch File Details
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads sponsor file contents from Google Drive API
- **Configuration choices:**
  - Method implied as GET
  - URL: file endpoint using selected Drive file ID
  - Query parameter: `alt=media`
  - Authentication: predefined Google Drive OAuth2
  - Response format: file
  - Output binary property name: `filename.txt`
- **Key expressions / variables used:**
  - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}`
- **Input / output connections:**
  - Input from `Search Drive Files`
  - Output to `Extract Content from File`
- **Version-specific requirements:** typeVersion `4.4`
- **Edge cases / failures:**
  - Fails if search returns no item
  - Fails if selected file is not downloadable text
  - OAuth scope or Drive ACL issues
- **Sub-workflow reference:** none

#### Extract Content from File
- **Type / role:** `n8n-nodes-base.extractFromFile` — converts binary file to text
- **Configuration choices:**
  - Operation: `text`
  - Binary property: `filename.txt`
- **Key expressions / variables used:** none
- **Input / output connections:**
  - Input from `Fetch File Details`
  - Output to `Content Analysis Agent`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases / failures:**
  - Binary property name mismatch
  - Unsupported file encoding
  - Empty file content leads to empty sponsor input
- **Sub-workflow reference:** none

---

## 2.5 AI Metadata Generation

### Overview
This block uses Gemini through an n8n LangChain Agent to transform transcript and sponsor text into structured YouTube metadata and a thumbnail prompt. A structured output parser enforces a JSON schema-like response format.

### Nodes Involved
- Content Analysis Agent
- Gemini Chat Model
- Parse Structured Output
- Gemini Analysis Model

### Node Details

#### Content Analysis Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt execution with language model and parser
- **Configuration choices:**
  - Prompt type: defined manually
  - Output parser enabled
  - Prompt asks for:
    - transcript analysis
    - title under 70 characters
    - formatted description
    - 15–20 tags
    - 3–5 hashtags
    - thumbnail prompt
    - sponsor extraction
  - Requires strict JSON output with keys:
    - `video_title`
    - `video_description`
    - `tags`
    - `hashtags`
    - `thumbnail_prompt`
    - `sponsors`
- **Key expressions / variables used:**
  - Transcript: `{{ $('Post Audio to Lemonfox').item.json.data }}`
  - Sponsors: `{{ $json.data }}`
- **Input / output connections:**
  - Main input from `Extract Content from File`
  - AI language model input from `Gemini Chat Model`
  - AI output parser input from `Parse Structured Output`
  - Main output to `Generate Thumbnail Image`
- **Version-specific requirements:** typeVersion `3.1`
- **Edge cases / failures:**
  - If sponsor file is missing upstream, this node may never receive input at all
  - Transcript is taken from `Post Audio to Lemonfox` raw data, not from cleaned SRT output
  - Prompt contains typos (“nothingh”, “seperately”), which likely do not break execution but may affect model clarity
  - Large transcripts may exceed context limits or increase latency
  - Model may still return malformed JSON; parser attempts to repair
- **Sub-workflow reference:** none

#### Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — primary Gemini chat model for the agent
- **Configuration choices:**
  - Model: `models/gemini-3.1-flash-lite-preview`
- **Key expressions / variables used:** none
- **Input / output connections:**
  - AI language model output to `Content Analysis Agent`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Google AI credential/auth issues
  - Preview model naming may become unavailable or deprecated
  - Model output quality may vary for long transcripts
- **Sub-workflow reference:** none

#### Parse Structured Output
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates/repairs model output into structured JSON
- **Configuration choices:**
  - `autoFix: true`
  - Uses example JSON schema shape for expected fields
- **Key expressions / variables used:** none
- **Input / output connections:**
  - AI output parser connected into `Content Analysis Agent`
  - Receives AI language model from `Gemini Analysis Model`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:**
  - Auto-fix may still fail on severely malformed output
  - Example-based schema is not as strict as a formal JSON schema validator
- **Sub-workflow reference:** none

#### Gemini Analysis Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — model attached to the structured parser path
- **Configuration choices:**
  - Model: `models/gemini-3.1-flash-lite-preview`
- **Key expressions / variables used:** none
- **Input / output connections:**
  - AI language model output to `Parse Structured Output`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Same credential and model availability risks as the other Gemini model node
- **Sub-workflow reference:** none

**Important implementation note:**  
This workflow uses two Gemini chat model nodes:
- one connected directly to the agent
- one connected to the parser

This is unusual but valid in some n8n LangChain setups. The parser node itself is supplied with a model. In practice, reproduce exactly as wired in the workflow unless you intentionally simplify after testing.

---

## 2.6 Thumbnail Generation and Final Upload

### Overview
This block uses the AI-generated thumbnail prompt to create an image, then uploads the video and metadata through upload-post, and finally removes the public Drive permission granted earlier.

### Nodes Involved
- Generate Thumbnail Image
- Upload to YouTube API
- Delete Original File permissions in Drive

### Node Details

#### Generate Thumbnail Image
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini image generation
- **Configuration choices:**
  - Resource: `image`
  - Model: `models/gemini-2.5-flash-image`
  - Prompt comes from `Content Analysis Agent` structured output
- **Key expressions / variables used:**
  - Prompt: `{{ $json.output.thumbnail_prompt }}`
- **Input / output connections:**
  - Input from `Content Analysis Agent`
  - Output to `Upload to YouTube API`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases / failures:**
  - Image model availability may change
  - Prompt safety filtering may block generation
  - If `thumbnail_prompt` is missing, expression fails
  - Binary output property must remain compatible with downstream upload node
- **Sub-workflow reference:** none

#### Upload to YouTube API
- **Type / role:** `n8n-nodes-base.httpRequest` — posts final package to upload-post service
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.upload-post.com/api/upload`
  - Content type: multipart form-data
  - Custom header:
    - `Authorization: Apikey enter-your-api-key`
  - Form fields:
    - `video`: public Google Drive download URL
    - `title`: AI-generated title
    - `description`: AI-generated description
    - `user`: placeholder username
    - `platform[]`: `youtube`
    - `thumbnail`: binary field from image generation output, field name `data`
- **Key expressions / variables used:**
  - Video URL: `https://drive.google.com/uc?export=download&id={{ $('When File Added in Drive').item.json.id }}`
  - Title: `{{ $('Content Analysis Agent').item.json.output.video_title }}`
  - Description: `{{ $('Content Analysis Agent').item.json.output.video_description }}`
- **Input / output connections:**
  - Input from `Generate Thumbnail Image`
  - Output to `Delete Original File permissions in Drive`
- **Version-specific requirements:** typeVersion `4.4`
- **Edge cases / failures:**
  - Placeholder values must be replaced before production use
  - Upload-post API errors, auth failures, or unsupported media formats
  - Thumbnail binary property must be `data`; if Gemini output changes, upload breaks
  - Only title and description are sent; tags and hashtags are not forwarded to this API in the current implementation
- **Sub-workflow reference:** none

#### Delete Original File permissions in Drive
- **Type / role:** `n8n-nodes-base.httpRequest` — removes the temporary public permission from Google Drive
- **Configuration choices:**
  - Method: `DELETE`
  - URL targets permission ID `anyoneWithLink`
  - Auth: predefined Google Drive OAuth2
- **Key expressions / variables used:**
  - URL: `https://www.googleapis.com/drive/v3/files/{{ $('When File Added in Drive').item.json.id }}/permissions/anyoneWithLink`
- **Input / output connections:**
  - Input from `Upload to YouTube API`
  - No downstream node
- **Version-specific requirements:** typeVersion `4.4`
- **Edge cases / failures:**
  - The permission ID `anyoneWithLink` may not always match the actual created permission ID in all Drive API situations
  - If upload fails, cleanup never runs because there is no error path
  - If sharing was not created successfully, delete may 404
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When File Added in Drive | n8n-nodes-base.googleDriveTrigger | Watches a specific Google Drive folder for newly created files |  | If Video Mime Type | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Trigger and access setup<br>Handles initial file trigger and provides temporary access. |
| If Video Mime Type | n8n-nodes-base.if | Filters for video MIME types only | When File Added in Drive | Grant Temp File Access | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Trigger and access setup<br>Handles initial file trigger and provides temporary access. |
| Grant Temp File Access | n8n-nodes-base.httpRequest | Creates public read access on the uploaded Drive file | If Video Mime Type | Post Audio to Lemonfox | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Trigger and access setup<br>Handles initial file trigger and provides temporary access. |
| Post Audio to Lemonfox | n8n-nodes-base.httpRequest | Sends the public video URL to LemonFox for SRT transcription | Grant Temp File Access | Clean SRT Content | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Transcribe and clean SRT<br>Transcribes the video file and cleans the output. |
| Clean SRT Content | n8n-nodes-base.code | Normalizes escaped transcription content into a clean SRT string | Post Audio to Lemonfox | Create Text File in Drive | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Transcribe and clean SRT<br>Transcribes the video file and cleans the output. |
| Create Text File in Drive | n8n-nodes-base.googleDrive | Saves the cleaned transcript back to Google Drive as an SRT text file | Clean SRT Content | Search Drive Files | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Manage processed file content<br>Saves text and retrieves file for processing. |
| Search Drive Files | n8n-nodes-base.googleDrive | Searches for a companion sponsor file in Drive | Create Text File in Drive | Fetch File Details | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Manage processed file content<br>Saves text and retrieves file for processing. |
| Fetch File Details | n8n-nodes-base.httpRequest | Downloads the sponsor file content from Google Drive | Search Drive Files | Extract Content from File | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Manage processed file content<br>Saves text and retrieves file for processing. |
| Extract Content from File | n8n-nodes-base.extractFromFile | Extracts plain text from the downloaded sponsor file | Fetch File Details | Content Analysis Agent | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Process with AI agent<br>Uses AI to analyze content and generate imagery. |
| Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Supplies Gemini chat capability to the AI agent |  | Content Analysis Agent | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Process with AI agent<br>Uses AI to analyze content and generate imagery. |
| Parse Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output for SEO metadata |  | Content Analysis Agent | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Process with AI agent<br>Uses AI to analyze content and generate imagery. |
| Gemini Analysis Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides the model connection used by the structured output parser |  | Parse Structured Output | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Process with AI agent<br>Uses AI to analyze content and generate imagery. |
| Content Analysis Agent | @n8n/n8n-nodes-langchain.agent | Generates SEO metadata, sponsor extraction, and thumbnail prompt | Extract Content from File; Gemini Chat Model; Parse Structured Output | Generate Thumbnail Image | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Process with AI agent<br>Uses AI to analyze content and generate imagery. |
| Generate Thumbnail Image | @n8n/n8n-nodes-langchain.googleGemini | Generates a thumbnail image from the AI-created thumbnail prompt | Content Analysis Agent | Upload to YouTube API | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Upload and finalize results<br>Uploads the final media and cleans up resources. |
| Upload to YouTube API | n8n-nodes-base.httpRequest | Uploads video, title, description, and thumbnail to upload-post for YouTube publishing | Generate Thumbnail Image | Delete Original File permissions in Drive | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Upload and finalize results<br>Uploads the final media and cleans up resources. |
| Delete Original File permissions in Drive | n8n-nodes-base.httpRequest | Removes temporary public sharing from the original Drive file | Upload to YouTube API |  | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters.<br>## Upload and finalize results<br>Uploads the final media and cleans up resources. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for the full workflow |  |  | ## Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox<br>### How it works<br>1. A Google Drive trigger initiates the workflow upon new file detection. 2. The system retrieves temporary access and sends the file to a transcription service. 3. Text content is cleaned, saved, and processed to extract key information. 4. An AI agent analyzes the content and generates a representative image. 5. The final output is uploaded to YouTube, followed by automated cleanup of temporary resources.<br>### Setup steps<br>- Configure the Google Drive Trigger with appropriate folder permissions.<br>- Connect LemonFox credentials for the audio transcription service.<br>- Set up Google Gemini credentials for the AI agent and image generation.<br>- Configure YouTube API credentials for the final video upload.<br>### Customization<br>Adjust the AI prompt in the Gemini node to change the style of the generated imagery or to provide more specific summarization parameters. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for trigger and access section |  |  | ## Trigger and access setup<br>Handles initial file trigger and provides temporary access. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for transcription section |  |  | ## Transcribe and clean SRT<br>Transcribes the video file and cleans the output. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for file management section |  |  | ## Manage processed file content<br>Saves text and retrieves file for processing. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for AI section |  |  | ## Process with AI agent<br>Uses AI to analyze content and generate imagery. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation for upload/finalization section |  |  | ## Upload and finalize results<br>Uploads the final media and cleans up resources. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it the title:  
   `Create YouTube SEO metadata and thumbnails from Google Drive videos with Gemini, LemonFox and upload-post`

2. **Add a Google Drive Trigger node** named `When File Added in Drive`.
   - Node type: `Google Drive Trigger`
   - Event: `File Created`
   - Trigger on: `Specific Folder`
   - Polling: every minute
   - Select the folder to watch
   - Credentials: Google Drive OAuth2 with permission to read files in the watched folder

3. **Add an If node** named `If Video Mime Type`.
   - Condition type: string
   - Left value: `{{ $json.mimeType }}`
   - Operation: `contains`
   - Right value: `video`
   - Connect `When File Added in Drive` → `If Video Mime Type`

4. **Add an HTTP Request node** named `Grant Temp File Access`.
   - Method: `POST`
   - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}/permissions`
   - Authentication: predefined credential type
   - Credential type: Google Drive OAuth2 API
   - Send body: enabled
   - Body parameters:
     - `role` = `reader`
     - `type` = `anyone`
   - Connect the true output of `If Video Mime Type` → `Grant Temp File Access`

5. **Add an HTTP Request node** named `Post Audio to Lemonfox`.
   - Method: `POST`
   - URL: `https://api.lemonfox.ai/v1/audio/transcriptions`
   - Authentication: generic credential type
   - Generic auth type: HTTP Bearer Auth
   - Create/store LemonFox bearer credential
   - Send body: enabled
   - Content type: `form-urlencoded`
   - Body parameters:
     - `file` = `https://drive.google.com/uc?id={{ $('When File Added in Drive').item.json.id }}&export=download`
     - `language` = `english`
     - `response_format` = `srt`
   - Connect `Grant Temp File Access` → `Post Audio to Lemonfox`

6. **Add a Code node** named `Clean SRT Content`.
   - Paste this logic conceptually:
     - Read `data` from the LemonFox response
     - Remove wrapping quotes if present
     - Replace escaped quotes
     - Convert escaped line breaks into real line breaks
     - Collapse excessive blank lines
     - Return `{ srt: cleanedText }`
   - Use the same JavaScript from the workflow if you want identical behavior
   - Connect `Post Audio to Lemonfox` → `Clean SRT Content`

7. **Add a Google Drive node** named `Create Text File in Drive`.
   - Resource: file/folder
   - Operation: `Create from Text`
   - Name: `{{ $('When File Added in Drive').item.json.name.replace(/\.[^/.]+$/, '') }}.srt`
   - Content: `{{ $json.srt }}`
   - Select destination Drive ID
   - Select destination folder ID
   - Credentials: Google Drive OAuth2
   - Connect `Clean SRT Content` → `Create Text File in Drive`

8. **Add a Google Drive node** named `Search Drive Files`.
   - Resource: file/folder
   - Search/query operation
   - Drive: `My Drive`
   - Folder: `Root`
   - Search target: `all`
   - Limit: `1`
   - Query string: `{{ $('When File Added in Drive').item.json.name.replace(/\.[^/.]+$/, '') }}_sponsors.txt`
   - Connect `Create Text File in Drive` → `Search Drive Files`

9. **Add an HTTP Request node** named `Fetch File Details`.
   - Method: `GET`
   - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}`
   - Authentication: predefined Google Drive OAuth2
   - Query parameter:
     - `alt` = `media`
   - Response format: `File`
   - Output binary property: `filename.txt`
   - Connect `Search Drive Files` → `Fetch File Details`

10. **Add an Extract From File node** named `Extract Content from File`.
    - Operation: `Text`
    - Binary property: `filename.txt`
    - Connect `Fetch File Details` → `Extract Content from File`

11. **Add a Gemini Chat Model node** named `Gemini Chat Model`.
    - Node type: Google Gemini chat model for LangChain
    - Model: `models/gemini-3.1-flash-lite-preview`
    - Credentials: Google Gemini / Google AI credentials

12. **Add a Structured Output Parser node** named `Parse Structured Output`.
    - Enable auto-fix
    - Provide example JSON structure with:
      - `video_title`
      - `video_description`
      - `tags`
      - `hashtags`
      - `thumbnail_prompt`
      - `sponsors` array containing `sponsor_name` and `sponsor_email`

13. **Add another Gemini Chat Model node** named `Gemini Analysis Model`.
    - Same type as above
    - Model: `models/gemini-3.1-flash-lite-preview`
    - Connect `Gemini Analysis Model` → `Parse Structured Output` through the AI language model connection

14. **Add an AI Agent node** named `Content Analysis Agent`.
    - Node type: LangChain Agent
    - Prompt type: define manually
    - Enable output parser
    - Paste the long prompt from the workflow, preserving:
      - transcript input from `{{ $('Post Audio to Lemonfox').item.json.data }}`
      - sponsors input from `{{ $json.data }}`
      - strict JSON return format
    - Connect:
      - `Extract Content from File` → `Content Analysis Agent` as main input
      - `Gemini Chat Model` → `Content Analysis Agent` as AI language model
      - `Parse Structured Output` → `Content Analysis Agent` as AI output parser

15. **Add a Google Gemini Image node** named `Generate Thumbnail Image`.
    - Resource: `image`
    - Model: `models/gemini-2.5-flash-image`
    - Prompt: `{{ $json.output.thumbnail_prompt }}`
    - Credentials: Google Gemini / Google AI
    - Connect `Content Analysis Agent` → `Generate Thumbnail Image`

16. **Add an HTTP Request node** named `Upload to YouTube API`.
    - Method: `POST`
    - URL: `https://api.upload-post.com/api/upload`
    - Content type: `multipart-form-data`
    - Send headers: enabled
    - Header:
      - `Authorization` = `Apikey enter-your-api-key`
    - Body parameters:
      - `video` = `https://drive.google.com/uc?export=download&id={{ $('When File Added in Drive').item.json.id }}`
      - `title` = `{{ $('Content Analysis Agent').item.json.output.video_title }}`
      - `description` = `{{ $('Content Analysis Agent').item.json.output.video_description }}`
      - `user` = `enter-your-username`
      - `platform[]` = `youtube`
      - `thumbnail` = binary form-data field from input binary property `data`
    - Connect `Generate Thumbnail Image` → `Upload to YouTube API`

17. **Add an HTTP Request node** named `Delete Original File permissions in Drive`.
    - Method: `DELETE`
    - URL: `https://www.googleapis.com/drive/v3/files/{{ $('When File Added in Drive').item.json.id }}/permissions/anyoneWithLink`
    - Authentication: predefined Google Drive OAuth2
    - Connect `Upload to YouTube API` → `Delete Original File permissions in Drive`

18. **Optionally add sticky notes** matching the original canvas sections:
    - Overall workflow description
    - Trigger and access setup
    - Transcribe and clean SRT
    - Manage processed file content
    - Process with AI agent
    - Upload and finalize results

19. **Configure credentials** before activation:
    - **Google Drive OAuth2**
      - Used by trigger, file creation, file search, file download, permission creation, permission deletion
      - Must allow file read/write and permission management
    - **LemonFox bearer token**
      - Used in `Post Audio to Lemonfox`
    - **Google Gemini / Google AI credentials**
      - Used in both Gemini chat nodes and image generation node
    - **upload-post API key**
      - Insert into `Authorization` header in `Upload to YouTube API`

20. **Test with a sample video file** placed in the watched Drive folder.
    - Also create a companion sponsor file named:
      - `<video-base-name>_sponsors.txt`
    - Example:
      - video: `episode01.mp4`
      - sponsor file: `episode01_sponsors.txt`

21. **Validate outputs**:
    - Transcript file is created in Drive as `<video-base-name>.srt`
    - AI output contains expected structured fields
    - Thumbnail is generated successfully
    - upload-post accepts the submission
    - Public Drive permission is removed afterward

## Important rebuild caveats

22. **Handle missing sponsor files if needed.**
    - The current workflow assumes the sponsor file exists.
    - To make the workflow robust, add:
      - an IF branch after `Search Drive Files`, or
      - “Continue On Fail” behavior plus fallback empty sponsor text

23. **Consider using the cleaned transcript instead of the raw transcript in the AI prompt.**
    - Current prompt references `$('Post Audio to Lemonfox').item.json.data`
    - If you want cleaner AI input, change it to reference the cleaned SRT or extracted plain transcript text

24. **Replace placeholders before production use.**
    - `enter-your-api-key`
    - `enter-your-username`
    - blank Drive IDs / folder IDs in Drive nodes

25. **Consider adding error handling paths.**
    - Cleanup currently only runs if upload succeeds
    - A more resilient version would always attempt permission removal via an error workflow or merge/finally pattern

## Sub-workflow setup
- This workflow does **not** invoke any sub-workflow.
- It has **one entry point**: `When File Added in Drive`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated YouTube SEO, Thumbnail Generation & Video Upload via AI, Google Drive & LemonFox | Overall workflow concept |
| Setup guidance included in canvas notes: configure Google Drive Trigger permissions, LemonFox credentials, Google Gemini credentials, and YouTube/upload credentials | Internal workflow documentation |
| Customization note: adjust the AI prompt in the Gemini node to change image style or summarization behavior | Prompt tuning guidance |
| The final upload is not performed by a native YouTube node; it is sent to `upload-post.com` through an HTTP API | `https://api.upload-post.com/api/upload` |
| LemonFox is used for transcription in SRT format | `https://api.lemonfox.ai/v1/audio/transcriptions` |
| Google Drive temporary public links are used as exchange URLs for both transcription and final upload | Google Drive API / public file access pattern |

## Additional implementation observations
- Tags and hashtags are generated but not passed into the final upload request.
- The sponsor file naming convention differs from the transcript file naming convention:
  - transcript: `<video-name>.srt`
  - sponsor companion file: `<video-name>_sponsors.txt`
- The cleanup step assumes the public permission can be deleted using `anyoneWithLink`; verify this against your Drive API behavior.
- The workflow will stall if the sponsor file is absent unless you add fallback logic.