Capture URL Screenshots from Google Sheets with ScreenshotOne & Save to Drive with Gmail Alerts

https://n8nworkflows.xyz/workflows/capture-url-screenshots-from-google-sheets-with-screenshotone---save-to-drive-with-gmail-alerts-3321


# Capture URL Screenshots from Google Sheets with ScreenshotOne & Save to Drive with Gmail Alerts

### 1. Workflow Overview

This workflow automates the process of capturing screenshots from URLs listed in a Google Sheets spreadsheet and managing the resulting images in Google Drive, with email notifications sent via Gmail. It is designed to eliminate the manual effort of visiting URLs, taking screenshots, saving files, and notifying stakeholders.

**Target Use Cases:**  
- Digital marketers monitoring campaign landing pages  
- Web developers/designers collecting website visuals  
- QA teams documenting web page states  
- SEO specialists tracking visual changes  
- Content managers overseeing content appearance  

**Logical Blocks:**  
- **1.1 Folder Monitoring (Trigger):** Watches a specific Google Drive folder for new spreadsheet files.  
- **1.2 URL Extraction:** Reads the "Url" column from the newly added spreadsheet.  
- **1.3 Screenshot Capture:** Sends each URL to ScreenshotOne API to capture screenshots.  
- **1.4 Saving & Notification:** Uploads screenshots to the same Google Drive folder and sends an email alert with the folder link.

---

### 2. Block-by-Block Analysis

#### 1.1 Folder Monitoring (Trigger)

- **Overview:**  
  This block initiates the workflow by monitoring a designated Google Drive folder for newly created spreadsheet files. When a new file is detected, it triggers the workflow.

- **Nodes Involved:**  
  - `spreadsheets file with urls created` (Google Drive Trigger)  
  - `Sticky Note` (commentary)

- **Node Details:**  

  - **Node:** `spreadsheets file with urls created`  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for new files of type spreadsheet (`application/vnd.google-apps.spreadsheet`).  
    - Configuration:  
      - Event: `fileCreated`  
      - File Type Filter: Spreadsheet only  
      - Polling Interval: Every minute  
      - Folder to Watch: User must specify the Google Drive folder ID (`--YOUR GOOGLE DRIVE FOLDER ID TO LISTEN--`)  
      - Credentials: Google Drive OAuth2 (authenticated user)  
    - Inputs: None (trigger node)  
    - Outputs: Emits metadata of the newly created spreadsheet file, including its ID and parent folder ID.  
    - Edge Cases:  
      - Permissions errors if folder access is restricted  
      - Delay due to polling interval  
      - Multiple files created simultaneously may trigger multiple workflow runs  
    - Sticky Note: "## This node initiates the workflow by monitoring the designated folder"

---

#### 1.2 URL Extraction

- **Overview:**  
  Extracts URLs from the "Url" column of the newly created spreadsheet file detected in the previous block.

- **Nodes Involved:**  
  - `Get Urls` (Google Sheets)  
  - `Sticky Note1` (commentary)

- **Node Details:**  

  - **Node:** `Get Urls`  
    - Type: Google Sheets  
    - Role: Reads data from the spreadsheet file identified by the trigger node.  
    - Configuration:  
      - Document ID: Dynamically set from the trigger node output (`={{ $json.id }}`)  
      - Sheet Name: `Sheet1` (default sheet)  
      - Reads all rows, focusing on the column named "Url"  
    - Inputs: Receives spreadsheet metadata from the trigger node  
    - Outputs: Emits each row as an item with the "Url" field extracted  
    - Edge Cases:  
      - Missing or misspelled "Url" column leads to empty or invalid data  
      - Empty rows or invalid URLs may cause downstream errors  
      - Permissions errors if spreadsheet access is restricted  
    - Sticky Note: "## Ensure your spreadsheet’s ‘Url.’ column contains valid URLs"

---

#### 1.3 Screenshot Capture

- **Overview:**  
  Sends each extracted URL to the ScreenshotOne API to capture a screenshot image.

- **Nodes Involved:**  
  - `Get screenshots` (HTTP Request)  
  - `Sticky Note2` (commentary)

- **Node Details:**  

  - **Node:** `Get screenshots`  
    - Type: HTTP Request  
    - Role: Calls ScreenshotOne API to capture screenshots for each URL.  
    - Configuration:  
      - HTTP Method: GET (default for this node)  
      - URL: `https://api.screenshotone.com/take?` with query parameters  
      - Query Parameters:  
        - `url`: dynamically set from the current item’s `Url` field (`={{ $json.Url }}`)  
        - `block_cookie_banners`: set to `true` to block cookie banners in screenshots  
      - Headers:  
        - `X-Access-Key`: User must replace placeholder with their ScreenshotOne access key (`--YOUR SCREENSHOTONE ACCESS KEY--`)  
      - Retry: Enabled with up to 3 attempts on failure  
      - Execute Once: False (runs for each URL)  
    - Inputs: Receives URLs from the previous node  
    - Outputs: Returns screenshot data (binary image data expected in response)  
    - Edge Cases:  
      - Invalid or unreachable URLs cause API errors or empty screenshots  
      - API rate limits or quota exceeded errors  
      - Incorrect or missing access key leads to authentication failure  
      - Network timeouts or HTTP errors  
    - Sticky Note: "## Replace the placeholder with your actual ScreenshotOne  [Access key](https://dash.screenshotone.com/access)"

---

#### 1.4 Saving & Notification

- **Overview:**  
  Saves the captured screenshots into the same Google Drive folder where the spreadsheet was uploaded, then sends an email notification with a link to that folder.

- **Nodes Involved:**  
  - `Upload images to the same folder` (Google Drive)  
  - `Send email with folder link` (Gmail)  
  - `Sticky Note3` (commentary)

- **Node Details:**  

  - **Node:** `Upload images to the same folder`  
    - Type: Google Drive  
    - Role: Uploads screenshot images to Google Drive folder.  
    - Configuration:  
      - File Name: Dynamically generated as `screenshot_{{ URL }}` (e.g., `screenshot_https://example.com`)  
      - Drive: `My Drive` (default Google Drive)  
      - Folder ID: Dynamically set to the parent folder ID of the spreadsheet (`={{ $('spreadsheets file with urls created').item.json.parents[0]}}`)  
      - Input Data Field: `data` (expects binary image data from ScreenshotOne response)  
    - Inputs: Receives screenshot binary data from the HTTP Request node  
    - Outputs: Metadata of the uploaded file(s)  
    - Edge Cases:  
      - Permissions errors if folder is not writable  
      - Invalid file names due to URL characters (may require sanitization)  
      - Upload failures due to network or quota issues  

  - **Node:** `Send email with folder link`  
    - Type: Gmail  
    - Role: Sends an email notification to a specified recipient with a link to the Google Drive folder containing screenshots.  
    - Configuration:  
      - Recipient: User must specify email address (`--YOUR EMAIL ADDRESS--`)  
      - Subject: "Screenshots are ready"  
      - Message Body: Includes a text message and a link to the folder using the folder ID (`https://drive.google.com/drive/folders/{{ $json.parents }}?usp=drive_link`)  
      - Email Type: Plain text  
      - Execute Once: True (only one email sent per workflow run)  
    - Inputs: Receives folder metadata from the upload node  
    - Outputs: Email send confirmation  
    - Edge Cases:  
      - Authentication errors if Gmail credentials are invalid or expired  
      - Email delivery failures or spam filtering  
      - Incorrect folder link if folder ID is missing or malformed  
    - Sticky Note: "## Review your email settings to ensure notifications are sent correctly"

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                     |
|-------------------------------|---------------------|----------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| spreadsheets file with urls created | Google Drive Trigger | Monitors folder for new spreadsheet files | None                             | Get Urls                        | ## This node initiates the workflow by monitoring the designated folder                        |
| Get Urls                      | Google Sheets       | Extracts URLs from spreadsheet          | spreadsheets file with urls created | Get screenshots                 | ## Ensure your spreadsheet’s ‘Url.’ column contains valid URLs                                |
| Get screenshots               | HTTP Request        | Captures screenshots via ScreenshotOne API | Get Urls                        | Upload images to the same folder | ## Replace the placeholder with your actual ScreenshotOne  [Access key](https://dash.screenshotone.com/access) |
| Upload images to the same folder | Google Drive       | Uploads screenshots to Google Drive folder | Get screenshots                 | Send email with folder link      |                                                                                                |
| Send email with folder link   | Gmail               | Sends notification email with folder link | Upload images to the same folder | None                           | ## Review your email settings to ensure notifications are sent correctly                      |
| Sticky Note                  | Sticky Note         | Comment on folder monitoring             | None                             | None                           | ## This node initiates the workflow by monitoring the designated folder                        |
| Sticky Note1                 | Sticky Note         | Comment on URL column requirement        | None                             | None                           | ## Ensure your spreadsheet’s ‘Url.’ column contains valid URLs                                |
| Sticky Note2                 | Sticky Note         | Comment on ScreenshotOne access key      | None                             | None                           | ## Replace the placeholder with your actual ScreenshotOne  [Access key](https://dash.screenshotone.com/access) |
| Sticky Note3                 | Sticky Note         | Comment on email notification settings   | None                             | None                           | ## Review your email settings to ensure notifications are sent correctly                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure:  
     - Event: `fileCreated`  
     - File Type: `application/vnd.google-apps.spreadsheet`  
     - Polling Interval: Every minute  
     - Folder to Watch: Specify your Google Drive folder ID where spreadsheets will be uploaded  
   - Authenticate with Google Drive OAuth2 credentials  
   - This node triggers the workflow when a new spreadsheet is added.

2. **Create Google Sheets Node ("Get Urls")**  
   - Type: Google Sheets  
   - Configure:  
     - Document ID: Set to `={{ $json.id }}` (from trigger node output)  
     - Sheet Name: `Sheet1` (or your target sheet)  
   - This node reads the spreadsheet rows, expecting a column named "Url" containing URLs.

3. **Create HTTP Request Node ("Get screenshots")**  
   - Type: HTTP Request  
   - Configure:  
     - URL: `https://api.screenshotone.com/take?`  
     - Query Parameters:  
       - `url`: `={{ $json.Url }}` (dynamic URL from previous node)  
       - `block_cookie_banners`: `true`  
     - Headers:  
       - `X-Access-Key`: Replace with your ScreenshotOne access key  
     - Retry: Enable retry on failure, max 3 attempts  
     - Execute Once: Disabled (runs for each URL)  
   - This node calls ScreenshotOne API to capture screenshots.

4. **Create Google Drive Node ("Upload images to the same folder")**  
   - Type: Google Drive  
   - Configure:  
     - File Name: `=screenshot_{{ $('Get screenshots').item.json.Url }}` (dynamic naming)  
     - Drive: `My Drive`  
     - Folder ID: `={{ $('spreadsheets file with urls created').item.json.parents[0]}}` (parent folder of the spreadsheet)  
     - Input Data Field Name: `data` (binary data from HTTP Request)  
   - This node uploads screenshots to the same folder as the spreadsheet.

5. **Create Gmail Node ("Send email with folder link")**  
   - Type: Gmail  
   - Configure:  
     - Send To: Your email address (e.g., `your.email@example.com`)  
     - Subject: "Screenshots are ready"  
     - Message:  
       ```
       Your screenshots are ready!
       You can find them in the following folder:
       https://drive.google.com/drive/folders/{{ $json.parents }}?usp=drive_link
       ```  
     - Email Type: Plain text  
     - Execute Once: Enabled (send only one email per workflow run)  
   - Authenticate with Gmail OAuth2 credentials.

6. **Connect Nodes in Order:**  
   - `spreadsheets file with urls created` → `Get Urls`  
   - `Get Urls` → `Get screenshots`  
   - `Get screenshots` → `Upload images to the same folder`  
   - `Upload images to the same folder` → `Send email with folder link`

7. **Add Sticky Notes (Optional):**  
   - Add notes near nodes to document purpose and reminders, such as:  
     - Folder monitoring explanation  
     - Reminder to have valid URLs in the spreadsheet  
     - Replace ScreenshotOne access key placeholder  
     - Review email settings

8. **Credentials Setup:**  
   - Google Drive OAuth2 credentials with permissions to read/write in the target folder  
   - Google Sheets OAuth2 credentials with read access to spreadsheets  
   - Gmail OAuth2 credentials with send email permission  
   - ScreenshotOne access key (replace placeholder in HTTP Request node)

9. **Test Workflow:**  
   - Upload a spreadsheet with a "Url" column to the monitored Google Drive folder  
   - Confirm screenshots are captured and saved in the same folder  
   - Confirm email notification is received with correct folder link

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| ScreenshotOne Access Key is required; obtain it from your ScreenshotOne dashboard.                  | https://dash.screenshotone.com/access                                                          |
| ScreenshotOne API documentation for advanced options like viewport size or device emulation.       | https://dash.screenshotone.com/playground                                                      |
| Customize the Google Drive folder ID in the trigger node to monitor your preferred folder.         | Workflow configuration step                                                                     |
| Customize the recipient email, subject, and message body in the Gmail node to suit your needs.     | Workflow configuration step                                                                     |
| This workflow reduces manual screenshot capture efforts, saving time for marketers, developers, QA, SEO, and content teams. | Workflow purpose                                                                                |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by users and AI agents alike.