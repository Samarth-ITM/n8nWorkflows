Create Multi-Speaker Podcasts with Google Sheets, ElevenLabs v3, and Drive

https://n8nworkflows.xyz/workflows/create-multi-speaker-podcasts-with-google-sheets--elevenlabs-v3--and-drive-7883


# Create Multi-Speaker Podcasts with Google Sheets, ElevenLabs v3, and Drive

### 1. Workflow Overview

This n8n workflow automates the creation of multi-speaker podcasts by transforming dialogue text from a Google Sheets document into expressive, natural-sounding audio using the ElevenLabs v3 Text-to-Dialogue API, then uploads the generated audio file to Google Drive.

**Target Use Cases:**  
- Content creators producing podcasts with multiple speakers and expressive dialogue.  
- Game developers or storytellers requiring immersive, multi-character audio dialogue.  
- Audiobook producers seeking natural and emotionally varied voice narration.

**Logical Blocks:**  

- **1.1 Input Reception:** Manual trigger to start the workflow.  
- **1.2 Data Retrieval:** Fetch dialogue data from a structured Google Sheet.  
- **1.3 Data Preparation:** Transform raw sheet data into the format required by ElevenLabs API.  
- **1.4 AI Processing:** Send prepared dialogue to ElevenLabs API to generate audio.  
- **1.5 Output Storage:** Upload the generated podcast audio file to Google Drive.  
- **1.6 Documentation/Help:** Sticky notes providing usage tips, background, and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Starts the workflow manually via user interaction.

- **Nodes Involved:**  
`When clicking ‘Execute workflow’`

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Initiates the workflow manually.  
  - **Configuration:** No parameters; triggers execution on user command.  
  - **Connections:** Outputs to `Get dialogue` node.  
  - **Edge cases:** None inherent; user must manually trigger.  
  - **Version Requirements:** n8n Manual Trigger node v1.

#### 2.2 Data Retrieval

- **Overview:**  
Fetches dialogue data from a specified Google Sheets document, including speaker names, voice IDs, and text lines.

- **Nodes Involved:**  
`Get dialogue`

- **Node Details:**  
  - **Type:** Google Sheets  
  - **Role:** Reads rows from Google Sheets to obtain raw dialogue input.  
  - **Configuration:**  
    - Document ID linked to a specific Google Sheet containing podcast dialogue data.  
    - Sheet name specified by gid=0 (first sheet).  
  - **Expressions/Variables:** None except static references to sheet and document.  
  - **Connections:** Input from Manual Trigger; output to `Prepare dialogue`.  
  - **Credentials:** Google Sheets OAuth2 (authorized Google account).  
  - **Edge cases:**  
    - Authentication failure if credentials expire or revoke.  
    - Empty or malformed sheets causing missing data.  
    - Rate limits on Google Sheets API.  
  - **Version Requirements:** Google Sheets node v4.7.

#### 2.3 Data Preparation

- **Overview:**  
Transforms the raw rows from Google Sheets into a JSON structure compatible with ElevenLabs API, mapping each dialogue entry to text and voice ID fields.

- **Nodes Involved:**  
`Prepare dialogue`

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Converts spreadsheet data into an array of dialogue objects with `text` and `voice_id`.  
  - **Configuration:**  
    - Loops through all input items.  
    - For each item, extracts `Input` as text and `Voice ID` as voice identifier.  
    - Constructs an array assigned to `podcast` key in output JSON.  
  - **Expressions:** Uses `$input.all()` to access all incoming items.  
  - **Connections:** Input from `Get dialogue`; output to `Generate podcast`.  
  - **Edge cases:**  
    - Missing or empty `Input` or `Voice ID` fields can produce incomplete entries.  
    - JavaScript runtime errors if input JSON structure changes.  
  - **Version Requirements:** Code node v2.

#### 2.4 AI Processing

- **Overview:**  
Calls the ElevenLabs v3 Text-to-Dialogue API to generate an audio podcast file from the prepared dialogue JSON.

- **Nodes Involved:**  
`Generate podcast`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Sends POST request to ElevenLabs API endpoint `/v1/text-to-dialogue` with dialogue JSON.  
  - **Configuration:**  
    - URL: `https://api.elevenlabs.io/v1/text-to-dialogue`  
    - Method: POST  
    - Body: JSON with `model_id` set to `"eleven_v3"` and `inputs` set to the stringified `podcast` array from previous step.  
    - Headers: `Content-Type: application/json` and authorization header with API key under name `xi-api-key` (configured in credentials).  
  - **Expressions:** Uses JSON.stringify to embed dialogue array into the request body.  
  - **Connections:** Input from `Prepare dialogue`; output to `Upload file`.  
  - **Credentials:** HTTP Header Auth with ElevenLabs API key.  
  - **Edge cases:**  
    - API key invalid or expired causing authorization errors.  
    - Network timeouts or API rate limits.  
    - Malformed request JSON causing API errors.  
  - **Version Requirements:** HTTP Request node v4.2.

#### 2.5 Output Storage

- **Overview:**  
Uploads the generated podcast audio file to a specified folder in Google Drive, naming it based on the current timestamp.

- **Nodes Involved:**  
`Upload file`

- **Node Details:**  
  - **Type:** Google Drive  
  - **Role:** Saves the audio file received from ElevenLabs API to Google Drive.  
  - **Configuration:**  
    - File name dynamically generated as `podcast_YYYYMMDDHHmm` based on current time.  
    - Drive: “My Drive”  
    - Folder ID: Specified folder (Elevenlabs folder).  
  - **Connections:** Input from `Generate podcast`; no further outputs.  
  - **Credentials:** Google Drive OAuth2 (authorized Google account).  
  - **Edge cases:**  
    - Authentication failure due to OAuth token expiry.  
    - Insufficient permissions or quota in Google Drive.  
    - File upload failure due to network issues.  
  - **Version Requirements:** Google Drive node v3.

#### 2.6 Documentation/Help

- **Overview:**  
Sticky note nodes provide contextual information, usage tips, and setup instructions for users.

- **Nodes Involved:**  
`Sticky Note`, `Sticky Note1`, `Sticky Note2`

- **Node Details:**  
  - **Type:** Sticky Note  
  - **Role:** Explain usage scenarios, provide tips on formatting dialogue text with audio events, and preliminary setup steps such as cloning the Google Sheet and setting API keys.  
  - **Content Summary:**  
    - `Sticky Note1`: Overview of ElevenLabs v3 Text-to-Dialogue API and its capabilities with a link to ElevenLabs v3 demo.  
    - `Sticky Note2`: Preliminary setup instructions including cloning the Google Sheet, creating an ElevenLabs account, setting voice IDs, and API key placement.  
    - `Sticky Note`: Examples of non-speech audio events to enhance dialogue expressiveness.  
  - **Connections:** None (informational only).  
  - **Edge cases:** N/A

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                          | Input Node(s)                 | Output Node(s)        | Sticky Note                                                                                  |
|-------------------------------|-------------------|----------------------------------------|------------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Initiates workflow manually             | —                            | Get dialogue           |                                                                                              |
| Get dialogue                  | Google Sheets      | Retrieves raw dialogue data from sheet  | When clicking ‘Execute workflow’ | Prepare dialogue       |                                                                                              |
| Prepare dialogue             | Code               | Transforms sheet data to API format     | Get dialogue                 | Generate podcast       |                                                                                              |
| Generate podcast             | HTTP Request       | Calls ElevenLabs API to generate audio | Prepare dialogue             | Upload file            |                                                                                              |
| Upload file                  | Google Drive       | Uploads generated podcast to Drive      | Generate podcast             | —                      |                                                                                              |
| Sticky Note                  | Sticky Note        | Provides tips on non-speech audio events| —                            | —                      | Non-speech audio event examples for expressive dialogue                                      |
| Sticky Note1                 | Sticky Note        | Explains ElevenLabs v3 API usage         | —                            | —                      | Overview and use cases for ElevenLabs Text-to-Dialogue API with link: https://try.elevenlabs.io/ahkbf00hocnu |
| Sticky Note2                 | Sticky Note        | Setup instructions for sheet and API    | —                            | —                      | Preliminary steps including cloning sheet, setting voice IDs, and API key setup              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - No specific parameters needed. This will start the workflow on user command.

2. **Create Google Sheets Node (Get dialogue):**  
   - Type: Google Sheets  
   - Operation: Read rows from sheet  
   - Document ID: Use the ID of your Google Sheet containing the podcast dialogue.  
   - Sheet Name: Use first sheet or specify by gid=0.  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Connect Manual Trigger output to this node’s input.

3. **Create Code Node (Prepare dialogue):**  
   - Type: Code (JavaScript)  
   - Paste the following code to transform input data:  
     ```javascript
     const transformedItems = [];
     for (const item of $input.all()) {
       transformedItems.push({
         text: item.json.Input,
         voice_id: item.json["Voice ID"]
       });
     }
     return [{ json: { podcast: transformedItems } }];
     ```  
   - Connect `Get dialogue` output to this node.

4. **Create HTTP Request Node (Generate podcast):**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.elevenlabs.io/v1/text-to-dialogue`  
   - Authentication: HTTP Header Auth  
     - Header name: `xi-api-key`  
     - Header value: Your ElevenLabs API key  
   - Request Body: JSON format with parameters:  
     ```json
     {
       "model_id": "eleven_v3",
       "inputs": [transformed dialogue array]
     }
     ```  
   - Use expression to stringify the `podcast` JSON from the previous node:  
     `={{ JSON.stringify($json.podcast) }}` inside `inputs` field.  
   - Set Content-Type header to `application/json`.  
   - Connect `Prepare dialogue` output to this node.

5. **Create Google Drive Node (Upload file):**  
   - Type: Google Drive  
   - Operation: Upload file  
   - Drive: My Drive  
   - Folder ID: ID of the target folder for podcasts (e.g. Elevenlabs folder)  
   - File Name: Use expression to generate timestamped name, e.g., `podcast_{{$now.format('yyyyLLddHHmm')}}`  
   - Credentials: Configure Google Drive OAuth2 credentials.  
   - Connect `Generate podcast` output to this node.

6. **Optional:** Add Sticky Note nodes for documentation inside the workflow for user guidance.

7. **Save and Activate Workflow.**  
   - Test by clicking “Execute workflow”.  
   - Verify Google Sheets data, ElevenLabs API response, and Google Drive upload.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| ElevenLabs v3 Text-to-Dialogue API demo and signup link. Useful for voice customization and multi-speaker dialogue generation.     | https://try.elevenlabs.io/ahkbf00hocnu                                                                  |
| The Google Sheet used in the workflow is publicly clonable for testing and modification. Ensure to clone and edit your own version. | https://docs.google.com/spreadsheets/d/1eB8iUQmhj3xJMpGam_slCS0ivtgTUpbcWAqeutG_HM8/edit?usp=sharing    |
| Non-speech audio events like [laughs], [pause], [whispers] can be inserted into dialogue text to enhance expressiveness.          | Provided in Sticky Note node within the workflow                                                          |
| Ensure API keys and OAuth credentials are kept secure and refreshed periodically to avoid workflow failures.                      | General best practice                                                                                      |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow created with n8n, respecting all content policies and containing only legal and public data.