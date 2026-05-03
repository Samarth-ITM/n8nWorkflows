Capture Website Screenshots via Google Sheets to Google Drive with CustomJS

https://n8nworkflows.xyz/workflows/capture-website-screenshots-via-google-sheets-to-google-drive-with-customjs-3332


# Capture Website Screenshots via Google Sheets to Google Drive with CustomJS

### 1. Workflow Overview

This workflow automates the process of capturing website screenshots based on URLs listed in a Google Sheet and storing those screenshots in a designated Google Drive folder. It is designed for users who want to monitor websites visually, create visual archives, or automate content curation workflows.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Monitoring Google Sheets for new rows containing website URLs and titles.
- **1.2 Screenshot Capture:** Using the CustomJS PDF Toolkit node to take screenshots of the specified websites.
- **1.3 Storage:** Uploading the captured screenshots to a specified folder in Google Drive with appropriate filenames.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block continuously monitors a specific Google Sheet for newly added rows. Each new row is expected to contain a website URL and a title. When a new row is detected, the data is extracted and passed downstream for processing.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  - **Google Sheets Trigger**  
    - **Type:** Trigger node for Google Sheets  
    - **Role:** Watches for new rows added to a specified Google Sheet and sheet tab.  
    - **Configuration:**  
      - Event: `rowAdded` (triggers on new rows)  
      - Polling interval: every 1 minute  
      - Document ID: Points to the Google Sheet with ID `1SP8Y-qffC96ZV3ueVUYWP5pjqtaycaM7Kbv5L-ztw5g`  
      - Sheet Name: Uses the sheet with `gid=0` (usually the first sheet)  
    - **Key Expressions:**  
      - Extracts `Url` and `Title` fields from each new row (case-sensitive column headers in the sheet)  
    - **Input/Output:**  
      - Input: None (trigger node)  
      - Output: JSON object containing the new row data, including `Url` and `Title`  
    - **Version Requirements:** Compatible with n8n Google Sheets Trigger node version 1  
    - **Edge Cases / Failures:**  
      - Authentication errors if Google credentials are invalid or expired  
      - Polling delays or missed triggers if Google Sheets API quota is exceeded  
      - Failure if the sheet or document ID is incorrect or access is revoked  
    - **Sub-workflow:** None

#### 2.2 Screenshot Capture

- **Overview:**  
  This block takes the URL from the Google Sheets Trigger and uses the CustomJS PDF Toolkit node to capture a screenshot of the website.

- **Nodes Involved:**  
  - Take a screenshot of a website

- **Node Details:**

  - **Take a screenshot of a website**  
    - **Type:** CustomJS PDF Toolkit node (`websiteScreenshot`)  
    - **Role:** Captures a screenshot image of the website at the given URL  
    - **Configuration:**  
      - URL Input: Dynamically set to the `Url` field from the incoming JSON (`={{ $json.Url }}`)  
      - Requires a valid CustomJS API key configured in n8n credentials  
    - **Input/Output:**  
      - Input: JSON containing `Url` and `Title` from Google Sheets Trigger  
      - Output: Binary data representing the screenshot image (PNG format) along with metadata  
    - **Version Requirements:** Requires installation of the community node `@custom-js/n8n-nodes-pdf-toolkit` on a self-hosted n8n instance  
    - **Edge Cases / Failures:**  
      - API key invalid or missing leading to authentication failure  
      - URL unreachable or timing out causing screenshot failure  
      - Websites with dynamic content or requiring authentication may not render correctly  
      - Rate limits or service downtime on CustomJS API  
    - **Sub-workflow:** None

#### 2.3 Storage

- **Overview:**  
  This block uploads the screenshot image produced by the previous node to a specified folder in Google Drive, naming the file according to the website title.

- **Nodes Involved:**  
  - Store Screenshots

- **Node Details:**

  - **Store Screenshots**  
    - **Type:** Google Drive node (upload file)  
    - **Role:** Saves the screenshot PNG file to Google Drive in a specified folder  
    - **Configuration:**  
      - File Name: Set dynamically as `{{ $json.Title }}.png` to use the website title from the sheet  
      - Drive ID: Set to `"My Drive"` (default user drive)  
      - Folder ID: Set to `1oFbmzgG2fsRix45r5JtowYjAdwskJ0P6` (target folder for screenshots)  
      - Uploads binary data received from the screenshot node  
    - **Input/Output:**  
      - Input: Binary screenshot image and metadata from the screenshot node  
      - Output: Metadata about the uploaded file (file ID, URL, etc.)  
    - **Version Requirements:** Compatible with Google Drive node version 3  
    - **Edge Cases / Failures:**  
      - Authentication errors if Google Drive credentials are invalid or expired  
      - Folder ID invalid or access revoked causing upload failure  
      - File naming conflicts if multiple files have the same title (overwrites or errors depending on settings)  
      - Upload size limits or API quota exceeded  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                          |
|-----------------------------|----------------------------------|-------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Google Sheets Trigger        | n8n-nodes-base.googleSheetsTrigger | Monitors Google Sheet for new rows | None                    | Take a screenshot of a website | Ensure Google Sheets has columns `Url` and `Title`. Requires Google Sheets credentials in n8n.     |
| Take a screenshot of a website | @custom-js/n8n-nodes-pdf-toolkit.websiteScreenshot | Captures website screenshot via CustomJS API | Google Sheets Trigger   | Store Screenshots         | Requires CustomJS API key credential. Community node only for self-hosted n8n.                      |
| Store Screenshots            | n8n-nodes-base.googleDrive       | Uploads screenshot PNG to Google Drive | Take a screenshot of a website | None                     | Set Google Drive folder ID to store screenshots. Requires Google Drive credentials in n8n.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Add a new node of type `Google Sheets Trigger`.  
   - Set event to `rowAdded`.  
   - Configure polling interval to every 1 minute.  
   - Set Document ID to your Google Sheet ID containing URLs and Titles.  
   - Set Sheet Name to the appropriate sheet tab (e.g., `gid=0`).  
   - Ensure your Google Sheets credential is configured and authorized in n8n.

2. **Create CustomJS Website Screenshot Node**  
   - Add a new node of type `@custom-js/n8n-nodes-pdf-toolkit.websiteScreenshot`.  
   - In the URL input parameter, set the expression to `{{$json.Url}}` to dynamically use the URL from the trigger.  
   - Configure the CustomJS API key credential in n8n (requires signing up at https://www.customjs.space and adding the API key).  
   - This node requires installation of the community node `@custom-js/n8n-nodes-pdf-toolkit` on your self-hosted n8n instance.

3. **Create Google Drive Upload Node**  
   - Add a new node of type `Google Drive`.  
   - Set the operation to upload a file.  
   - For the file name, use the expression `{{$json.Title}}.png` to name the file after the website title.  
   - Set Drive ID to `"My Drive"` or your target drive.  
   - Set Folder ID to the Google Drive folder ID where screenshots should be stored.  
   - Ensure your Google Drive credentials are configured and authorized in n8n.

4. **Connect Nodes**  
   - Connect the output of the Google Sheets Trigger node to the input of the CustomJS Website Screenshot node.  
   - Connect the output of the Website Screenshot node to the input of the Google Drive Upload node.

5. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by adding a new row with `Url` and `Title` columns in your Google Sheet.  
   - Confirm that a screenshot is captured and uploaded to Google Drive with the correct filename.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Community nodes like CustomJS PDF Toolkit require a self-hosted n8n instance to install.      | https://docs.n8n.io/integrations/community-nodes/                                               |
| CustomJS API key signup and management available at https://www.customjs.space                | https://www.customjs.space                                                                       |
| Google Sheets must have columns named exactly `Url` and `Title` for this workflow to function | Naming conventions are case-sensitive in Google Sheets                                          |
| Google Drive folder ID can be found in the folder URL after `/folders/`                        | Example: https://drive.google.com/drive/folders/1oFbmzgG2fsRix45r5JtowYjAdwskJ0P6                 |
| This workflow is ideal for website monitoring, visual archiving, and content curation automation |                                                                                                 |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and automation agents to reproduce, modify, and troubleshoot it effectively.