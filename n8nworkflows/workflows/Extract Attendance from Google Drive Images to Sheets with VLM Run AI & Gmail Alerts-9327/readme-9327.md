Extract Attendance from Google Drive Images to Sheets with VLM Run AI & Gmail Alerts

https://n8nworkflows.xyz/workflows/extract-attendance-from-google-drive-images-to-sheets-with-vlm-run-ai---gmail-alerts-9327


# Extract Attendance from Google Drive Images to Sheets with VLM Run AI & Gmail Alerts

### 1. Workflow Overview

This workflow automates attendance extraction from images stored in a Google Drive folder, processes the image using AI to recognize user names, appends the attendance data to a Google Sheet, and sends an email alert with the attendance summary. It is ideal for scenarios like workshop sign-ins, classroom roll calls, or whiteboard standups.

The workflow consists of the following logical blocks:

- **1.1 Google Drive Monitoring & Download**: Watches for new attendance image uploads in a specified Drive folder and downloads the images for processing.
- **1.2 AI Processing with VLM Run**: Uses the VLM Run AI agent to analyze the downloaded image and extract attendance data in a structured JSON format.
- **1.3 Webhook Receiver**: Receives the processed attendance JSON asynchronously from the AI agent’s callback.
- **1.4 Append Attendance to Google Sheets**: Appends the attendance data as a new row in a Google Sheet with a specific column structure.
- **1.5 Email Notification via Gmail**: Sends a formatted email containing the attendance list to a chosen recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Google Drive Monitoring & Download

- **Overview:**  
  This block monitors a dedicated Google Drive folder for new image files representing attendance lists, then downloads the new files in binary format for AI processing.

- **Nodes Involved:**  
  - Monitor List Uploads (Google Drive Trigger)  
  - Download List (Google Drive)

- **Node Details:**  

  **Monitor List Uploads**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches a specific Google Drive folder for new files created every minute.  
  - *Configuration:*  
    - Event: `fileCreated`  
    - Poll interval: every minute  
    - Folder to watch: Folder ID `1S6baavqJn98MjUlbB6KtmARCWuWEekIZ` (attendance images folder)  
  - *Connections:* Output → Download List  
  - *Potential Failures:*  
    - Authentication failures with Google Drive OAuth2  
    - Folder permission errors  
    - API rate limits or quota exceeded  

  **Download List**  
  - *Type:* Google Drive  
  - *Role:* Downloads the file by ID received from the trigger node as binary data under property `data`.  
  - *Configuration:*  
    - File Id: taken dynamically from trigger output `$json.id`  
    - Operation: `download`  
    - Binary property name: `data`  
  - *Connections:* Output → VLM Run for Extraction  
  - *Potential Failures:*  
    - File not found or deleted before download  
    - Authentication or permission issues  
    - Network timeouts  

---

#### 1.2 AI Processing with VLM Run

- **Overview:**  
  Executes an AI agent from VLM Run to analyze the downloaded attendance image and extract a JSON-formatted attendance list with date, total count, and user names. The AI agent posts the results asynchronously to a webhook.

- **Nodes Involved:**  
  - VLM Run for Extraction

- **Node Details:**  

  **VLM Run for Extraction**  
  - *Type:* VLM Run node (`@vlm-run/n8n-nodes-vlmrun.vlmRun`)  
  - *Role:* Sends the image binary to the VLM Run AI agent by triggering the `executeAgent` operation. The agent uses a prompt to parse the image and return attendance data in a strict JSON format.  
  - *Configuration:*  
    - Operation: `executeAgent`  
    - Agent prompt includes instructions to:  
      - Analyze the image of user list names  
      - Output JSON with structure:  
        ```json
        {
          "majorDimension": "ROWS",
          "values": [
            ["val1", "val2", "name1", "name2", ...]
          ]
        }
        ```  
      - `val1` = current date, `val2` = total user count, rest = user names  
    - Agent callback URL: `https://playground.attensys.ai/webhook/check-attendance` (n8n webhook endpoint)  
  - *Credentials:* VLM Run API account  
  - *Connections:* None (async)  
  - *Potential Failures:*  
    - API authentication errors  
    - Timeout or connectivity issues with VLM Run  
    - Improper prompt leading to malformed JSON  
    - Callback URL unreachable or rejected  

---

#### 1.3 Webhook Receiver

- **Overview:**  
  Receives the processed attendance JSON from the VLM agent’s callback asynchronously, acting as the entry point for appending the data to Google Sheets.

- **Nodes Involved:**  
  - Receives the list (Webhook)

- **Node Details:**  

  **Receives the list**  
  - *Type:* Webhook (HTTP POST)  
  - *Role:* Public webhook endpoint named `check-attendance` to receive JSON attendance data from VLM Run agent.  
  - *Configuration:*  
    - HTTP method: POST  
    - Path: `/check-attendance`  
  - *Connections:* Output → Append New Attendance to Sheet  
  - *Potential Failures:*  
    - Unreachable webhook (e.g., localhost URL in callback)  
    - Invalid or malformed JSON payload  
    - Security concerns if webhook is public without auth (should be protected)  

---

#### 1.4 Append Attendance to Google Sheets

- **Overview:**  
  Appends the received attendance data as a new row in a Google Sheets spreadsheet, inserting the date, total count, and user names across columns.

- **Nodes Involved:**  
  - Append New Attendance to Sheet (HTTP Request)

- **Node Details:**  

  **Append New Attendance to Sheet**  
  - *Type:* HTTP Request  
  - *Role:* Calls Google Sheets API `values:append` endpoint to insert data into `Sheet1!A:Z`.  
  - *Configuration:*  
    - URL:  
      `https://sheets.googleapis.com/v4/spreadsheets/1lg0aJKvd7E2pbhumHNjcgxUfEQKvlBs9h1zZbhSeqas/values/Sheet1!A:Z:append`  
    - Method: POST  
    - Query parameters:  
      - `valueInputOption=RAW`  
      - `insertDataOption=INSERT_ROWS`  
      - `includeValuesInResponse=true`  
    - Headers: `Content-Type: application/json`  
    - Body: JSON from webhook payload (`{{$json.body.response}}`) expected to follow the VLM Run JSON format  
    - Authentication: Google Sheets OAuth2  
  - *Connections:* Output → Send a message  
  - *Potential Failures:*  
    - Google Sheets API quota limits  
    - Authentication errors  
    - Malformed JSON body causing API rejection  
    - Spreadsheet ID or range invalid or permissions missing  

---

#### 1.5 Email Notification via Gmail

- **Overview:**  
  Sends an email summarizing the attendance list, including the date, total count, and numbered list of attendees extracted from the Google Sheets append response.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**  

  **Send a message**  
  - *Type:* Gmail node  
  - *Role:* Sends an email with subject "Attendance List" to a specified recipient. The message body is dynamically composed from the appended sheet data.  
  - *Configuration:*  
    - To: `mehediahamed@iut-dhaka.edu` (recipient email)  
    - Subject: `Attendance List`  
    - Message: Dynamically constructed HTML string by:  
      - Extracting updated values from the Google Sheets append response  
      - Formatting date (YYYY-MM-DD → DD-MM-YYYY)  
      - Listing total count and enumerated attendee names with line breaks `<br>`  
    - Uses Gmail OAuth2 credentials  
  - *Connections:* None (end node)  
  - *Potential Failures:*  
    - Gmail OAuth2 authentication failures  
    - Invalid recipient address  
    - API rate limits or send quota exceeded  
    - Errors in expression could lead to empty or malformed email body  

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                    | Input Node(s)         | Output Node(s)             | Sticky Note                                                                                                                         |
|---------------------------|----------------------------------|----------------------------------|-----------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Monitor List Uploads       | Google Drive Trigger             | Detect new attendance images     | —                     | Download List              | Monitors Google Drive folder for new receipt uploads and triggers processing automatically.                                        |
| Download List             | Google Drive                    | Download newly uploaded files    | Monitor List Uploads   | VLM Run for Extraction     | Downloads receipt files from Google Drive for AI processing.                                                                        |
| VLM Run for Extraction     | VLM Run AI node                 | Extract attendance from image    | Download List         | —                          | Execute Agent processes the downloaded image using AI to extract attendance data.                                                  |
| Receives the list          | Webhook                        | Receive AI processed attendance  | —                     | Append New Attendance to Sheet | Webhook receives ready-to-append Sheets payload asynchronously from AI agent.                                                      |
| Append New Attendance to Sheet | HTTP Request (Google Sheets API) | Append attendance data to Sheet | Receives the list      | Send a message             | Calls Google Sheets API to append attendance data as a new row, formatting columns as date, total, then attendee names.            |
| Send a message             | Gmail                          | Email attendance summary         | Append New Attendance to Sheet | —                          | Sends formatted attendance email to recipient with date, total count, and list of attendees.                                       |
| Sticky Note                | Sticky Note                    | Informational                    | —                     | —                          | ## 🟣 Append to Sheets + Send Email — Explains the append and email process in detail with sample input and output.                |
| Sticky Note1               | Sticky Note                    | Informational                    | —                     | —                          | ## 🟡 VLM Run: Extraction Logic — Details AI prompt and output format for attendance extraction.                                    |
| Sticky Note2               | Sticky Note                    | Informational                    | —                     | —                          | ## 🟢 Google Drive: Monitor & Download — Describes trigger and download nodes watching Drive folder for new images.                 |
| Sticky Note3               | Sticky Note                    | Informational                    | —                     | —                          | ## 📝 Webhook: Set Public Callback URL — Reminder to use a public reachable webhook URL for AI callback.                            |
| Sticky Note4               | Sticky Note                    | Informational                    | —                     | —                          | ## 🧾 Attendance Extraction Pipeline — Overall workflow description, use cases, and required credentials.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Name: `Monitor List Uploads`  
   - Event: `fileCreated`  
   - Poll interval: every minute  
   - Folder to watch: Specify your attendance images folder ID  
   - Credentials: Google Drive OAuth2 setup with access to the folder  
   - Connect output to `Download List`

2. **Create Google Drive node**  
   - Type: Google Drive  
   - Name: `Download List`  
   - Operation: `download`  
   - File ID: `={{ $json.id }}` (from Trigger output)  
   - Binary property: `data`  
   - Credentials: Same Google Drive OAuth2 as trigger  
   - Connect output to `VLM Run for Extraction`

3. **Create VLM Run node**  
   - Type: `@vlm-run/n8n-nodes-vlmrun.vlmRun`  
   - Name: `VLM Run for Extraction`  
   - Operation: `executeAgent`  
   - Agent Prompt:  
     ```
     Analyze the image of user list names and show them in the following json format, make sure to follow this exact structure, where val1 will be current date, val2 will be total user count and rest of the values will be the user names:
     { "majorDimension": "ROWS", "values": [ ["val1", "val2"] ] }
     ```  
   - Agent callback URL: Provide your n8n webhook public URL `https://<your-n8n-domain>/webhook/check-attendance`  
   - Credentials: VLM Run API credentials  
   - No direct output connection (async callback)

4. **Create Webhook node**  
   - Type: Webhook  
   - Name: `Receives the list`  
   - HTTP method: POST  
   - Path: `check-attendance`  
   - No authentication configured (optional: add security)  
   - Connect output to `Append New Attendance to Sheet`

5. **Create HTTP Request node for Google Sheets**  
   - Type: HTTP Request  
   - Name: `Append New Attendance to Sheet`  
   - Method: POST  
   - URL:  
     `https://sheets.googleapis.com/v4/spreadsheets/<your-spreadsheet-id>/values/Sheet1!A:Z:append`  
     Replace `<your-spreadsheet-id>` with your Sheet’s ID  
   - Query Parameters:  
     - `valueInputOption=RAW`  
     - `insertDataOption=INSERT_ROWS`  
     - `includeValuesInResponse=true`  
   - Headers: `Content-Type: application/json`  
   - Body: `={{ $json.body.response }}` (payload from webhook)  
   - Authentication: Google Sheets OAuth2 credentials  
   - Connect output to `Send a message`

6. **Create Gmail node**  
   - Type: Gmail  
   - Name: `Send a message`  
   - To: Your recipient email (e.g., `mehediahamed@iut-dhaka.edu`)  
   - Subject: `Attendance List`  
   - Message: Use expression to format attendance data:  
     ```javascript
     (() => {
       const raw = $json.updates?.updatedData?.values;
       const parts = Array.isArray(raw)
         ? (Array.isArray(raw[0]) ? raw[0] : raw).map(String)
         : String(raw || "").split(",").map(s => s.trim());
       if (!parts.length) return "No data received";
       const date = parts[0];
       const total = parts[1];
       const names = parts.slice(2).filter(Boolean);
       const prettyDate = /^\d{4}-\d{2}-\d{2}$/.test(date)
         ? date.replace(/^(\d{4})-(\d{2})-(\d{2})$/, "$3-$2-$1")
         : date;
       const list = names.map((n, i) => `${i + 1}. ${n}`).join("<br>");
       return `Today's Attendance List (${prettyDate})<br>Total: ${total}<br>${list}`;
     })()
     ```  
   - Credentials: Gmail OAuth2  
   - No output

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Use a public reachable n8n webhook URL in VLM Run Callback URL field; localhost URLs are unreachable | Sticky Note near Webhook node (`Receives the list`)                                              |
| The workflow is suitable for whiteboard standups, workshops, classrooms for attendance automation    | Sticky Note4 overview                                                                             |
| Requires VLM Run API credentials with Execute Agent access, Google Drive and Sheets OAuth2 credentials | Sticky Note4 and throughout credential configurations                                           |
| Google Sheets appends data with date in column A, total count in column B, then attendee names       | Sticky Note explaining append format and email formatting                                        |
| Example attendance JSON format:                                                                       | `{ "majorDimension": "ROWS", "values": [["2025-10-03","6","Camila Torres Rivera","Mellissa"]] }` |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected elements. All data processed is legal and public.