Generate AI Voiceovers from Scripts with Gemini TTS and Upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-voiceovers-from-scripts-with-gemini-tts-and-upload-to-google-drive-10649


# Generate AI Voiceovers from Scripts with Gemini TTS and Upload to Google Drive

### 1. Workflow Overview

This workflow automates generating AI voiceovers from video script texts stored in a Google Sheet, using Google Gemini’s Text-to-Speech (TTS) API, and manages the audio files by saving them locally, converting them into a standardized format, uploading them to Google Drive, and updating the original Google Sheet with links to the generated audio files. It is designed primarily for self-hosted n8n instances due to local file handling and FFmpeg usage.

Logical blocks:

- **1.1 Input Reception & Filtering:** Reads scripts from Google Sheets and filters out those already processed.
- **1.2 Filename Sanitization:** Generates safe, unique filenames based on script content to manage file storage.
- **1.3 AI Voice Generation:** Sends scripts to Google Gemini TTS API to generate raw audio content.
- **1.4 Local File Handling:** Converts raw audio data into binary files, writes them locally, and runs FFmpeg to convert the audio format.
- **1.5 Upload to Cloud Storage:** Uploads the finalized audio files to Google Drive.
- **1.6 Updating Tracking Sheet:** Updates the Google Sheet with links to the uploaded audio files and marks scripts as processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  This block fetches video scripts from a designated Google Sheet and filters to only process scripts that have not yet been voiced.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Getting Video Scripts (Google Sheets)  
  - Filter  

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Technical Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; simply triggers workflow execution.  
    - Inputs: None  
    - Outputs: Connects to Getting Video Scripts node.  
    - Edge Cases: None typical; user must execute manually.  

  - **Getting Video Scripts**  
    - Type: Google Sheets node  
    - Technical Role: Reads rows from the Google Sheet containing video script data.  
    - Configuration:  
      - Sheet URL and sheet name point to the “Text Overlays” tab in a specified Google Sheet.  
      - Uses OAuth2 credentials linked to the user’s Google account.  
    - Key Expressions: None dynamic; static sheet and document references.  
    - Input: Trigger node  
    - Output: Raw rows with script data.  
    - Edge Cases:  
      - Auth errors if credentials expire or are invalid.  
      - Sheet or tab renamed or URL changed will cause failure.  
    - Version Requirements: Google Sheets node version 4.6 supports latest API features.  

  - **Filter**  
    - Type: Filter node  
    - Technical Role: Filters out any script rows where the “Voice Generated?” column is set to “Yes,” thus skipping already processed scripts.  
    - Configuration:  
      - Condition: `$json['Voice Generated?'] != "Yes"` (strict, case-sensitive)  
    - Input: Output from Google Sheets node  
    - Output: Passes only unvoiced scripts to next node.  
    - Edge Cases:  
      - If the column “Voice Generated?” is missing or empty, script is processed (acceptable).  
      - Case sensitivity may cause some scripts to be processed if “yes” appears in lowercase.  

#### 2.2 Filename Sanitization

- **Overview:**  
  Creates a safe, unique filename for each script to be used in file storage, avoiding invalid characters and limiting length.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Code1 (Code)  

- **Node Details:**  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Technical Role: Processes each script item individually to handle large data sets and sequential processing.  
    - Configuration: Default batch size (1) is implied; no advanced options specified.  
    - Input: Filter node output  
    - Output: Single item per batch to Code1 node; supports asynchronous processing.  
    - Edge Cases: Large sheets may cause slow processing if batch size is small.  

  - **Code1**  
    - Type: Code (JavaScript)  
    - Technical Role: Generates a sanitized, lowercase filename based on the row number and a sanitized snippet of the “full_script_text” field.  
    - Configuration:  
      - Converts script text to lowercase, replaces non-alphanumeric characters with underscores, truncates to 60 characters.  
      - Prepends the `row_number` to ensure uniqueness.  
      - Sets result as `safeFilename` in each item’s JSON.  
    - Input: Single script item from Loop Over Items  
    - Output: Modified item with `safeFilename` field  
    - Key Expressions: Uses standard JS string methods and regex.  
    - Edge Cases:  
      - Scripts shorter than 60 chars handled gracefully.  
      - Special characters replaced to avoid filesystem issues.  

#### 2.3 AI Voice Generation

- **Overview:**  
  Uses the Google Gemini TTS API to generate raw audio data from the script text.

- **Nodes Involved:**  
  - HTTP Request To Generate Voice  

- **Node Details:**  

  - **HTTP Request To Generate Voice**  
    - Type: HTTP Request  
    - Technical Role: Calls Google Gemini TTS API endpoint to generate AI voice audio content.  
    - Configuration:  
      - URL: Gemini TTS API endpoint for model `gemini-2.5-flash-preview-tts`.  
      - Method: POST  
      - Query Parameter: `key` (insert Google API Key here)  
      - JSON Body: Contains the TTS request, including the scripted prompt with an accent placeholder, voice config set to “Kore,” and audio response requested.  
    - Key Expressions:  
      - Uses `{{$json.full_script_text}}` to insert script dynamically.  
      - Prompt instructs voice tone and accent (editable).  
    - Input: From Code1 node (which provides script and safeFilename)  
    - Output: JSON containing audio data in `candidates[0].content.parts[0].inlineData.data` and `mimeType`.  
    - Edge Cases:  
      - API key missing/invalid will cause auth failure.  
      - Network timeout or rate limits from API.  
      - API response format changes may break parsing.  
    - Version: HTTP Request node version 4.2 or later recommended for JSON body support.  

#### 2.4 Local File Handling and Conversion

- **Overview:**  
  Converts raw base64 audio data to a binary file, saves it locally as .pcm, then uses FFmpeg (via command line) to convert to WAV format.

- **Nodes Involved:**  
  - Convert to File  
  - Read/Write Files from Disk (write PCM)  
  - Extracting fileName (Set)  
  - Running FFMPEG Local Code To Change Format Of File (Code)  
  - Running FFMPEG Code (Execute Command)  
  - Writing File Output Onto Disk (readWriteFile)  

- **Node Details:**  

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts base64-encoded PCM audio data into a binary file representation within n8n.  
    - Configuration:  
      - File name: Uses `safeFilename` with `.pcm` extension.  
      - MIME type: Extracted from API response field.  
      - Source Property: `candidates[0].content.parts[0].inlineData.data`  
    - Input: Output from HTTP Request node  
    - Output: Item with binary file data attached.  
    - Edge Cases:  
      - Missing or malformed base64 data causes failure.  

  - **Read/Write Files from Disk**  
    - Type: readWriteFile  
    - Role: Writes the binary PCM file to the local filesystem.  
    - Configuration:  
      - File path: User must replace placeholder path `/Users/INSERT_YOUR_LOCAL_STORAGE_HERE/` with a valid local directory writable by n8n.  
      - File name: Same as Convert to File `.pcm` name.  
      - Operation: Write  
    - Input: Convert to File output  
    - Output: Passes through file name/path info.  
    - Edge Cases:  
      - Incorrect or inaccessible file path causes write failure.  
      - Permissions issues.  

  - **Extracting fileName (Set node)**  
    - Type: Set  
    - Role: Captures the file name/path from previous node output for use in FFmpeg processing.  
    - Configuration:  
      - Sets a new field `fileName` equal to the file path from the previous node output.  
    - Input: Output from Read/Write Files from Disk  
    - Output: Prepares data for FFmpeg command generation.  

  - **Running FFMPEG Local Code To Change Format Of File**  
    - Type: Code (JavaScript)  
    - Role: Constructs the FFmpeg command to convert `.pcm` to `.wav`.  
    - Configuration:  
      - Reads `fileName` path, replaces `.pcm` suffix with `.wav`.  
      - Builds command string using FFmpeg CLI syntax with parameters for sample rate and channels.  
      - Outputs JSON containing `command`, `pcmPath`, and `wavPath`.  
    - Input: Extracting fileName node output  
    - Output: JSON with command string for execution  
    - Edge Cases:  
      - File path must be valid and accessible.  

  - **Running FFMPEG Code**  
    - Type: Execute Command  
    - Role: Runs the FFmpeg command generated to perform audio conversion.  
    - Configuration:  
      - Command is dynamically set from previous node’s `command` field.  
    - Input: FFmpeg command JSON  
    - Output: Execution result; typically no direct data passed forward except success/failure.  
    - Edge Cases:  
      - FFmpeg not installed or not in system PATH causes failure.  
      - Invalid command or file paths cause errors.  

  - **Writing File Output Onto Disk**  
    - Type: readWriteFile  
    - Role: Post-processing or verification step to handle the output `.wav` file on disk.  
    - Configuration:  
      - File Selector: Uses the `.wav` file path from FFmpeg code output.  
      - Operation: Write or read (context suggests managing file).  
    - Input: FFmpeg execution node output  
    - Output: File info passed to next node.  
    - Edge Cases:  
      - File not found if FFmpeg failed.  

#### 2.5 Upload to Cloud Storage

- **Overview:**  
  Uploads the final `.wav` audio file to a specified Google Drive folder.

- **Nodes Involved:**  
  - Uploading Wav File (Google Drive)  

- **Node Details:**  

  - **Uploading Wav File**  
    - Type: Google Drive node  
    - Role: Uploads `.wav` audio file to Google Drive.  
    - Configuration:  
      - File name: Uses `safeFilename.wav` for Google Drive.  
      - Drive: “My Drive” selected.  
      - Folder: Root folder by default (user can change).  
      - Credentials: OAuth2 linked to user’s Google Drive.  
    - Input: File path from previous node output.  
    - Output: Google Drive file metadata, including file ID and web links.  
    - Edge Cases:  
      - Permission errors if OAuth token expired or insufficient.  
      - Folder ID invalid or missing.  

#### 2.6 Updating Tracking Sheet

- **Overview:**  
  Updates the original Google Sheet to mark the script as having a generated voiceover and stores links to the uploaded audio file.

- **Nodes Involved:**  
  - Uploading Google Drive Link of File To Google Sheet (Google Sheets)  

- **Node Details:**  

  - **Uploading Google Drive Link of File To Google Sheet**  
    - Type: Google Sheets node  
    - Role: Appends or updates row with voiceover generation status and file links.  
    - Configuration:  
      - Matches rows by `full_script_text` to update correct script.  
      - Fields updated:  
        - “Voice Link” with the Drive file’s `webViewLink`  
        - “Google Drive ID” with Drive file ID  
        - “Voice Generated?” set to “Yes”  
        - “Google Drive Download Link” with `webContentLink`  
        - Also updates “Computer File Name” and “full_script_text” for tracking.  
      - Uses same sheet URL and tab as input.  
    - Input: Google Drive upload node output and Loop Over Items node to correlate data.  
    - Output: Confirmation of sheet update.  
    - Edge Cases:  
      - Auth errors or permission denied.  
      - Row matching fails if script text changed or duplicated.  
      - API rate limits.  

---

### 3. Summary Table

| Node Name                                | Node Type            | Functional Role                                | Input Node(s)                        | Output Node(s)                                | Sticky Note                                                                                                                      |
|-----------------------------------------|----------------------|------------------------------------------------|------------------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’        | Manual Trigger       | Starts the workflow manually                    | None                               | Getting Video Scripts                         |                                                                                                                                 |
| Getting Video Scripts                    | Google Sheets        | Reads video script data from Google Sheet      | When clicking ‘Execute workflow’   | Filter                                        | **👇 STEP 1: CONNECT TO YOUR SCRIPTS** Connect Google Sheets account and paste URL of scripts sheet from Part 2 workflow. Filter ensures only unvoiced scripts processed.         |
| Filter                                  | Filter               | Filters out scripts already voiced              | Getting Video Scripts              | Loop Over Items                               |                                                                                                                                 |
| Loop Over Items                         | SplitInBatches       | Processes scripts one at a time                  | Filter                            | Code1                                         |                                                                                                                                 |
| Code1                                   | Code                 | Creates safe, unique filenames for scripts      | Loop Over Items                   | HTTP Request To Generate Voice                 | **🧠 STEP 2: CONFIGURE AI VOICE** Configure Gemini API key and optionally edit voice prompt.                                    |
| HTTP Request To Generate Voice          | HTTP Request         | Calls Google Gemini TTS API to generate audio  | Code1                            | Convert to File                               |                                                                                                                                 |
| Convert to File                         | ConvertToFile        | Converts base64 audio to binary file             | HTTP Request To Generate Voice   | Read/Write Files from Disk                     | **⚠️ STEP 3: CONFIGURE LOCAL FILE PROCESSING** Requires self-hosted n8n and FFmpeg installed. Set proper local file path.       |
| Read/Write Files from Disk (write PCM)  | readWriteFile        | Writes raw PCM audio file to local disk          | Convert to File                  | Extracting fileName                           |                                                                                                                                 |
| Extracting fileName                     | Set                  | Captures file path for FFmpeg conversion         | Read/Write Files from Disk       | Running FFMPEG Local Code To Change Format Of File |                                                                                                                                 |
| Running FFMPEG Local Code To Change Format Of File | Code                 | Builds FFmpeg command to convert PCM to WAV     | Extracting fileName             | Running FFMPEG Code                           |                                                                                                                                 |
| Running FFMPEG Code                     | Execute Command      | Executes FFmpeg command to convert audio          | Running FFMPEG Local Code To Change Format Of File | Writing File Output Onto Disk               |                                                                                                                                 |
| Writing File Output Onto Disk           | readWriteFile        | Manages final WAV file on disk                     | Running FFMPEG Code             | Uploading Wav File                            |                                                                                                                                 |
| Uploading Wav File                      | Google Drive         | Uploads WAV audio file to Google Drive            | Writing File Output Onto Disk   | Uploading Google Drive Link of File To Google Sheet | **✅ STEP 4: SET UPLOAD DESTINATION** Connect Google Drive and select folder for voiceovers.                                      |
| Uploading Google Drive Link of File To Google Sheet | Google Sheets        | Updates Google Sheet with audio file links and status | Uploading Wav File             | Loop Over Items                               | **🚀 STEP 5: UPDATE TRACKING SHEET** Marks scripts as voiced and adds Google Drive audio links for tracking.                    |
| Sticky Note                            | Sticky Note          | Workflow overview and critical setup instructions | None                            | None                                          | Detailed workflow description, requirements for self-hosting, FFmpeg, and setup instructions with links.                         |
| Sticky Note1                           | Sticky Note          | Instructions for connecting scripts Google Sheet | None                            | None                                          | **👇 STEP 1: CONNECT TO YOUR SCRIPTS** message with setup instructions.                                                          |
| Sticky Note2                           | Sticky Note          | Instructions for configuring AI voice generation | None                            | None                                          | **🧠 STEP 2: CONFIGURE AI VOICE** message with API key and prompt customization instructions.                                    |
| Sticky Note3                           | Sticky Note          | Instructions for local file processing and FFmpeg setup | None                            | None                                          | **⚠️ STEP 3: CONFIGURE LOCAL FILE PROCESSING** message emphasizing self-hosted requirement and local path setup.                 |
| Sticky Note4                           | Sticky Note          | Instructions for setting upload destination      | None                            | None                                          | **✅ STEP 4: SET UPLOAD DESTINATION** message about Google Drive connection and folder selection.                                |
| Sticky Note5                           | Sticky Note          | Instructions for updating Google Sheet tracking  | None                            | None                                          | **🚀 STEP 5: UPDATE TRACKING SHEET** message about marking scripts as complete and recording links.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters.  

2. **Add Google Sheets node:**  
   - Name: `Getting Video Scripts`  
   - Operation: Read rows from a specific sheet.  
   - Set Document URL to your Google Sheet containing video scripts (output from Part 2 workflow).  
   - Set Sheet Name to the tab with scripts (e.g., "Text Overlays").  
   - Connect Google Sheets OAuth2 credentials.  
   - Connect input from Manual Trigger node.  

3. **Add Filter node:**  
   - Name: `Filter`  
   - Configure condition:  
     - Field: `Voice Generated?`  
     - Operator: `not equals`  
     - Value: `Yes` (case-sensitive)  
   - Connect input from `Getting Video Scripts`.  

4. **Add SplitInBatches node:**  
   - Name: `Loop Over Items`  
   - Default batch size (1) to process scripts sequentially.  
   - Connect input from `Filter`.  

5. **Add Code node:**  
   - Name: `Code1`  
   - JavaScript code to generate safe filename:  
     ```js
     const items = $items();
     for (const item of items) {
       const originalScript = item.json.full_script_text;
       const rowNumber = item.json.row_number;
       const sanitizedScriptPart = originalScript.toLowerCase().replace(/[^a-z0-9]/g, '_').substring(0, 60);
       const safeFilename = `${rowNumber}_${sanitizedScriptPart}`;
       item.json.safeFilename = safeFilename;
     }
     return items;
     ```  
   - Connect input from `Loop Over Items`.  

6. **Add HTTP Request node:**  
   - Name: `HTTP Request To Generate Voice`  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent`  
   - Set Query Parameter `key` to your Google Gemini API Key.  
   - Set JSON Body:  
     ```json
     {
       "contents": [
         {
           "parts": [
             {
               "text": "Say this script in a neutral <INSERT YOUR DESIRED ACCENT HERE> professional accent at a face pace enthusiastically: {{ $json.full_script_text }}"
             }
           ]
         }
       ],
       "generationConfig": {
         "responseModalities": ["AUDIO"],
         "speechConfig": {
           "voiceConfig": {
             "prebuiltVoiceConfig": {"voiceName": "Kore"}
           }
         }
       },
       "model": "gemini-2.5-flash-preview-tts"
     }
     ```  
   - Connect input from `Code1`.  

7. **Add ConvertToFile node:**  
   - Name: `Convert to File`  
   - Operation: Convert base64 to binary file.  
   - Source Property: `candidates[0].content.parts[0].inlineData.data`  
   - File Name: `={{ $('Code1').item.json.safeFilename }}.pcm`  
   - MIME Type: `={{ $json.candidates[0].content.parts[0].inlineData.mimeType }}`  
   - Connect input from `HTTP Request To Generate Voice`.  

8. **Add readWriteFile node:**  
   - Name: `Read/Write Files from Disk` (write)  
   - Operation: Write  
   - File Name: Local path where n8n can write files, e.g., `/home/n8n/audio/{{ $json.safeFilename }}.pcm` (replace placeholder with real path)  
   - Connect input from `Convert to File`.  

9. **Add Set node:**  
   - Name: `Extracting fileName`  
   - Set field: `fileName = {{ $json.fileName }}` (path from previous node)  
   - Connect input from `Read/Write Files from Disk`.  

10. **Add Code node:**  
    - Name: `Running FFMPEG Local Code To Change Format Of File`  
    - JavaScript code:  
      ```js
      const inputData = $input.all()[0].json;
      const pcmFilePath = inputData.fileName;
      const wavFilePath = pcmFilePath.replace('.pcm', '.wav');
      const ffmpegCommand = `ffmpeg -y -f s16le -ar 24000 -ac 1 -i "${pcmFilePath}" "${wavFilePath}"`;
      return { json: { pcmPath: pcmFilePath, wavPath: wavFilePath, command: ffmpegCommand } };
      ```  
    - Connect input from `Extracting fileName`.  

11. **Add Execute Command node:**  
    - Name: `Running FFMPEG Code`  
    - Command: `={{ $json.command }}`  
    - Connect input from `Running FFMPEG Local Code To Change Format Of File`.  

12. **Add readWriteFile node:**  
    - Name: `Writing File Output Onto Disk`  
    - File Selector: `={{ $('Running FFMPEG Local Code To Change Format Of File').item.json.wavPath }}`  
    - Operation: Read or manage output WAV file in local system.  
    - Connect input from `Running FFMPEG Code`.  

13. **Add Google Drive node:**  
    - Name: `Uploading Wav File`  
    - Operation: Upload file  
    - File Name: `={{ $('Code1').item.json.safeFilename }}.wav`  
    - Drive: Select “My Drive” or desired drive  
    - Folder: Select target folder or root  
    - Credentials: Connect your Google Drive OAuth2 account  
    - Connect input from `Writing File Output Onto Disk`.  

14. **Add Google Sheets node:**  
    - Name: `Uploading Google Drive Link of File To Google Sheet`  
    - Operation: Append or Update  
    - Document URL and Sheet Name: Same as `Getting Video Scripts` node  
    - Matching Columns: `full_script_text`  
    - Columns to update:  
      - `Voice Link` = `{{ $json.webViewLink }}`  
      - `Google Drive ID` = `{{ $json.id }}`  
      - `Voice Generated?` = `Yes`  
      - `Google Drive Download Link` = `{{ $json.webContentLink }}`  
      - `full_script_text` and `Computer File Name` updated accordingly  
    - Credentials: Connect same Google Sheets OAuth2 account  
    - Connect input from `Uploading Wav File`.  
    - Output connects back to `Loop Over Items` to continue processing.  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow requires a self-hosted n8n instance due to local file read/write and command execution nodes. It will **not** run on n8n Cloud. | Sticky Note (overview) |
| FFmpeg must be installed and available in the system PATH on the machine where n8n runs. Download from https://ffmpeg.org/download.html | Sticky Note (overview) |
| This is part 3 of a multi-part AI content factory series, requiring outputs from Part 2 (Generate AI Video Ad Scripts). Part 1 and 2 workflows are available for free at https://n8n.io/creators/jj-tham/ | Sticky Note (overview) |
| Google Gemini API key required for TTS calls. Replace `INSERT YOUR API KEY HERE` in HTTP Request node query parameters. | Sticky Note2, HTTP Request node |
| For local file paths, replace placeholder `/Users/INSERT_YOUR_LOCAL_STORAGE_HERE/` with a valid writable directory on your system. | Sticky Note3, Read/Write Files node |
| Google Sheets and Google Drive OAuth2 credentials must be configured correctly with sufficient permissions to read/write sheets and upload files. | Multiple nodes |
| The workflow uses filenames sanitized to avoid illegal filesystem characters and limits the length to 60 characters for compatibility. | Code1 node |
| The workflow updates the Google Sheet with links to Drive files and a “Voice Generated?” flag to avoid reprocessing scripts. | Final Google Sheets node |

---

**Disclaimer:** The provided text comes exclusively from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.