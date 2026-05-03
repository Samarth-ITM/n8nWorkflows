Generate Realistic Sound Effects with CassetteAI and Save to Google Drive

https://n8nworkflows.xyz/workflows/generate-realistic-sound-effects-with-cassetteai-and-save-to-google-drive-5158


# Generate Realistic Sound Effects with CassetteAI and Save to Google Drive

### 1. Workflow Overview

This workflow, titled **"Generate Realistic Sound Effects with CassetteAI and Save to Google Drive"**, is designed to automate the generation of high-quality, realistic sound effects using the CassetteAI Sound Effects Model. It accepts user input via a form to specify a textual prompt and desired sound duration (up to 30 seconds). The workflow then triggers the CassetteAI API to generate the audio, polls for completion, retrieves the generated audio file, and finally uploads it to a specified Google Drive folder.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user input (prompt and duration) from a web form.
- **1.2 Audio Generation Request:** Sends a request to CassetteAI to generate the sound effect audio.
- **1.3 Polling for Completion:** Waits and checks the generation status until the audio is completed.
- **1.4 Audio Retrieval and Upload:** Downloads the generated audio file and uploads it to Google Drive.
- **1.5 Informational Notes:** Sticky notes providing usage instructions, API key setup, and workflow overview.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives user input from a web form, capturing the sound effect description (prompt) and duration. It serves as the workflow's entry point.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point capturing form data with fields "Prompt" (text, required) and "Duration (max 30 sec.)" (number, required)  
    - Configuration: Webhook ID attached; form titled "Sound Effects Generator" with description  
    - Inputs: None (trigger)  
    - Outputs: JSON containing user inputs  
    - Edge cases: Missing required fields; invalid duration (e.g., above 30 seconds) may cause API rejection or errors downstream  
    - Version: 2.2

#### 1.2 Audio Generation Request

- **Overview:**  
  Sends a POST request to the CassetteAI API to generate a sound effect based on the prompt and duration provided by the user.

- **Nodes Involved:**  
  - Create audio

- **Node Details:**  
  - **Create audio**  
    - Type: HTTP Request  
    - Role: Initiates sound effect generation request to CassetteAI API  
    - Configuration: POST to `https://queue.fal.run/cassetteai/sound-effects-generator` with JSON body containing `prompt` and `duration` from form input  
    - Headers: Content-Type `application/json`, Authorization header with API key from Fal.run credentials  
    - Inputs: JSON from form submission node  
    - Outputs: JSON with request_id for polling  
    - Edge cases: HTTP errors, invalid API key, invalid JSON body, request rate limits  
    - Version: 4.2

#### 1.3 Polling for Completion

- **Overview:**  
  Implements polling logic to repeatedly check the status of the audio generation until it reaches "COMPLETED".

- **Nodes Involved:**  
  - Wait 10 sec.  
  - Get status  
  - Completed?

- **Node Details:**  
  - **Wait 10 sec.**  
    - Type: Wait  
    - Role: Delays workflow execution for 10 seconds between status checks to avoid flooding the API  
    - Configuration: Fixed 10 seconds wait  
    - Inputs: From Create audio (initial) and from Completed? (loop)  
    - Outputs: To Get status node  
    - Edge cases: Delay may cause workflow timeout if API is very slow  
    - Version: 1.1

  - **Get status**  
    - Type: HTTP Request  
    - Role: Requests the current status of the audio generation using the request_id from Create audio  
    - Configuration: GET request to `https://queue.fal.run/cassetteai/sound-effects-generator/requests/{{request_id}}/status` with Authorization header  
    - Inputs: From Wait 10 sec.  
    - Outputs: JSON containing status of generation  
    - Edge cases: HTTP errors, invalid request_id, API downtime, auth errors  
    - Version: 4.2

  - **Completed?**  
    - Type: If  
    - Role: Checks if the status returned by Get status node equals "COMPLETED"  
    - Configuration: Condition with string equality on `{{$json.status}} == "COMPLETED"`  
    - Inputs: From Get status  
    - Outputs: If true, proceeds to next block; if false, loops back to wait 10 sec.  
    - Edge cases: Unexpected status values, missing status field  
    - Version: 2.2

#### 1.4 Audio Retrieval and Upload

- **Overview:**  
  After confirming generation completion, this block retrieves the audio URL, downloads the audio file, and uploads it to a Google Drive folder.

- **Nodes Involved:**  
  - Get Audio Url  
  - Get Audio File  
  - Upload Audio

- **Node Details:**  
  - **Get Audio Url**  
    - Type: HTTP Request  
    - Role: Retrieves detailed request info including audio file metadata using request_id  
    - Configuration: GET request to `https://queue.fal.run/cassetteai/sound-effects-generator/requests/{{request_id}}` with Authorization header  
    - Inputs: From Completed? node (true branch)  
    - Outputs: JSON containing `audio_file` object with `url` and `file_name`  
    - Edge cases: HTTP errors, missing audio_file in response, auth errors  
    - Version: 4.2

  - **Get Audio File**  
    - Type: HTTP Request  
    - Role: Downloads the actual audio file using URL from Get Audio Url node  
    - Configuration: GET request to `{{$json.audio_file.url}}`  
    - Inputs: From Get Audio Url  
    - Outputs: Binary data of audio file with metadata  
    - Edge cases: File not found, download timeout, network errors  
    - Version: 4.2

  - **Upload Audio**  
    - Type: Google Drive  
    - Role: Uploads the downloaded audio file to a specified folder in Google Drive  
    - Configuration:  
      - File name dynamically generated as current timestamp + original file name extension  
      - Upload target folder specified by folder ID "1aHRwLWyrqfzoVC8HoB-YMrBvQ4tLC-NZ" ("Fal.run" folder)  
      - Google Drive OAuth2 credentials configured  
    - Inputs: Binary data from Get Audio File  
    - Outputs: None (terminal node)  
    - Edge cases: Google Drive auth failure, insufficient permissions, quota limits, file size issues  
    - Version: 3

#### 1.5 Informational Notes

- **Overview:**  
  Provides context, instructions, and guidance to users and maintainers of the workflow.

- **Nodes Involved:**  
  - Sticky Note3  
  - Sticky Note5  
  - Sticky Note6  
  - Sticky Note7

- **Node Details:**  
  - **Sticky Note3**  
    - Content: Overview of workflow purpose, highlighting CassetteAI's ability to generate realistic sound effects up to 30 seconds in 1 second processing time, and saving to Google Drive  
    - Positioned early for visibility

  - **Sticky Note5**  
    - Content: Instructions for Step 2 (Main Flow) - Start workflow and enter prompt and duration

  - **Sticky Note6**  
    - Content: Instructions for Step 1 - Obtain API key from https://fal.ai/ and set it in the "Create Image" node's header auth as "Authorization: Key YOURAPIKEY"

  - **Sticky Note7**  
    - Content: Reminder to set the API key created in Step 2

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                           |
|------------------|---------------------|------------------------------------|-----------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------|
| On form submission| Form Trigger        | Entry point capturing user input   | None                        | Create audio              | ## STEP 2 - MAIN FLOW\nStart the workflow and type the prompt and the duration (in seconds, max 30).                  |
| Create audio     | HTTP Request        | Sends generation request to API    | On form submission          | Wait 10 sec.              | ## STEP 1 - GET API KEY (YOURAPIKEY)\nCreate an account [here](https://fal.ai/) and obtain API KEY.                   |
| Wait 10 sec.     | Wait                | Delay between polling attempts     | Create audio, Completed?    | Get status                |                                                                                                                       |
| Get status       | HTTP Request        | Polls API for generation status    | Wait 10 sec.                | Completed?                |                                                                                                                       |
| Completed?       | If                  | Checks if generation is complete   | Get status                  | Wait 10 sec., Get Audio Url|                                                                                                                       |
| Get Audio Url    | HTTP Request        | Retrieves audio file metadata      | Completed? (true branch)    | Get Audio File            |                                                                                                                       |
| Get Audio File   | HTTP Request        | Downloads audio file                | Get Audio Url               | Upload Audio              |                                                                                                                       |
| Upload Audio     | Google Drive        | Uploads audio to Google Drive      | Get Audio File              | None                     | Set API Key created in Step 2                                                                                         |
| Sticky Note3     | Sticky Note         | Workflow overview                  | None                        | None                     | # Sound Effects Generator\n\nCreate stunningly realistic sound effects in seconds - CassetteAI's Sound Effects Model...|
| Sticky Note5     | Sticky Note         | Usage instructions (Step 2)        | None                        | None                     | ## STEP 2 - MAIN FLOW\nStart the workflow and type the prompt and the duration (in seconds, max 30).                  |
| Sticky Note6     | Sticky Note         | API key setup instructions         | None                        | None                     | ## STEP 1 - GET API KEY (YOURAPIKEY)\nCreate an account [here](https://fal.ai/) and obtain API KEY.                   |
| Sticky Note7     | Sticky Note         | API key reminder                   | None                        | None                     | Set API Key created in Step 2                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure webhook with a unique ID  
   - Form Title: "Sound Effects Generator"  
   - Form Description: "Sound Effects Generator"  
   - Add two fields:  
     - "Prompt" (Text, required)  
     - "Duration (max 30 sec.)" (Number, required)  

2. **Create HTTP Request Node ("Create audio"):**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://queue.fal.run/cassetteai/sound-effects-generator`  
   - Authentication: HTTP Header Auth with Fal.run API key (create credential beforehand)  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Key YOURAPIKEY (replace YOURAPIKEY with actual key)  
   - Body (JSON):  
     ```json
     {
       "prompt": "={{ $json.Prompt }}",
       "duration": {{ $json['Duration (max 30 sec.)'] }}
     }
     ```  
   - Connect output of "On form submission" to this node

3. **Create Wait Node ("Wait 10 sec."):**  
   - Type: Wait  
   - Duration: 10 seconds  
   - Connect output of "Create audio" to this node

4. **Create HTTP Request Node ("Get status"):**  
   - Type: HTTP Request  
   - Method: GET  
   - URL:  
     ```
     =https://queue.fal.run/cassetteai/sound-effects-generator/requests/{{ $('Create audio').item.json.request_id }}/status
     ```  
   - Authentication: HTTP Header Auth with Fal.run API key (reuse credential)  
   - Connect output of "Wait 10 sec." to this node

5. **Create If Node ("Completed?"):**  
   - Type: If  
   - Condition: Check if `{{$json.status}} == "COMPLETED"` (case sensitive string equality)  
   - Connect output of "Get status" to this node

6. **Connect "Completed?" node outputs:**  
   - If false: connect back to "Wait 10 sec." for polling loop  
   - If true: connect to next block "Get Audio Url"

7. **Create HTTP Request Node ("Get Audio Url"):**  
   - Type: HTTP Request  
   - Method: GET  
   - URL:  
     ```
     =https://queue.fal.run/cassetteai/sound-effects-generator/requests/{{ $json.request_id }}
     ```  
   - Authentication: HTTP Header Auth with Fal.run API key  
   - Connect true output of "Completed?" to this node

8. **Create HTTP Request Node ("Get Audio File"):**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.audio_file.url }}`  
   - Connect output of "Get Audio Url" to this node

9. **Create Google Drive Node ("Upload Audio"):**  
   - Type: Google Drive  
   - Operation: Upload File  
   - File Name: Use expression `={{ $now.format('yyyyLLddHHmmss') }}.{{ $json.audio_file.file_name }}`  
   - Drive: Select "My Drive"  
   - Folder ID: Paste folder ID `1aHRwLWyrqfzoVC8HoB-YMrBvQ4tLC-NZ` (Fal.run folder)  
   - Credentials: Set up Google Drive OAuth2 credentials with necessary permissions  
   - Connect output of "Get Audio File" to this node

10. **Add Sticky Notes (optional for documentation):**  
    - Add notes for API key setup pointing to https://fal.ai/  
    - Add overview note about the workflow purpose  
    - Add usage instructions for form inputs  
    - Place notes accordingly for clarity

**Credential Setup:**

- Create HTTP Header Auth credential for Fal.run API using your API key:  
  - Header Name: Authorization  
  - Header Value: Key YOURAPIKEY

- Create Google Drive OAuth2 credential with write permission to your desired Google Drive folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| Create an account at [fal.ai](https://fal.ai/) and obtain an API key to use the CassetteAI Sound Effects API.                                         | API key setup instruction      |
| CassetteAI Sound Effects Model generates realistic sound effects up to 30 seconds in length with only 1 second processing time.                        | Workflow overview              |
| Save generated .wav audio files automatically to a Google Drive folder for easy access and sharing.                                                    | Feature highlight              |
| Folder ID `1aHRwLWyrqfzoVC8HoB-YMrBvQ4tLC-NZ` corresponds to the "Fal.run" folder in Google Drive; change if you want to upload elsewhere.              | Google Drive folder info       |
| Ensure your Google Drive OAuth2 credentials have correct scopes to upload files and manage the target folder.                                           | Credential requirement         |

---

**Disclaimer:**  
The provided text and workflow represent an automated process using n8n and publicly available APIs. All data processed is legal and non-sensitive. The workflow respects content policies and contains no offensive or protected elements.