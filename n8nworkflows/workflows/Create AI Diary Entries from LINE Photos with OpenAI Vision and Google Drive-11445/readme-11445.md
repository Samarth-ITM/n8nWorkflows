Create AI Diary Entries from LINE Photos with OpenAI Vision and Google Drive

https://n8nworkflows.xyz/workflows/create-ai-diary-entries-from-line-photos-with-openai-vision-and-google-drive-11445


# Create AI Diary Entries from LINE Photos with OpenAI Vision and Google Drive

### 1. Workflow Overview

This workflow automates the creation of AI-generated diary entries from photos sent via the LINE messaging app. When a user sends a photo to a LINE bot, the workflow:

- Receives the photo via a webhook
- Extracts necessary metadata from the LINE message event
- Downloads the photo content from LINE
- Uses OpenAI Vision capabilities to generate a short Japanese diary-style sentence describing the photo
- Prepares a dated folder structure on Google Drive (`KidsDiary/YYYY/MM`)
- Uploads both the original photo and a text file containing the AI-generated diary sentence into the corresponding Google Drive folder
- Sends a confirmation reply back to the user on LINE

**Target Use Cases:**

- Parents or caregivers wanting an automatic kids’ photo diary
- Users who want to archive LINE photos with contextual AI-generated notes
- Anyone preferring a “take a photo and forget” journaling style

---

The workflow logic is grouped into four main blocks:

- **1.1 Input Reception:** Receiving and extracting photo data from LINE
- **1.2 AI Diary Generation & Path Preparation:** Using OpenAI Vision to generate diary text and preparing Google Drive folder paths
- **1.3 Diary File Preparation:** Combining data and creating a text file for Google Drive
- **1.4 Storage & Confirmation:** Uploading files to Google Drive and replying to the user on LINE

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Handles receiving the photo message from LINE, extracting relevant metadata, and downloading the photo binary.

**Nodes Involved:**  
- LINE Webhook  
- Extract LINE Message Data  
- Get Image from LINE  

**Node Details:**

- **LINE Webhook**  
  - Type: Webhook (n8n native)  
  - Role: Entry point receiving POST requests from LINE's messaging API when users send messages  
  - Configuration: Path set to `line-webhook`, HTTP method POST, response from last node  
  - Input: External HTTP POST from LINE platform  
  - Output: JSON event payload containing message data  
  - Edge Cases: Missing events, invalid webhook setup, unauthorized requests if LINE signature check not implemented (not shown here)  

- **Extract LINE Message Data**  
  - Type: Code (JavaScript)  
  - Role: Parse the incoming webhook JSON to extract `messageId`, `replyToken`, and message type for further processing  
  - Key Expressions: Accesses `$input.first().json.body.events` array, safely returns empty if no events  
  - Input: JSON event body from LINE Webhook  
  - Output: JSON with extracted fields and full event for reference  
  - Edge Cases: Empty events array, missing message or fields, malformed JSON  

- **Get Image from LINE**  
  - Type: HTTP Request  
  - Role: Calls LINE Content API to download the photo content in binary file format  
  - Configuration: URL templated with `messageId`, Authorization header with bearer token (placeholder `YOUR_TOKEN_HERE` to be replaced)  
  - Input: Extracted `messageId` from previous node  
  - Output: Binary photo data  
  - Edge Cases: Authorization failure, invalid/expired token, missing or invalid messageId, network timeout  

---

#### 1.2 AI Diary Generation & Path Preparation

**Overview:**  
Uses OpenAI Vision to generate a concise diary sentence based on the photo and prepares the folder path for storage on Google Drive.

**Nodes Involved:**  
- OpenAI Vision - Generate Diary  
- Prepare Folder Path  
- Create Folder Structure  

**Node Details:**

- **OpenAI Vision - Generate Diary**  
  - Type: OpenAI node via LangChain integration  
  - Role: Analyzes the photo image (input as base64 binary) and generates a short diary sentence in Japanese (~50 characters)  
  - Configuration: Model set to `gpt-4o`, operation `analyze` on image input, prompt text instructs diary style output  
  - Input: Binary photo content from "Get Image from LINE"  
  - Output: JSON with AI-generated diary sentence in `content` property  
  - Edge Cases: API quota limits, latency/timeouts, malformed image data, unexpected output formatting  

- **Prepare Folder Path**  
  - Type: Code (JavaScript)  
  - Role: Builds the Google Drive folder path string based on current date (`KidsDiary/YYYY/MM`), extracts diary text from AI output, and bundles binary image data for next steps  
  - Key Expressions: Uses `Date` object, pads month, accesses previous node output for diary text and binary image  
  - Input: AI-generated diary text and image binary  
  - Output: JSON with folder path details and binary image data attached for file uploads  
  - Edge Cases: Missing diary text fallback to empty string, date/time inconsistencies  

- **Create Folder Structure**  
  - Type: Google Drive node  
  - Role: Creates (or ensures existence of) the folder on Google Drive for the current year/month under `KidsDiary`  
  - Configuration: Folder name dynamically built as `KidsDiary_YYYY_MM`, created at root drive folder  
  - Input: Folder path info from "Prepare Folder Path"  
  - Output: Folder metadata including folder ID  
  - Edge Cases: Google Drive API rate limits, insufficient permissions, folder name conflicts  

---

#### 1.3 Diary File Preparation

**Overview:**  
Combines AI diary text and image metadata, converts diary text to a `.txt` file, and prepares it for upload.

**Nodes Involved:**  
- Merge  
- Convert to File  
- Upload Photo to diary　new  

**Node Details:**

- **Merge**  
  - Type: Merge (Combine mode)  
  - Role: Joins outputs from "Create Folder Structure" (folder info) and from "Prepare Folder Path" (binary image and diary text) into one data set for file upload  
  - Input: Two inputs, combined by position  
  - Output: Merged JSON and binary data  
  - Edge Cases: Misaligned inputs, missing data in either input  

- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts diary text string into a text file (binary) for Google Drive upload  
  - Configuration: Operation set to `toText`, source property is `diaryText` from merged data  
  - Input: Merged diary text JSON  
  - Output: Binary text file data  
  - Edge Cases: Empty diary text producing empty files  

- **Upload Photo to diary　new**  
  - Type: Google Drive node  
  - Role: Uploads the diary text `.txt` file to Google Drive, placing it inside the folder created earlier  
  - Configuration: File name timestamped with current date/time followed by `_diary.txt`, folder ID set dynamically from folder creation node output  
  - Input: Binary text file and folder ID  
  - Output: File metadata on Google Drive  
  - Edge Cases: Upload failure, permission issues, naming conflicts  

---

#### 1.4 Storage & Confirmation

**Overview:**  
Uploads the original photo to Google Drive and sends a confirmation message back to the LINE user.

**Nodes Involved:**  
- Upload Photo to Drive  
- Reply to LINE  

**Node Details:**

- **Upload Photo to Drive**  
  - Type: Google Drive node  
  - Role: Uploads the original photo binary to Google Drive under the `KidsDiary/YYYY/MM` folder with AI-generated diary text as filename  
  - Configuration: File name set dynamically using diary text, folder ID set to root (appears inconsistent with folder creation, see notes)  
  - Input: Photo binary and diary text from merged data  
  - Output: File metadata on Google Drive  
  - Edge Cases: File naming conflicts, folder ID misconfiguration, upload errors  

- **Reply to LINE**  
  - Type: HTTP Request  
  - Role: Sends a reply message to the LINE user confirming successful photo and diary saving  
  - Configuration: POST to LINE Messaging API's reply endpoint, JSON body includes `replyToken` and a fixed text message, Authorization bearer token (placeholder)  
  - Input: Reply token extracted from LINE message data earlier  
  - Output: HTTP response from LINE API  
  - Edge Cases: Invalid or expired reply token, authorization failures, network issues  

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                              | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                         |
|-------------------------------|---------------------------|----------------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------------------------------------------|
| LINE Webhook                  | Webhook                   | Receives photo messages from LINE users      | External HTTP POST            | Extract LINE Message Data     | Step 1 – Receive photo: Handles incoming photos from LINE via webhook                            |
| Extract LINE Message Data     | Code                      | Extracts messageId, replyToken, and type     | LINE Webhook                 | Get Image from LINE           | Step 1 – Receive photo: Parses event payload                                                     |
| Get Image from LINE           | HTTP Request              | Downloads photo binary from LINE content API | Extract LINE Message Data    | OpenAI Vision - Generate Diary | Step 1 – Receive photo: Downloads image binary                                                    |
| OpenAI Vision - Generate Diary| OpenAI LangChain Node     | Generates short diary text from photo        | Get Image from LINE          | Prepare Folder Path           | Step 2 – Create diary text & folder path: Uses AI to generate diary sentence                      |
| Prepare Folder Path           | Code                      | Builds folder path & prepares data for upload| OpenAI Vision - Generate Diary | Create Folder Structure, Merge | Step 2 – Create diary text & folder path: Prepares Google Drive path and bundles data             |
| Create Folder Structure       | Google Drive Folder       | Creates folder in Google Drive                | Prepare Folder Path          | Merge                        | Step 2 – Create diary text & folder path: Ensures folder exists                                  |
| Merge                        | Merge                     | Combines folder info and file data           | Create Folder Structure, Prepare Folder Path | Convert to File, Upload Photo to Drive | Step 3 – Prepare diary file for Google Drive: Joins data for file uploads                         |
| Convert to File              | Convert To File           | Converts diary text to binary text file      | Merge                       | Upload Photo to diary　new    | Step 3 – Prepare diary file for Google Drive: Creates diary text file                            |
| Upload Photo to diary　new    | Google Drive File Upload  | Uploads diary text file to Google Drive      | Convert to File             | Reply to LINE                | Step 3 – Prepare diary file for Google Drive: Saves diary text file                              |
| Upload Photo to Drive         | Google Drive File Upload  | Uploads original photo to Google Drive       | Merge                       | Reply to LINE                | Step 4 – Save original photo & confirm: Saves photo file                                        |
| Reply to LINE                | HTTP Request              | Sends confirmation message back to LINE user | Upload Photo to diary　new, Upload Photo to Drive | None                       | Step 4 – Save original photo & confirm: Confirms successful save                                |
| Sticky Note - Step 1          | Sticky Note               | Documentation for Step 1                      | None                       | None                       | Step 1 – Receive photo group explanation                                                        |
| Sticky Note - Step 2          | Sticky Note               | Documentation for Step 2                      | None                       | None                       | Step 2 – Create diary text & folder path explanation                                            |
| Sticky Note - Step 3          | Sticky Note               | Documentation for Step 3                      | None                       | None                       | Step 3 – Prepare diary file for Google Drive explanation                                       |
| Sticky Note - Step 4          | Sticky Note               | Documentation for Step 4                      | None                       | None                       | Step 4 – Save original photo & confirm explanation                                              |
| Sticky Note - Overview1       | Sticky Note               | Workflow overview and user context            | None                       | None                       | Overview of full workflow purpose and audience                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create LINE Webhook node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `line-webhook`  
   - Response Mode: Last Node  
   - No credentials needed here, but LINE webhook URL must be registered in LINE Developer Console

2. **Create Extract LINE Message Data node**  
   - Type: Code  
   - JavaScript code to parse incoming JSON to extract `messageId`, `replyToken`, and `messageType` from `body.events[0]`  
   - Connect from `LINE Webhook` output

3. **Create Get Image from LINE node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api-data.line.me/v2/bot/message/{{ $json.messageId }}/content` (dynamic from previous node)  
   - Response Format: File (binary)  
   - Headers: `Authorization: Bearer YOUR_TOKEN_HERE` (replace with actual LINE Channel Access Token)  
   - Connect from `Extract LINE Message Data`

4. **Create OpenAI Vision - Generate Diary node**  
   - Type: OpenAI (LangChain)  
   - Operation: Analyze (image)  
   - Model: `gpt-4o`  
   - Input Type: Base64 (pass image binary)  
   - Text prompt: "この写真を見て、50文字以内で日記風の一言を書いてください。"  
   - Connect from `Get Image from LINE`

5. **Create Prepare Folder Path node**  
   - Type: Code  
   - JavaScript builds folder path string `KidsDiary/YYYY/MM` based on current date  
   - Reads diary text from previous node output (`content`)  
   - Also collects binary image data from `Get Image from LINE`  
   - Connect from `OpenAI Vision - Generate Diary`

6. **Create Create Folder Structure node**  
   - Type: Google Drive (Folder)  
   - Folder Name: `KidsDiary_{{ $json.year }}_{{ $json.month }}`  
   - Drive: My Drive  
   - Parent Folder: root  
   - Connect from `Prepare Folder Path`

7. **Create Merge node**  
   - Type: Merge (Combine mode)  
   - Combine inputs by position (two inputs)  
   - Connect first input from `Create Folder Structure`  
   - Connect second input from `Prepare Folder Path` (parallel stream)

8. **Create Convert to File node**  
   - Type: Convert To File  
   - Operation: toText  
   - Source Property: `diaryText` (from merged input)  
   - Connect from `Merge`

9. **Create Upload Photo to diary　new node**  
   - Type: Google Drive (File Upload)  
   - File Name: `{{ $now.format('yyyy-MM-dd_HHmmss') }}_diary.txt` (timestamped)  
   - Drive: My Drive  
   - Folder ID: set dynamically from folder creation node output (`$json.id`)  
   - Connect from `Convert to File`

10. **Create Upload Photo to Drive node**  
    - Type: Google Drive (File Upload)  
    - File Name: `{{ $json.diaryText }}` (AI diary sentence as filename)  
    - Drive: My Drive  
    - Folder ID: root (note: consider changing to folder ID for logical consistency)  
    - Connect from `Merge`

11. **Create Reply to LINE node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Headers:  
      - Authorization: Bearer YOUR_TOKEN_HERE (replace with LINE Channel Access Token)  
      - Content-Type: application/json  
    - Body (JSON):  
      ```json
      {
        "replyToken": "{{ $('Extract LINE Message Data').first().json.replyToken }}",
        "messages": [
          {
            "type": "text",
            "text": "写真と日記を保存しました！"
          }
        ]
      }
      ```  
    - Connect from both `Upload Photo to diary new` and `Upload Photo to Drive` (parallel)

12. **Set credentials:**  
    - For Google Drive nodes: Configure OAuth2 credentials with appropriate permissions to create folders and upload files  
    - For HTTP Request nodes calling LINE API: Use a valid LINE Channel Access Token with messaging API permissions  
    - For OpenAI node: Set up OpenAI API credentials with access to GPT-4o and Vision features  

13. **Testing & Validation:**  
    - Test with real LINE messages containing photos  
    - Confirm folder creation and file uploads in Google Drive under `KidsDiary/YYYY/MM`  
    - Confirm reply messages appear in LINE chat

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow assumes valid tokens for LINE Messaging API and Google Drive OAuth2 credentials are set up properly.    | Required for HTTP Request and Google Drive nodes respectively                                     |
| OpenAI Vision usage requires API access to GPT-4o with Vision capabilities; ensure API subscription supports this.  | OpenAI documentation for GPT-4o Vision: https://platform.openai.com/docs/models/gpt-4o             |
| Folder naming convention uses underscores and date formatting to keep files organized for easy retrieval later.     | Naming pattern: `KidsDiary_YYYY_MM` and diary files timestamped `yyyy-MM-dd_HHmmss_diary.txt`      |
| The workflow does not currently handle multiple events in a single webhook call; only processes the first event.     | Enhancement could include looping over all events in the webhook payload                           |
| Consider verifying LINE webhook signature for security to reject unauthorized requests.                             | LINE docs: https://developers.line.biz/en/docs/messaging-api/verify-signature/                     |
| The original photo upload node uses the diary text as filename; this may cause issues if diary text contains invalid filename characters. | Adding sanitization or alternative naming recommended                                              |
| Google Drive "Upload Photo to Drive" node uses root folder ID; consider changing to the created folder for consistency. | Possible bug or oversight in folder ID assignment                                                 |
| Sticky notes provide step-by-step documentation visible in the n8n editor to assist users in understanding workflow. | Useful for maintenance and onboarding new users                                                   |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.