Convert PDF Articles to Audio Podcasts with Google TTS & Cloudflare R2

https://n8nworkflows.xyz/workflows/convert-pdf-articles-to-audio-podcasts-with-google-tts---cloudflare-r2-9521


# Convert PDF Articles to Audio Podcasts with Google TTS & Cloudflare R2

### 1. Workflow Overview

This workflow automates the conversion of PDF articles into audio podcasts using Google Cloud Text-to-Speech (TTS) and Cloudflare R2 object storage. It targets users who want to listen to long-form content such as research papers or study materials in podcast format, supporting multitasking or on-the-go learning. The workflow handles PDF upload, text extraction and cleaning, section detection and splitting, TTS conversion, audio stitching, cloud storage, RSS feed generation, and email notification.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Configuration:** Handles PDF upload via web form and sets up workflow-wide configuration variables.
- **1.2 Text Extraction and Preparation:** Extracts text from the PDF and cleans/formats it for processing.
- **1.3 Section Detection and Splitting:** Detects logical sections in the text and splits long sections into manageable chunks for TTS.
- **1.4 TTS Usage Control:** Checks monthly TTS usage limits to avoid exceeding quotas.
- **1.5 Text-to-Speech Conversion:** Converts text chunks to audio using Google Cloud TTS API.
- **1.6 Audio Processing:** Converts audio from base64 to binary and stitches all audio segments into a single MP3 file.
- **1.7 Cloud Storage Upload:** Uploads individual audio files and the stitched MP3 plus RSS feed XML file to Cloudflare R2.
- **1.8 RSS Feed Generation:** Generates a rich iTunes-compatible RSS feed XML describing the podcast and episodes.
- **1.9 Usage Tracking and Notification:** Updates monthly usage statistics and sends an email notification with podcast details and RSS feed URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Configuration

- **Overview:**  
  Initiates the workflow by receiving PDF files uploaded through a web form and sets essential workflow-wide configuration parameters.

- **Nodes Involved:**  
  - Upload PDF for Podcast  
  - ⚙️ Workflow Config  
  - Merge

- **Node Details:**

  - **Upload PDF for Podcast**  
    - *Type:* Form Trigger  
    - *Role:* Provides a web form interface allowing users to upload a single PDF file to start the conversion process.  
    - *Configuration:* Accepts only one PDF file; required field; triggers workflow immediately after upload.  
    - *Inputs:* None (trigger node)  
    - *Outputs:* PDF binary file to Extract PDF Text node and passes data to Workflow Config node.  
    - *Edge cases:* Uploading unsupported file types is prevented by acceptFileTypes filter; large PDFs may cause timeouts downstream.

  - **⚙️ Workflow Config**  
    - *Type:* Set  
    - *Role:* Defines global constants such as Cloudflare R2 bucket name, public URL, RSS feed filename, and podcast artwork URL.  
    - *Configuration:* Hardcoded string variables set for R2_PUBLIC_URL, R2_BUCKET, RSS_FILENAME, PODCAST_ARTWORK_URL.  
    - *Inputs:* Receives data from Upload PDF for Podcast node.  
    - *Outputs:* Passes config data to Merge node.  
    - *Edge cases:* Misconfiguration of these values will cause file upload or RSS feed URL errors.

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines the binary data from PDF upload flow and the workflow config data into a single item stream for downstream processing.  
    - *Configuration:* Mode set to "combine by position" to merge corresponding items from both inputs.  
    - *Inputs:* Receives from Upload MP3 to R2 (audio processing) and ⚙️ Workflow Config (configuration).  
    - *Outputs:* Sends combined data to Build RSS XML node.  
    - *Edge cases:* Mismatched input counts can cause merging issues.

---

#### 2.2 Text Extraction and Preparation

- **Overview:**  
  Extracts textual content from the uploaded PDF and cleans it to remove unwanted characters and sections, preparing it for section detection.

- **Nodes Involved:**  
  - Extract PDF Text  
  - Clean & Process Text

- **Node Details:**

  - **Extract PDF Text**  
    - *Type:* Read PDF  
    - *Role:* Extracts raw text from the uploaded PDF binary file.  
    - *Configuration:* Reads from binaryPropertyName "pdfFile".  
    - *Inputs:* Receives PDF binary from Upload PDF for Podcast.  
    - *Outputs:* Outputs extracted text as JSON to Clean & Process Text.  
    - *Edge cases:* Corrupt or encrypted PDFs may fail extraction.

  - **Clean & Process Text**  
    - *Type:* Code (JavaScript)  
    - *Role:* Cleans extracted text by removing form feeds, multiple newlines, page numbers, and specific unwanted text patterns. Calculates metadata like word count, character count, and estimated reading time.  
    - *Key expressions:*  
      - `pdfText.replace(/\f/g, '\n')` to normalize form feeds  
      - Regex removal of specific course/module text (hardcoded)  
      - Word and character count calculations  
    - *Inputs:* Extracted text JSON.  
    - *Outputs:* JSON with cleanedText, documentTitle, fileName, and metadata.  
    - *Edge cases:* Hardcoded regex patterns may need adjustment for different PDFs.

---

#### 2.3 Section Detection and Splitting

- **Overview:**  
  Detects numbered sections in the cleaned text using regex, splits large sections into chunks not exceeding a byte size limit, and prepares episode metadata.

- **Nodes Involved:**  
  - Detect Sections & Split

- **Node Details:**

  - **Detect Sections & Split**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the cleaned text to find section headings matching the pattern `\d+\.\d+` followed by uppercase titles, segments text accordingly, and splits large sections into smaller chunks under ~4500 bytes for TTS.  
    - *Key expressions:*  
      - Regex `/(\d+\.\d+)\s+([A-Z][A-Z\s]{10,})/g` to find section headings  
      - Function `splitTextIntoChunks(text, maxBytes)` that splits text on sentence boundaries without exceeding maxBytes  
      - Builds array of episode objects with episodeNumber, episodeTitle, sectionTitle, textContent, charCount, totalEpisodes  
    - *Inputs:* JSON from Clean & Process Text.  
    - *Outputs:* JSON array of episodes for TTS conversion.  
    - *Edge cases:* PDFs without recognizable section headings will result in fallback splitting into parts. Overly large sections are handled but may produce many small chunks.

---

#### 2.4 TTS Usage Control

- **Overview:**  
  Checks accumulated monthly TTS character usage against a set limit to prevent quota overruns.

- **Nodes Involved:**  
  - Check TTS Usage Limit

- **Node Details:**

  - **Check TTS Usage Limit**  
    - *Type:* Code (JavaScript)  
    - *Role:* Uses workflow static data storage scoped globally to store usage per month. Compares current usage plus new request characters against a monthly limit of 950,000 characters. Throws error if limit exceeded.  
    - *Key expressions:*  
      - Month key format: `YYYY-MM`  
      - Usage stored as `{ charCount, requestCount }`  
    - *Inputs:* One episode item at a time (runOnceForEachItem).  
    - *Outputs:* Passes episode JSON with usageInfo metadata.  
    - *Edge cases:* Static data persistence is critical; reset or corruption can cause inaccurate limits. Concurrency could cause race conditions.

---

#### 2.5 Text-to-Speech Conversion

- **Overview:**  
  Converts each episode's text content into speech using Google Cloud Text-to-Speech API with WaveNet voices.

- **Nodes Involved:**  
  - Google TTS API

- **Node Details:**

  - **Google TTS API**  
    - *Type:* HTTP Request  
    - *Role:* Calls Google TTS API endpoint to synthesize speech from text content.  
    - *Configuration:*  
      - POST to `https://texttospeech.googleapis.com/v1/text:synthesize`  
      - Voice: en-US, Wavenet-D  
      - Audio encoding: MP3  
      - Speaking rate and pitch set to normal  
      - Auth: HTTP Header with API key or OAuth token (configurable credential)  
      - Timeout 60 seconds, retry on failure enabled with 2-second wait  
    - *Inputs:* Episode JSON with textContent from Check TTS Usage Limit.  
    - *Outputs:* JSON with base64 audioContent.  
    - *Edge cases:* API rate limits, authentication failures, large text causing timeout or truncation. "OnError" set to continue to allow workflow to proceed even if one TTS call fails.

---

#### 2.6 Audio Processing

- **Overview:**  
  Converts base64 audio from Google TTS into binary format, generates safe file names, and stitches all audio segments into one complete MP3 file.

- **Nodes Involved:**  
  - Convert Audio to Binary  
  - Stitch All Mp3 Together  
  - Merge

- **Node Details:**

  - **Convert Audio to Binary**  
    - *Type:* Code (JavaScript)  
    - *Role:* Converts base64 audioContent to binary buffer and assigns metadata such as fileName, mimeType, and episode information.  
    - *Key expressions:*  
      - Creates sanitized file names using episode title and date  
      - Returns binary data under key `audioFile` for downstream upload  
    - *Inputs:* Google TTS API JSON output (including audioContent).  
    - *Outputs:* Item with binary audioFile and metadata JSON.  
    - *Edge cases:* Skips items with missing audioContent (logs and returns null).

  - **Stitch All Mp3 Together**  
    - *Type:* Code (JavaScript)  
    - *Role:* Sorts all episode audio items by episodeNumber, combines their buffers into one MP3 buffer, and creates a single stitched episode metadata object.  
    - *Key expressions:*  
      - Buffer.concat to merge MP3 buffers  
      - Generates combined base64 audio for binary output  
      - Creates combined file name with current date  
    - *Inputs:* All outputs from Convert Audio to Binary (all episodes).  
    - *Outputs:* Single combined audio item with binary audioFile and metadata.  
    - *Edge cases:* Episode numbering missing or malformed may affect sort order.

  - **Merge** (reused)  
    - Combines output of Stitch All Mp3 Together with workflow config data for RSS generation.

---

#### 2.7 Cloud Storage Upload

- **Overview:**  
  Uploads individual audio files and generated RSS feed XML to Cloudflare R2 storage for public access.

- **Nodes Involved:**  
  - Upload MP3 to R2  
  - Upload RSS to R2

- **Node Details:**

  - **Upload MP3 to R2**  
    - *Type:* Cloudflare R2 Storage  
    - *Role:* Uploads the stitched MP3 audio file binary to the specified R2 bucket with correct content type "audio/mpeg".  
    - *Configuration:*  
      - Object key from item JSON fileName  
      - Bucket name from workflow config R2_BUCKET  
      - Binary property: audioFile  
    - *Inputs:* Binary audio file from Stitch All Mp3 Together.  
    - *Outputs:* Passes metadata to Build RSS XML node.  
    - *Edge cases:* Misconfigured bucket or permissions cause upload failures.

  - **Upload RSS to R2**  
    - *Type:* Cloudflare R2 Storage  
    - *Role:* Uploads the generated RSS feed (XML) stored as base64 in binary to R2 bucket with content type "application/rss+xml".  
    - *Configuration:*  
      - Object key from workflow config RSS_FILENAME  
      - Bucket name from workflow config R2_BUCKET  
      - Binary property: rssFile  
    - *Inputs:* RSS feed binary from Build RSS XML.  
    - *Outputs:* Passes metadata to Update Monthly Usage node.  
    - *Edge cases:* Same as above; file overwrite risks if naming conflicts.

---

#### 2.8 RSS Feed Generation

- **Overview:**  
  Constructs a well-structured RSS XML feed with iTunes podcast metadata describing the podcast, the combined audio file, and individual episodes.

- **Nodes Involved:**  
  - Build RSS XML

- **Node Details:**

  - **Build RSS XML**  
    - *Type:* Code (JavaScript)  
    - *Role:* Reads combined MP3 metadata and episodes list, uses workflow config for URLs and artwork, then generates an enhanced RSS feed XML string. Converts it to base64 for binary output.  
    - *Key expressions:*  
      - Calculates estimated duration from MP3 file size (~128 kbps)  
      - Includes iTunes-specific tags: author, owner, categories, explicit flag, episodeType  
      - Embeds episodes descriptions in plain text and HTML  
      - Adds copyright and generator metadata  
    - *Inputs:* JSON and binary data from Upload MP3 to R2 and Merge node.  
    - *Outputs:* JSON + binary rssFile property with XML content.  
    - *Edge cases:* Incorrect URLs or missing config values degrade feed quality.

---

#### 2.9 Usage Tracking and Notification

- **Overview:**  
  Updates monthly TTS usage statistics and sends an email notification summarizing the new podcast episode and usage.

- **Nodes Involved:**  
  - Update Monthly Usage  
  - Aggregate for Email  
  - Send Email

- **Node Details:**

  - **Update Monthly Usage**  
    - *Type:* Code (JavaScript)  
    - *Role:* Updates global static data tracking character count and request count per month, appending last updated timestamp.  
    - *Inputs:* JSON data from Upload RSS to R2.  
    - *Outputs:* JSON with updated usage data for email aggregation.  
    - *Edge cases:* Static data persistence critical for accurate tracking.

  - **Aggregate for Email**  
    - *Type:* Code (JavaScript)  
    - *Role:* Prepares HTML email content summarizing total episodes, character counts, file sizes, episode list, podcast RSS feed URL, and usage stats.  
    - *Inputs:* JSON data from Update Monthly Usage.  
    - *Outputs:* JSON with email subject and HTML body.  
    - *Edge cases:* Email clients may truncate complex HTML.

  - **Send Email**  
    - *Type:* Email Send  
    - *Role:* Sends the notification email to configured recipient with podcast details and RSS feed link.  
    - *Configuration:*  
      - From and To email addresses set in parameters  
      - Subject and HTML body from previous node  
      - SMTP or OAuth credentials configured in n8n  
    - *Inputs:* JSON with email content from Aggregate for Email.  
    - *Outputs:* None (end node).  
    - *Edge cases:* SMTP configuration errors or email delivery failures.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                |
|-------------------------|--------------------------------|----------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Upload PDF for Podcast   | Form Trigger                   | Receives PDF upload via web form              | None                          | Extract PDF Text, ⚙️ Workflow Config |                                                                                                                            |
| ⚙️ Workflow Config       | Set                            | Defines global configuration constants        | Upload PDF for Podcast         | Merge                          | Update config values: R2 bucket, public URL, RSS filename, podcast artwork URL                                              |
| Extract PDF Text         | Read PDF                       | Extracts raw text from PDF binary              | Upload PDF for Podcast         | Clean & Process Text            |                                                                                                                            |
| Clean & Process Text     | Code                           | Cleans and preprocesses extracted text         | Extract PDF Text               | Detect Sections & Split         |                                                                                                                            |
| Detect Sections & Split  | Code                           | Detects sections, splits large text into chunks | Clean & Process Text           | Check TTS Usage Limit           |                                                                                                                            |
| Check TTS Usage Limit    | Code                           | Checks monthly TTS usage quota                  | Detect Sections & Split        | Google TTS API                 |                                                                                                                            |
| Google TTS API           | HTTP Request                   | Converts text chunks to speech via Google TTS  | Check TTS Usage Limit          | Convert Audio to Binary         | Retry on fail enabled; continues on error to avoid full workflow failure                                                    |
| Convert Audio to Binary  | Code                           | Converts base64 audio to binary, sets metadata | Google TTS API                | Stitch All Mp3 Together         |                                                                                                                            |
| Stitch All Mp3 Together  | Code                           | Combines all audio chunks into single MP3      | Convert Audio to Binary        | Merge                          |                                                                                                                            |
| Merge                   | Merge                          | Combines audio data with workflow config       | Stitch All Mp3 Together, ⚙️ Workflow Config | Build RSS XML                   |                                                                                                                            |
| Build RSS XML            | Code                           | Generates iTunes-compatible RSS XML feed        | Upload MP3 to R2, Merge       | Upload RSS to R2                |                                                                                                                            |
| Upload MP3 to R2         | Cloudflare R2 Storage          | Uploads stitched MP3 audio file to R2 bucket   | Stitch All Mp3 Together        | Build RSS XML                  |                                                                                                                            |
| Upload RSS to R2         | Cloudflare R2 Storage          | Uploads generated RSS XML file to R2 bucket    | Build RSS XML                 | Update Monthly Usage            |                                                                                                                            |
| Update Monthly Usage     | Code                           | Updates monthly TTS usage stats in static data | Upload RSS to R2              | Aggregate for Email             |                                                                                                                            |
| Aggregate for Email      | Code                           | Builds HTML email summarizing episodes and usage | Update Monthly Usage          | Send Email                    |                                                                                                                            |
| Send Email               | Email Send                     | Sends notification email with podcast details  | Aggregate for Email           | None                          |                                                                                                                            |
| Sticky Note             | Sticky Note                    | Documentation and overview                       | None                          | None                          | # Convert PDF Articles to Podcast... See detailed notes and links in sticky note content                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node "Upload PDF for Podcast":**
   - Type: Form Trigger  
   - Configure form with one required file field named "pdfFile" accepting PDF files only.  
   - Set form title and description.  
   - This will trigger the workflow upon PDF upload.

2. **Create Set Node "⚙️ Workflow Config":**
   - Set variables:  
     - R2_PUBLIC_URL: Your Cloudflare R2 public URL, e.g. `https://pub-YOUR-R2-ID.r2.dev`  
     - R2_BUCKET: Your R2 bucket name  
     - RSS_FILENAME: Desired RSS feed filename, e.g. `your-podcast-feed.xml`  
     - PODCAST_ARTWORK_URL: URL to podcast artwork image  
   - No inputs; this node will be triggered from Upload PDF node.

3. **Connect "Upload PDF for Podcast" to "Extract PDF Text":**
   - Create Read PDF node named "Extract PDF Text"  
   - Set `binaryPropertyName` to the name of uploaded PDF file binary property ("pdfFile").

4. **Create Code Node "Clean & Process Text":**
   - Input: JSON from Extract PDF Text node  
   - Script cleans text by removing form feeds, multiple newlines, unwanted patterns, and calculates metadata (word count, char count, estimated reading minutes).  
   - Outputs cleanedText and metadata.

5. **Create Code Node "Detect Sections & Split":**
   - Input: cleanedText and metadata from previous node  
   - Script:  
     - Uses regex to detect section headings like "1.1 SECTION TITLE"  
     - Splits text into sections and further splits large sections into chunks ≤ 4500 bytes  
     - Outputs array of episode objects with titles, numbers, text chunks.

6. **Create Code Node "Check TTS Usage Limit":**
   - Runs once per episode item  
   - Checks monthly usage stored in workflow static data against 950,000 character limit  
   - Throws error if limit exceeded, otherwise passes episode data forward.

7. **Create HTTP Request Node "Google TTS API":**
   - POST to `https://texttospeech.googleapis.com/v1/text:synthesize`  
   - Body parameters: textContent, voice settings (en-US Wavenet-D), audio encoding MP3  
   - Authentication via HTTP Header (API key or OAuth token)  
   - Enable retry and set onError to continue.

8. **Create Code Node "Convert Audio to Binary":**
   - Converts base64 audioContent from Google TTS response into binary buffer  
   - Generates safe filename from episode title and date  
   - Outputs binary audioFile and metadata.

9. **Create Code Node "Stitch All Mp3 Together":**
   - Collect all audio files, sort by episodeNumber  
   - Concatenate all buffers into one MP3 buffer  
   - Output combined MP3 binary and metadata.

10. **Create Merge Node "Merge":**
    - Combine output of stitched MP3 file and workflow config data  
    - Mode: combine by position.

11. **Create Cloudflare R2 Storage Node "Upload MP3 to R2":**
    - Upload combined MP3 binary to R2 bucket  
    - Configure object key from fileName  
    - Content type: audio/mpeg  
    - Binary property: audioFile

12. **Create Code Node "Build RSS XML":**
    - Generate enhanced iTunes-compatible RSS feed XML string using combined MP3 metadata and config variables  
    - Convert XML to base64 and output as binary rssFile.

13. **Create Cloudflare R2 Storage Node "Upload RSS to R2":**
    - Upload RSS XML binary file to R2 bucket  
    - Configure object key from RSS_FILENAME  
    - Content type: application/rss+xml  
    - Binary property: rssFile

14. **Create Code Node "Update Monthly Usage":**
    - Update global static data for monthly TTS usage counters  
    - Inputs: JSON from RSS upload node  
    - Outputs updated usage data.

15. **Create Code Node "Aggregate for Email":**
    - Build HTML email content summarizing episodes, usage, RSS feed URL  
    - Inputs: Output from usage update node  
    - Outputs emailSubject and emailHTML.

16. **Create Email Send Node "Send Email":**
    - Configure SMTP or OAuth credentials  
    - Set To and From email addresses  
    - Use emailSubject and emailHTML from previous node.

17. **Connect all nodes following the order and data flow described above.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow transforms PDFs into podcasts using Google Cloud TTS and Cloudflare R2 object storage. Designed for personal use with automatic RSS feed generation compatible with major podcast apps.                             | Sticky Note content in workflow.                                                                             |
| Google Cloud Text-to-Speech API free tier offers 1M characters per month for WaveNet voices. Setup guide: https://cloud.google.com/text-to-speech/docs                                                                       | Important API usage limit and setup resource.                                                               |
| Cloudflare R2 Object Storage offers free tier with 10GB storage and unlimited egress. Setup guide: https://developers.cloudflare.com/r2/                                                                                     | Storage backend for audio and RSS files.                                                                     |
| Cloudflare R2 Storage Community Node for n8n can be installed via Settings → Community Nodes → Install → `n8n-nodes-cloudflare-r2-storage`                                                                                   | Required node installation for R2 integration.                                                               |
| Email service must be configured with SMTP or OAuth credentials (e.g., Gmail, SendGrid) for Send Email node to function correctly.                                                                                         | Email notification delivery setup.                                                                            |
| GitHub repository with full setup instructions and workflow source: https://github.com/devdutta/PDF-to-Podcast---N8N                                                                                                         | Additional detailed documentation and workflow source code reference.                                        |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. The process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.