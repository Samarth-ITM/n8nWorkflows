Automated Google Sheet to CSV Conversion via Slack Messages

https://n8nworkflows.xyz/workflows/automated-google-sheet-to-csv-conversion-via-slack-messages-7771


# Automated Google Sheet to CSV Conversion via Slack Messages

### 1. Workflow Overview

This workflow automates the conversion of a Google Sheet to a CSV file triggered by Slack messages in a specific Slack channel. When a user sends a message containing a Google Sheets URL or mentions the Slack app, the workflow extracts the spreadsheet ID, reads the sheet data, converts it to CSV format, and uploads the resulting CSV file back to the same Slack channel.

Logical blocks included:  
- **1.1 Slack Message Reception & Trigger**: Listens for Slack messages or app mentions in a defined channel.  
- **1.2 Sheet ID Extraction**: Parses the Google Sheets URL from the Slack message text to extract the spreadsheet ID.  
- **1.3 Google Sheets Data Reading**: Uses the extracted spreadsheet ID to read the sheet content.  
- **1.4 CSV Conversion**: Converts the sheet data into a CSV file.  
- **1.5 Slack File Upload**: Uploads the generated CSV file back to the designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Slack Message Reception & Trigger

- **Overview:**  
  This block listens for any event or app mention in a specific Slack channel to initiate the workflow.

- **Nodes Involved:**  
  - Slack Trigger

- **Node Details:**

  - **Slack Trigger**  
    - Type & Role: Slack event trigger node that waits for specified Slack events.  
    - Configuration:  
      - Trigger on any event or specifically on app mentions.  
      - Limited to channel ID `C099B0PKEF2` (the monitored Slack channel).  
    - Key expressions/variables: Uses the Slack API credentials linked to “Slack account 4”.  
    - Input/Output: This is a trigger node; it outputs the Slack event JSON to the next node.  
    - Version: v1.  
    - Potential failure types:  
      - Authentication errors if Slack credentials are invalid or expired.  
      - Network or webhook delivery issues.  
      - Permissions issues if the app is not authorized in the channel.  
    - Sub-workflow: None.

#### 1.2 Sheet ID Extraction

- **Overview:**  
  Parses the Slack message text to find a Google Sheets URL and extracts the sheet ID parameter.

- **Nodes Involved:**  
  - Extract Sheet ID (Function node)

- **Node Details:**

  - **Extract Sheet ID**  
    - Type & Role: Function node that runs custom JavaScript to parse text.  
    - Configuration:  
      - Uses a regex to extract the sheet ID from the Slack message text field (`$json.text`).  
      - Returns an object with `sheetId` and the original `channel` for downstream use.  
    - Key expressions/variables:  
      - Regex pattern: `/https:\/\/docs\.google\.com\/spreadsheets\/d\/([a-zA-Z0-9-_]+)/`  
      - Outputs sheetId extracted from the URL.  
    - Input: Slack event JSON containing message text.  
    - Output: JSON with extracted `sheetId` and `channel`.  
    - Version: v1.  
    - Potential failure types:  
      - No valid URL present → returns an empty array, halting downstream nodes.  
      - Regex failures or unexpected message formats.  
    - Sub-workflow: None.

#### 1.3 Google Sheets Data Reading

- **Overview:**  
  Reads the content of the specified Google Sheet tab using the extracted sheet ID.

- **Nodes Involved:**  
  - Read Google Sheet

- **Node Details:**

  - **Read Google Sheet**  
    - Type & Role: Google Sheets node that fetches sheet rows.  
    - Configuration:  
      - `documentId`: dynamically set using `={{ $json.sheetId }}` from the previous node output.  
      - `sheetName`: statically set to the sheet with GID `253150622` (Sheet4).  
    - Credentials: Uses Google Sheets OAuth2 credentials named “Google Sheets account”.  
    - Input: JSON containing `sheetId`.  
    - Output: Sheet data rows as JSON.  
    - Version: 4 (latest stable for Google Sheets).  
    - Potential failure types:  
      - OAuth token expiry or permission errors.  
      - Invalid or missing sheet ID.  
      - Sheet name not found or changed.  
      - Google API rate limiting or network issues.  
    - Sub-workflow: None.

#### 1.4 CSV Conversion

- **Overview:**  
  Converts the retrieved spreadsheet data into CSV file format.

- **Nodes Involved:**  
  - Convert to CSV

- **Node Details:**

  - **Convert to CSV**  
    - Type & Role: Spreadsheet File node converting JSON data to CSV file.  
    - Configuration:  
      - Operation: `toFile`  
      - File format: `csv`  
    - Input: Sheet data JSON from Google Sheets node.  
    - Output: Binary CSV file data.  
    - Version: v1.  
    - Potential failure types:  
      - Data format inconsistencies causing conversion errors.  
      - Large data sets potentially causing memory/timeouts.  
    - Sub-workflow: None.

#### 1.5 Slack File Upload

- **Overview:**  
  Uploads the generated CSV file to the Slack channel where the initial message was received.

- **Nodes Involved:**  
  - Upload to Slack

- **Node Details:**

  - **Upload to Slack**  
    - Type & Role: Slack node to upload files to channels.  
    - Configuration:  
      - File name: fixed string "Converted CSV" (no dynamic naming).  
      - Upload channel: statically set to `C099B0PKEF2`.  
      - Uses OAuth2 authentication.  
      - Sends binary data from previous CSV conversion node.  
    - Credentials: Slack OAuth2 credentials named “Slack account 3”.  
    - Input: Binary CSV file from “Convert to CSV” node.  
    - Output: Slack file upload response.  
    - Version: v1.  
    - Potential failure types:  
      - OAuth token expiry or permission issues.  
      - File size limits on Slack uploads.  
      - Network or Slack API rate limits.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name        | Node Type               | Functional Role                       | Input Node(s)       | Output Node(s)         | Sticky Note                         |
|------------------|-------------------------|------------------------------------|---------------------|------------------------|-----------------------------------|
| Slack Trigger    | Slack Trigger           | Listen for Slack channel events     | -                   | Extract Sheet ID       |                                   |
| Extract Sheet ID | Function                | Extract Google Sheets ID from text  | Slack Trigger       | Read Google Sheet      |                                   |
| Read Google Sheet| Google Sheets           | Read data from Google Sheet         | Extract Sheet ID    | Convert to CSV         |                                   |
| Convert to CSV   | Spreadsheet File        | Convert sheet data JSON to CSV file | Read Google Sheet   | Upload to Slack        |                                   |
| Upload to Slack  | Slack                   | Upload CSV file back to Slack       | Convert to CSV      | -                      |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Type: Slack Trigger  
   - Configure trigger events: Select both “any_event” and “app_mention”.  
   - Set channel: Use channel ID `C099B0PKEF2`.  
   - Credentials: Assign Slack API credentials (OAuth or token) authorized to listen in the channel.

2. **Create Extract Sheet ID Node**  
   - Type: Function  
   - Connect Slack Trigger node output → this node input.  
   - Paste this JavaScript code:
     ```javascript
     const regex = /https:\/\/docs\.google\.com\/spreadsheets\/d\/([a-zA-Z0-9-_]+)/;
     const match = regex.exec($json.text);
     if (match) {
       return [{
         sheetId: match[1],
         channel: $json.channel
       }];
     }
     return [];
     ```
   - This extracts the spreadsheet ID from the message text.

3. **Create Read Google Sheet Node**  
   - Type: Google Sheets  
   - Connect Extract Sheet ID output → this node input.  
   - Set `documentId` parameter to expression: `={{ $json.sheetId }}` to dynamically use extracted ID.  
   - Set `sheetName` to the sheet’s GID or name; in this case, use the sheet with GID `253150622` (Sheet4).  
   - Credentials: Assign Google Sheets OAuth2 credentials with read access.

4. **Create Convert to CSV Node**  
   - Type: Spreadsheet File  
   - Connect Read Google Sheet output → this node input.  
   - Operation: `toFile`  
   - File format: `csv`  
   - No additional configuration needed.

5. **Create Upload to Slack Node**  
   - Type: Slack  
   - Connect Convert to CSV output → this node input.  
   - Configure resource: `file`  
   - Set `fileName` to “Converted CSV”.  
   - Set `channelIds` to `["C099B0PKEF2"]`.  
   - Enable binary data upload.  
   - Authentication: OAuth2.  
   - Credentials: Assign Slack OAuth2 credentials with file upload permissions.

6. **Connect nodes in the order:**  
   Slack Trigger → Extract Sheet ID → Read Google Sheet → Convert to CSV → Upload to Slack.

7. **Activate the workflow and test by sending a Google Sheets URL or app mention in the Slack channel.**  
   - Ensure that the Slack app has necessary permissions (events and file uploads).  
   - Ensure Google Sheets credentials have access to the target sheets.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                 |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------|
| Slack app must be installed with permissions for event subscriptions and file uploads in the channel.| Slack API documentation and app setup guides.  |
| Google Sheets OAuth2 credentials require read access to target sheets.                               | Google Cloud Console and OAuth2 setup docs.    |
| The regex used assumes standard Google Sheets URL format: `https://docs.google.com/spreadsheets/d/{sheetId}`. | Adjust regex if URLs vary or are shortened.    |
| Slack message must contain the full URL; partial or malformed links will not trigger conversion.    | Message formatting guidelines for users.        |

---

**Disclaimer:** The provided text is exclusively sourced from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.