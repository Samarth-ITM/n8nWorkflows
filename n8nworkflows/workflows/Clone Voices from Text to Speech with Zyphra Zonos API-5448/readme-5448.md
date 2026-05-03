Clone Voices from Text to Speech with Zyphra Zonos API

https://n8nworkflows.xyz/workflows/clone-voices-from-text-to-speech-with-zyphra-zonos-api-5448


# Clone Voices from Text to Speech with Zyphra Zonos API

### 1. Workflow Overview

This n8n workflow enables voice cloning from text using the Zyphra Zonos Text-to-Speech API. It accepts a text input along with a sample voice audio file and various parameters to synthesize speech that mimics the given voice sample. The workflow is triggered by an HTTP POST webhook, processes the input parameters, reads the voice sample file from local storage, encodes it, sends a request to the Zyphra API to generate cloned speech audio, saves the resulting audio file locally, and finally responds with a JSON summary.

**Target Use Cases:**  
- Automated voice generation for applications requiring personalized voice synthesis.  
- Testing and prototyping voice cloning with Zyphra’s API using local sample files.  
- Integrating voice cloning capabilities into a broader automation pipeline via webhook triggers.

**Logical Blocks:**  
- **1.1 Webhook Input and Parameter Setup:** Receives POST requests, extracts and sets parameters including text, voice sample path, emotions, and output preferences.  
- **1.2 Sample Voice File Handling:** Reads the sample voice audio from local storage and converts it to a base64 string.  
- **1.3 File Validation:** Checks if the voice sample file was successfully loaded.  
- **1.4 Zyphra API Call:** Sends the prepared data to the Zyphra voice cloning API and handles the API response.  
- **1.5 Output Handling:** Saves the cloned audio file locally and prepares a success response.  
- **1.6 Error Handling and Responses:** Manages errors related to missing files or API failures and sends appropriate JSON error responses.  

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Input and Parameter Setup

- **Overview:**  
  This block starts the workflow upon receiving an HTTP POST webhook call to `/voice-clone`. It extracts required and optional parameters from the incoming JSON body, applying default values where necessary.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Set Clone Parameters

- **Node Details:**  

  - **Webhook Trigger**  
    - *Type:* Webhook Trigger  
    - *Technical Role:* Acts as the entry point for the workflow, listening for POST requests at `/voice-clone`.  
    - *Configuration:* HTTP Method set to POST; webhook path set as `voice-clone`. No authentication configured (can be added externally).  
    - *Key Expressions:* Accesses incoming JSON body for parameters like `text`, `sample_voice_path`, `emotion`, etc.  
    - *Input/Output:* No input; outputs JSON body payload.  
    - *Edge Cases:* Invalid or malformed POST payload; missing required parameters; webhook path conflicts.  
    - *Version:* 1  

  - **Set Clone Parameters**  
    - *Type:* Set  
    - *Technical Role:* Normalizes and cleans input parameters, applying defaults for missing optional fields.  
    - *Configuration:* Sets multiple numeric emotion parameters (`happiness`, `neutral`, `sadness`, etc.) with default fallback values using expressions like `{{$json.body.emotion.happiness || 0.6}}`. Also sets strings like `text` (default: test message), `speaking_rate` (default: 15), `language_iso_code` (default: "en-us"), `mime_type` (default: "audio/wav"), `model` (default: "zonos-v0.1-transformer"), and paths (`sample_voice_path` and `output_path`).  
    - *Key Expressions:* Uses JavaScript expressions to fall back on defaults if parameters are missing.  
    - *Input:* JSON from webhook trigger.  
    - *Output:* Cleaned parameter set for downstream nodes.  
    - *Edge Cases:* Missing or invalid parameter types; empty or non-existent file paths; incorrect emotion value ranges.  
    - *Version:* 1  

---

#### 1.2 Sample Voice File Handling

- **Overview:**  
  Reads the sample voice audio file from the specified local path and converts the binary data into a base64-encoded string to be included in the API request.

- **Nodes Involved:**  
  - Read Sample Voice  
  - Base64 convertor

- **Node Details:**  

  - **Read Sample Voice**  
    - *Type:* Read Binary File (Read/Write File node)  
    - *Technical Role:* Reads the audio file specified by the `sample_voice_path` parameter.  
    - *Configuration:* Uses dynamic expression for fileSelector based on `sample_voice_path`. MIME type set dynamically from parameters. On error, configured to continue error output to allow custom error handling.  
    - *Input:* Parameters from Set Clone Parameters.  
    - *Output:* Binary data of the audio file or error.  
    - *Edge Cases:* File not found, permission denied, invalid path, unsupported file type.  
    - *Version:* 1  

  - **Base64 convertor**  
    - *Type:* Code (JavaScript)  
    - *Technical Role:* Converts binary audio data read by the previous node into a base64 string for API consumption.  
    - *Configuration:* Custom JS code extracts the `data` property from binary data and returns it as `base64_content` in JSON. Throws error if no binary data found.  
    - *Input:* Binary output from Read Sample Voice.  
    - *Output:* JSON with base64-encoded audio content.  
    - *Edge Cases:* Missing or empty binary data causing an error; malformed binary.  
    - *Version:* 2  

---

#### 1.3 File Validation

- **Overview:**  
  Verifies that the sample voice file was successfully loaded and is not empty before proceeding to call the API.

- **Nodes Involved:**  
  - Check File Loaded

- **Node Details:**  

  - **Check File Loaded**  
    - *Type:* If condition  
    - *Technical Role:* Checks existence and validity of binary data by testing if base64 content is present.  
    - *Configuration:* Condition checks if `$binary.data` exists and is not false. Uses strict validation and case sensitivity.  
    - *Input:* JSON from Base64 convertor.  
    - *Output:* Two branches: true (file loaded) leads to API call; false leads to file error response.  
    - *Edge Cases:* Empty or corrupted binary data; false negatives due to expression errors.  
    - *Version:* 2  

---

#### 1.4 Zyphra API Call

- **Overview:**  
  Sends the text, voice sample (base64), emotion parameters, and other settings to Zyphra’s Text-to-Speech API endpoint to generate cloned speech.

- **Nodes Involved:**  
  - Call Zyphra Clone API

- **Node Details:**  

  - **Call Zyphra Clone API**  
    - *Type:* HTTP Request  
    - *Technical Role:* Makes a POST request to Zyphra’s TTS clone endpoint with JSON body containing all parameters including the encoded sample voice.  
    - *Configuration:*  
      - URL: `http://api.zyphra.com/v1/audio/text-to-speech`  
      - Method: POST  
      - Headers: Includes `X-API-Key` which must be replaced with a valid Zyphra API key.  
      - Body: JSON with dynamic expressions pulling parameters from `Set Clone Parameters` and base64 audio from previous node.  
      - On error: Continue error output to allow error response handling.  
    - *Input:* JSON with parameters and base64 audio.  
    - *Output:* API response JSON (success or error).  
    - *Edge Cases:* Authentication failure due to missing/invalid API key; API timeout; invalid payload; network errors.  
    - *Version:* 4.1  

---

#### 1.5 Output Handling

- **Overview:**  
  Saves the cloned audio file returned by Zyphra API to the specified output directory with a timestamped filename, then prepares a success JSON response.

- **Nodes Involved:**  
  - Save Cloned Audio  
  - Success Response  
  - Webhook Response

- **Node Details:**  

  - **Save Cloned Audio**  
    - *Type:* Read/Write File  
    - *Technical Role:* Writes the cloned audio file to local storage.  
    - *Configuration:*  
      - Filename dynamically constructed from `output_path` parameter plus prefix `cloned_voice_` and current timestamp, with `.webm` extension.  
      - Operation set to write mode.  
    - *Input:* API response binary data (assumed).  
    - *Output:* Passes filename info downstream.  
    - *Edge Cases:* Write permission errors; invalid output path; disk full.  
    - *Version:* 1  

  - **Success Response**  
    - *Type:* Set  
    - *Technical Role:* Sets JSON response indicating success, including message, filename, processed text, and sample voice path used.  
    - *Configuration:* Static strings and dynamic expressions referencing previous nodes.  
    - *Input:* Output metadata from Save Cloned Audio.  
    - *Output:* JSON success payload.  
    - *Edge Cases:* Missing filename; partial writes.  
    - *Version:* 1  

  - **Webhook Response**  
    - *Type:* Respond to Webhook  
    - *Technical Role:* Sends the final JSON response back to the webhook caller.  
    - *Configuration:* Responds with JSON using the data from Success Response or error response nodes.  
    - *Input:* Success or error JSON.  
    - *Output:* HTTP response.  
    - *Edge Cases:* Network issues sending response; malformed JSON.  
    - *Version:* 1  

---

#### 1.6 Error Handling and Responses

- **Overview:**  
  Handles error scenarios by preparing specific JSON error responses for file loading failures or API errors and sending them back to the webhook caller.

- **Nodes Involved:**  
  - File Error Response  
  - API Error Response  
  - Webhook Response (shared)

- **Node Details:**  

  - **File Error Response**  
    - *Type:* Set  
    - *Technical Role:* Creates JSON error message for missing or unreadable sample voice file.  
    - *Configuration:* Sets `success: false`, `error_type: file_not_found`, with descriptive message and suggestion.  
    - *Input:* From failed branch of Read Sample Voice or Check File Loaded.  
    - *Output:* JSON error payload.  
    - *Edge Cases:* Incorrect error routing; insufficient info for debugging.  
    - *Version:* 1  

  - **API Error Response**  
    - *Type:* Set  
    - *Technical Role:* Creates JSON error message for Zyphra API failures including authentication or other errors.  
    - *Configuration:* Sets `success: false`, `error_type: api_error`, includes error message from API response or unknown error default, plus suggestion to check API key.  
    - *Input:* From failed branch of Call Zyphra Clone API.  
    - *Output:* JSON error payload.  
    - *Edge Cases:* Missing error details; API returns non-standard error structure.  
    - *Version:* 1  

  - **Webhook Response** (described above)  
    - Used to send error JSON back to the caller.  

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                         |
|-----------------------|---------------------|----------------------------------------|-------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Webhook Trigger       | Webhook Trigger     | Entry point, receives webhook POST     | None                    | Set Clone Parameters         | 🎙️ VOICE CLONING WORKFLOW - ZONOS API - Basic setup instructions and input/output descriptions                      |
| Set Clone Parameters  | Set                 | Extracts and sets input parameters     | Webhook Trigger         | Read Sample Voice            | 📥 REQUIRED WEBHOOK PARAMETERS - Required and optional params description                                           |
| Read Sample Voice     | Read/Write File     | Reads sample voice audio from disk     | Set Clone Parameters    | Base64 convertor, File Error Response | 📁 FILE HANDLING PROCESS - Steps and file existence importance                                                     |
| Base64 convertor      | Code (JavaScript)   | Converts binary audio to base64 string | Read Sample Voice       | Check File Loaded            |                                                                                                                     |
| Check File Loaded     | If                  | Validates sample voice file loaded     | Base64 convertor        | Call Zyphra Clone API, File Error Response |                                                                                                                     |
| Call Zyphra Clone API | HTTP Request        | Sends voice clone request to Zyphra API | Check File Loaded       | Save Cloned Audio, API Error Response | 🔐 API KEY CONFIGURATION REQUIRED! - API key setup instructions                                                   |
| Save Cloned Audio     | Read/Write File     | Saves generated audio locally           | Call Zyphra Clone API   | Success Response             | 🎵 OUTPUT DETAILS - Output location, format, and naming explanation                                                |
| Success Response      | Set                 | Sets success JSON response              | Save Cloned Audio       | Webhook Response             |                                                                                                                     |
| File Error Response   | Set                 | Sets file error JSON response           | Read Sample Voice, Check File Loaded | Webhook Response             | 🚨 ERROR HANDLING & RESPONSES - Error types and messages description                                               |
| API Error Response    | Set                 | Sets API error JSON response            | Call Zyphra Clone API   | Webhook Response             | 🚨 ERROR HANDLING & RESPONSES - Error types and messages description                                               |
| Webhook Response      | Respond to Webhook  | Sends final JSON response to caller     | Success Response, File Error Response, API Error Response | None                       |                                                                                                                     |
| Sticky Note           | Sticky Note         | Informational notes                     | None                    | None                        | Multiple notes covering usage, parameters, errors, API key, file handling, and example payload                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `voice-clone`  
   - No authentication (optional to add).  

2. **Create Set Node "Set Clone Parameters"**  
   - Extract values from incoming JSON body:  
     - Numeric: `happiness`, `neutral`, `sadness`, `disgust`, `fear`, `surprise`, `anger`, `other` with defaults (e.g., 0.6, 0.05, etc.)  
     - String: `text` (default: "Hello, this is a test of voice cloning!"), `speaking_rate` (default: 15), `language_iso_code` (default: "en-us"), `mime_type` (default: "audio/wav"), `model` (default: "zonos-v0.1-transformer"), `sample_voice_path`, `output_path`.  
   - Use expressions to assign defaults if parameters are missing.  

3. **Create Read/Write File Node "Read Sample Voice"**  
   - Operation: Read  
   - File Selector: Set to dynamic expression from `sample_voice_path`  
   - MIME Type: Dynamic from `mime_type`  
   - On Error: Continue error output (to handle missing file gracefully).  

4. **Create Code Node "Base64 convertor"**  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const binaryData = $input.first().binary?.data;
     if (!binaryData) throw new Error("No binary data found");
     return { json: { base64_content: binaryData.data } };
     ```  
   - Input: Binary from "Read Sample Voice".  

5. **Create If Node "Check File Loaded"**  
   - Condition: Check if `$binary.data` exists and is not false (expression: `{{$ifEmpty($binary.data, false) !== false}}`)  
   - True branch: proceed to API call  
   - False branch: file error response  

6. **Create HTTP Request Node "Call Zyphra Clone API"**  
   - URL: `http://api.zyphra.com/v1/audio/text-to-speech`  
   - Method: POST  
   - Headers: Add header `X-API-Key` with your Zyphra API key value  
   - Body Type: JSON  
   - Body: Construct JSON with fields:  
     - `text`, `speaking_rate`, `language_iso_code`, `mime_type`, `model` from Set Clone Parameters  
     - `emotion` object with all emotion parameters  
     - `speaker_audio` set to base64 content from previous node  
   - On Error: Continue error output to handle API errors gracefully  

7. **Create Read/Write File Node "Save Cloned Audio"**  
   - Operation: Write  
   - Filename: Dynamic expression combining `output_path` plus `"cloned_voice_" + current timestamp + ".webm"`  
   - Content: Audio data from API response  

8. **Create Set Node "Success Response"**  
   - Fields:  
     - `success`: true  
     - `message`: "Voice cloning completed successfully!"  
     - `filename`: filename from Save Cloned Audio node  
     - `text_processed`: original text parameter  
     - `sample_voice_used`: sample voice path parameter  

9. **Create Set Node "File Error Response"**  
   - Fields:  
     - `success`: false  
     - `error_type`: "file_not_found"  
     - `message`: "Sample voice file could not be read from: [sample_voice_path]" (dynamic)  
     - `suggestion`: "Please check the file path and ensure the file exists"  

10. **Create Set Node "API Error Response"**  
    - Fields:  
      - `success`: false  
      - `error_type`: "api_error"  
      - `message`: "Voice cloning API request failed"  
      - `error_details`: from API error message if present, else "Unknown API error"  
      - `suggestion`: "Please check your API key and try again"  

11. **Create Respond to Webhook Node "Webhook Response"**  
    - Respond With: JSON  
    - Response Body: Use JSON from success or error nodes  

12. **Connect Nodes:**  
    - Webhook Trigger → Set Clone Parameters → Read Sample Voice → Base64 convertor → Check File Loaded  
    - Check File Loaded true → Call Zyphra Clone API → Save Cloned Audio → Success Response → Webhook Response  
    - Check File Loaded false → File Error Response → Webhook Response  
    - Call Zyphra Clone API error → API Error Response → Webhook Response  

13. **Credentials:**  
    - No special credentials needed for file nodes.  
    - For HTTP Request node, add the Zyphra API key in the header parameter `X-API-Key`.  
    - Ensure local file paths used in `sample_voice_path` and `output_path` are accessible by n8n.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 🎙️ VOICE CLONING WORKFLOW - ZONOS API: Basic setup and usage instructions; configure Zyphra API key; ensure sample voice files are accessible; test webhook endpoint accessibility.                            | Sticky Note on workflow start node                                                             |
| 📥 REQUIRED WEBHOOK PARAMETERS: `text`, `sample_voice_path`, `output_path` are mandatory; optional parameters include speaking rate, language ISO code, mime type, model, and detailed emotion settings.       | Sticky Note near Set Clone Parameters node                                                     |
| 🔐 API KEY CONFIGURATION REQUIRED: Register at [playground.zyphra.com](https://playground.zyphra.com); obtain API key from settings; add key to HTTP header as `X-API-Key`.                                   | Sticky Note near API call node                                                                 |
| 📁 FILE HANDLING PROCESS: Steps for reading sample voice file, converting to base64, and validating file existence; importance of correct file paths.                                                       | Sticky Note near file handling nodes                                                           |
| 🚨 ERROR HANDLING & RESPONSES: Defines error types returned for missing files, API errors, and authentication issues, with user-friendly messages and suggestions.                                         | Sticky Note near error response nodes                                                          |
| 🎵 OUTPUT DETAILS: Generated audio file saved with timestamped filename under specified output path; format matches requested MIME type; success response includes file info for download/access.            | Sticky Note near output saving nodes                                                           |
| Example POST request payload provided for webhook testing includes all required and optional parameters with sample values for quick testing and validation.                                                | Sticky Note near workflow start nodes                                                          |

---

This document fully describes the "Text to speech Voice clone using Zonos via API (local storage)" workflow, enabling users or AI agents to understand, reproduce, and maintain the automation reliably.