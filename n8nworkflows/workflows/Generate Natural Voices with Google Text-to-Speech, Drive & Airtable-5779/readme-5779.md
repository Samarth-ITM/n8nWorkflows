Generate Natural Voices with Google Text-to-Speech, Drive & Airtable

https://n8nworkflows.xyz/workflows/generate-natural-voices-with-google-text-to-speech--drive---airtable-5779


# Generate Natural Voices with Google Text-to-Speech, Drive & Airtable

### 1. Workflow Overview

This workflow automates the generation of natural-sounding voiceover audio files from user-submitted text scripts using Google Text-to-Speech (TTS) technology. It is designed for content creators, marketers, developers, or anyone needing AI-generated voiceovers, providing a seamless pipeline from text input to audio file management.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception**: Captures user input (script, voice, language) via a form trigger and prepares the parameters for the TTS request.
- **1.2 Text-to-Speech Conversion**: Sends the text script to Google’s TTS API, receives the audio in base64 format, and converts it into a binary audio file.
- **1.3 Audio File Management**: Uploads the audio file to Google Drive for storage.
- **1.4 Audio Metadata Retrieval**: Requests and monitors the audio file’s duration metadata using the fal.ai ffmpeg API, handling asynchronous job status checks.
- **1.5 Database Update**: Once the audio file and metadata are ready, creates a new record in Airtable with all relevant information for asset tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives user input via a form, including the text script, voice selection, and language code. It formats these inputs to prepare for the TTS API request.

- **Nodes Involved:**  
  - On form submission2  
  - Edit Fields  
  - Sticky Note6 (Instructional)  
  - Sticky Note7 (Overview & usage instructions)  
  - Sticky Note8 (Geo restrictions warning)

- **Node Details:**

  - **On form submission2**  
    - Type: Form Trigger  
    - Role: Entry point capturing user inputs: Script (text), Voice (dropdown), Language (dropdown)  
    - Configuration: Form fields defined with pre-populated voice and language options; form description guides users.  
    - Variables: Outputs JSON with keys `Script`, `Voice`, `Langauge` (note the typo in "Langauge").  
    - Connections: Passes data to the `Edit Fields` node.  
    - Edge cases: Form typos or missing required fields can cause issues; invalid voice-language combinations may cause API errors.  
    - No sub-workflow.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Transforms form data to match Google TTS API expectations.  
    - Configuration: Creates JSON with keys:  
      - `"script"` from `Script` field  
      - `"voice"` composed as `<Langauge>-Chirp3-HD-<Voice>` (e.g., "en-US-Chirp3-HD-Aoede")  
    - Variables: Uses expressions to concatenate language and voice.  
    - Connections: Sends formatted data to `Request TTS` node.  
    - Edge cases: Typo in language field (`Langauge`) must be consistent; malformed voice name causes API errors.

  - **Sticky Notes** (Informational only)  
    - Sticky Note6: Provides tips on scripting and voice capabilities (e.g., punctuation, pause tags).  
    - Sticky Note7: Describes overall workflow and usage instructions.  
    - Sticky Note8: Warns about Google TTS geo restrictions.

---

#### 1.2 Text-to-Speech Conversion

- **Overview:**  
  Sends the formatted script and voice parameters to Google Text-to-Speech API, receives audio in base64 format, converts it to a binary file for subsequent processing.

- **Nodes Involved:**  
  - Request TTS  
  - Convert to File  
  - Sticky Note (explains base64 conversion necessity)

- **Node Details:**

  - **Request TTS**  
    - Type: HTTP Request  
    - Role: Calls Google Text-to-Speech API endpoint `text:synthesize` with POST method.  
    - Configuration:  
      - Uses Google OAuth2 credentials for authentication.  
      - Sends JSON body with:  
        - `input.markup` containing the script text (with markup support).  
        - `voice.languageCode` and `voice.name` from formatted inputs.  
        - `audio_config` with `audio_encoding` as `LINEAR16`, speaking rate 1.0.  
    - Variables: Pulls script and voice from previous node’s JSON.  
    - Connections: Outputs base64 audio content to `Convert to File`.  
    - Edge cases: API authentication errors, invalid voice/language, network timeouts, malformed JSON.  
    - Version: Uses Google OAuth2 API v4.2 credentials.

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts base64 audio string (`audioContent`) into binary file format (`data`).  
    - Configuration:  
      - Source property: `audioContent`  
      - Operation: toBinary  
    - Connections: Sends binary file data to `Upload file` node.  
    - Edge cases: If `audioContent` missing or corrupted, conversion fails.  
    - Note: Required for compatibility with Google Drive upload node.

  - **Sticky Note**  
    - Explains that Google TTS returns base64 string requiring conversion to a file before upload.

---

#### 1.3 Audio File Management

- **Overview:**  
  Uploads the converted binary audio file to a specific folder in Google Drive.

- **Nodes Involved:**  
  - Upload file

- **Node Details:**

  - **Upload file**  
    - Type: Google Drive  
    - Role: Uploads binary audio file to Google Drive folder for storage and sharing.  
    - Configuration:  
      - Target drive: "My Drive"  
      - Folder ID: a specific Google Drive folder (`1Kmvt_nPFDbAO7y6zu95c5_snNC9A1P1n`) named "Content Engine Storage"  
      - File name: taken from binary data `fileName` property  
      - Input data field: `data` (binary audio file)  
    - Credentials: Google Drive OAuth2 account  
    - Connections: Outputs metadata including `webContentLink` and `webViewLink` to `Request Duration`.  
    - Edge cases: Authentication errors, quota limits, folder permission issues, file name conflicts.

---

#### 1.4 Audio Metadata Retrieval

- **Overview:**  
  Uses fal.ai’s ffmpeg API to asynchronously retrieve the duration metadata of the uploaded audio file, with polling for job completion.

- **Nodes Involved:**  
  - Request Duration  
  - Get Status  
  - If  
  - Wait

- **Node Details:**

  - **Request Duration**  
    - Type: HTTP Request  
    - Role: Sends POST request to fal.ai ffmpeg API endpoint to analyze media URL and get metadata.  
    - Configuration:  
      - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/metadata`  
      - JSON body: `{ "media_url": "<uploaded audio file webContentLink>" }`  
      - Authentication: HTTP header with fal.ai API key  
    - Connections: Outputs job status URL to `Get Status`.  
    - Edge cases: Invalid URLs, API key errors, network issues.

  - **Get Status**  
    - Type: HTTP Request  
    - Role: Checks the status of the asynchronous metadata job using the status URL.  
    - Configuration:  
      - URL derived from previous node’s `status_url` field  
      - Authenticated with fal.ai API key  
    - Connections: Outputs job status JSON to `If` node.  
    - Edge cases: API key expiration, wrong status URL, network failure.

  - **If**  
    - Type: If node  
    - Role: Checks if the job status equals "COMPLETED".  
    - Configuration: Condition: `$json.status === "COMPLETED"`  
    - Connections:  
      - If yes → `Get Duration` node to fetch metadata result  
      - If no → `Wait` node to pause before polling again  
    - Edge cases: Unexpected status values, timing issues.

  - **Wait**  
    - Type: Wait node  
    - Role: Delays workflow for 1 second before retrying status check.  
    - Connections: Loops back to `Get Status`.  
    - Edge cases: Excessive retries may cause long delays or resource consumption.

  - **Get Duration**  
    - Type: HTTP Request  
    - Role: Retrieves final metadata response once job is completed.  
    - Configuration: URL from `response_url` in status check response, authenticated with fal.ai key.  
    - Connections: Sends complete metadata JSON to `Create a record`.  
    - Edge cases: Invalid response URL, API errors.

  - **Sticky Note1**  
    - Explains the use of fal.ai API queue system and polling with If node.

---

#### 1.5 Database Update

- **Overview:**  
  Creates a new record in Airtable to log the generated audio asset details, including URLs, duration, script description, and content type.

- **Nodes Involved:**  
  - Create a record  
  - Sticky Note2

- **Node Details:**

  - **Create a record**  
    - Type: Airtable node  
    - Role: Inserts a new row into the specified Airtable base and table with audio asset details.  
    - Configuration:  
      - Base: `appMBgjXji9DQtbY2` ("Content Engine")  
      - Table: `tblj8qQXJKjjRqaTq` ("Assets")  
      - Fields populated:  
        - `URL`: Google Drive audio `webContentLink`  
        - `Duration`: audio duration from metadata  
        - `Asset Name`: fixed string "Voiceover Audio File"  
        - `Description`: includes the original script text  
        - `WebView URL`: Google Drive `webViewLink`  
        - `Content Type`: set to "Audio"  
    - Credentials: Airtable Personal Access Token  
    - Connections: End node (no outputs)  
    - Edge cases: Airtable API rate limits, incorrect base/table/field names, token expiration.

  - **Sticky Note2**  
    - Describes Airtable usage as a media asset database.

---

### 3. Summary Table

| Node Name          | Node Type            | Functional Role                     | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                          |
|--------------------|----------------------|-----------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------------------------------|
| On form submission2 | Form Trigger         | Receives user script, voice, lang | None                  | Edit Fields           | See Sticky Note7 for workflow overview and Sticky Note6 for scripting tips; Sticky Note8 warns geo restrictions.      |
| Edit Fields        | Set                  | Formats inputs for TTS API         | On form submission2    | Request TTS           |                                                                                                                      |
| Request TTS         | HTTP Request         | Calls Google TTS API               | Edit Fields           | Convert to File       |                                                                                                                      |
| Convert to File     | Convert To File      | Converts base64 audio to binary    | Request TTS            | Upload file           | Explains necessity of conversion (see Sticky Note)                                                                   |
| Upload file         | Google Drive         | Uploads audio file to Drive        | Convert to File        | Request Duration      |                                                                                                                      |
| Request Duration    | HTTP Request         | Requests audio metadata job        | Upload file            | Get Status            |                                                                                                                      |
| Get Status          | HTTP Request         | Polls job status                   | Request Duration       | If                    |                                                                                                                      |
| If                  | If                   | Checks if job completed            | Get Status             | Get Duration (yes), Wait (no) |                                                                                                                      |
| Wait                | Wait                 | Waits before next poll             | If (no)                | Get Status            |                                                                                                                      |
| Get Duration        | HTTP Request         | Gets final audio metadata          | If (yes)               | Create a record       |                                                                                                                      |
| Create a record     | Airtable             | Logs audio asset details           | Get Duration           | None                  | Explains Airtable usage (Sticky Note2)                                                                               |
| Sticky Note7        | Sticky Note          | Workflow overview & instructions   | None                   | None                  | Covers workflow overview and usage                                                                                   |
| Sticky Note6        | Sticky Note          | Input scripting and prompting tips | None                   | None                  | Provides scripting tips and voice capabilities                                                                        |
| Sticky Note8        | Sticky Note          | Geo restrictions warning           | None                   | None                  | Warns about Google TTS geographic availability restrictions                                                          |
| Sticky Note         | Sticky Note          | Explains base64 to file conversion | None                   | None                  | Describes why Convert to File node is needed                                                                          |
| Sticky Note1        | Sticky Note          | Explains fal.ai API polling        | None                   | None                  | Details queue and polling mechanism for audio duration retrieval                                                      |
| Sticky Note2        | Sticky Note          | Airtable asset tracking explanation| None                   | None                  | Describes Airtable role for media asset management                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission2`  
   - Configure webhook with title "Generate Text-to-Speech"  
   - Add three form fields:  
     - Script (text, required)  
     - Voice (dropdown, required) with options: Aoede, Puck, Charon, Kore, Fenrir, Leda, Orus, Zephyr, Achird, Algenib, Algieba, Alnilam, Autonoe, Callirrhoe, Despina, Enceladus, Erinome, Gacrux, Iapetus, Laomedeia, Pulcherrima, Rasalgethi, Sadachbia, Sadaltager, Schedar, Sulafat, Umbriel, Vindemiatrix, Zubenelgenubi, Achernar  
     - Langauge (dropdown, required) with options: en-US, en-AU, en-GB, en-IN, es-US, de-DE, fr-FR, hi-IN, pt-BR, ar-XA, es-ES, fr-CA, id-ID, it-IT, ja-JP, tr-TR, vi-VN, bn-IN, gu-IN, kn-IN, ml-IN, mr-IN, ta-IN, te-IN, nl-BE, nl-NL, ko-KR, cmn-CN, pl-PL, ru-RU, sw-KE, th-TH, ur-IN, uk-UA  
   - Include form description: "Add your text you want generated and select a voice."

2. **Add a Set node**  
   - Name: `Edit Fields`  
   - Mode: Raw JSON  
   - Set JSON output to:  
     ```json
     {
       "script": "{{ $json.Script }}",
       "voice": "{{ $json.Langauge }}-Chirp3-HD-{{ $json.Voice }}"
     }
     ```  
   - Connect `On form submission2` output to this node.

3. **Add HTTP Request node to call Google TTS**  
   - Name: `Request TTS`  
   - HTTP Method: POST  
   - URL: `https://texttospeech.googleapis.com/v1/text:synthesize`  
   - Authentication: Google OAuth2 (configure credentials with Google Cloud OAuth2 client having Text-to-Speech API enabled)  
   - Body content type: JSON  
   - JSON Body:  
     ```json
     {
       "input": {
         "markup": "{{ $json.script }}"
       },
       "voice": {
         "languageCode": "{{ $('On form submission2').item.json.Langauge }}",
         "name": "{{ $json.voice }}"
       },
       "audio_config": {
         "audio_encoding": "LINEAR16",
         "speaking_rate": 1.0
       }
     }
     ```  
   - Connect `Edit Fields` to this node.

4. **Add Convert To File node**  
   - Name: `Convert to File`  
   - Operation: toBinary  
   - Source Property: `audioContent`  
   - Connect `Request TTS` to this node.

5. **Add Google Drive node**  
   - Name: `Upload file`  
   - Operation: Upload file  
   - Drive: My Drive  
   - Folder ID: `1Kmvt_nPFDbAO7y6zu95c5_snNC9A1P1n` (Content Engine Storage folder)  
   - File name: from binary property `fileName`  
   - Input data field: `data` (binary)  
   - Authentication: Google Drive OAuth2 credentials  
   - Connect `Convert to File` to this node.

6. **Add HTTP Request node to request audio duration**  
   - Name: `Request Duration`  
   - HTTP Method: POST  
   - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/metadata`  
   - Authentication: HTTP Header Auth with fal.ai API key  
   - JSON Body: `{"media_url": "{{ $json.webContentLink }}"}`  
   - Connect `Upload file` to this node.

7. **Add HTTP Request node to get job status**  
   - Name: `Get Status`  
   - HTTP Method: GET  
   - URL: `={{ $json.status_url }}` (from previous node response)  
   - Authentication: HTTP Header Auth with fal.ai API key  
   - Connect `Request Duration` to this node.

8. **Add If node**  
   - Name: `If`  
   - Condition: Check if `$json.status === "COMPLETED"`  
   - Connect `Get Status` output to this node.

9. **Add Wait node**  
   - Name: `Wait`  
   - Wait time: 1 second  
   - Connect `If` node’s false output to this node.

10. **Loop Wait back to Get Status**  
    - Connect `Wait` node output to `Get Status` node input.

11. **Add HTTP Request node to retrieve final metadata**  
    - Name: `Get Duration`  
    - HTTP Method: GET  
    - URL: `={{ $json.response_url }}` (from If node’s true branch)  
    - Authentication: HTTP Header Auth with fal.ai API key  
    - Connect `If` node’s true output to this node.

12. **Add Airtable node**  
    - Name: `Create a record`  
    - Operation: Create  
    - Base: `appMBgjXji9DQtbY2` (Content Engine)  
    - Table: `tblj8qQXJKjjRqaTq` (Assets)  
    - Fields to populate:  
      - URL: `={{ $('Upload file').item.json.webContentLink }}`  
      - Duration: `={{ $json.media.duration }}` (from metadata)  
      - Asset Name: `Voiceover Audio File` (static text)  
      - Description: `Script: {{ $('On form submission2').item.json.Script }}`  
      - WebView URL: `={{ $('Upload file').item.json.webViewLink }}`  
      - Content Type: `Audio` (static)  
    - Authentication: Airtable Personal Access Token  
    - Connect `Get Duration` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow demonstrates how to use Google's Chirp 3 HD voices for natural-sounding text-to-speech generation. The voices support punctuation and pause tags.    | [Google Chirp 3 HD Scripting Tips](https://cloud.google.com/text-to-speech/docs/chirp3-hd#scripting-and-prompting-tips) |
| Google TTS API is region-restricted; ensure your account and usage comply with supported countries.                                                                  | Sticky Note8 warning in workflow                                                                   |
| fal.ai API uses a queue-based system requiring status polling to get media metadata such as audio duration.                                                          | fal.ai API documentation: https://queue.fal.run/fal-ai/ffmpeg-api/metadata                         |
| Airtable is used as a media asset database to store generated audio metadata and links. Ensure your Airtable base and table have matching fields.                   | Airtable: https://www.airtable.com                                                                |
| For the Convert To File node, read more here: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.converttofile/                                      | n8n documentation                                                                                  |
| Community support is available at the [n8n Forum](https://community.n8n.io/)                                                                                          | n8n Community                                                                                      |

---

**Disclaimer:** The text provided is from an automated workflow created with n8n, respecting content policies and handling only legal, public data.