Webpage Change Detection & Alerts with Google Suite and Hash Tracking

https://n8nworkflows.xyz/workflows/webpage-change-detection---alerts-with-google-suite-and-hash-tracking-3366


# Webpage Change Detection & Alerts with Google Suite and Hash Tracking

### 1. Workflow Overview

This workflow automates the monitoring of webpage content changes and sends notifications only when a change is detected. It is designed for tracking publicly accessible web documents such as company Terms of Service, government policies, or competitor pages.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Variable Setup**: Initiates the workflow on a schedule and sets the URL to monitor.
- **1.2 Webpage Fetching & Content Extraction**: Retrieves the webpage and extracts the specific HTML content to track.
- **1.3 Content Hashing & Change Detection**: Generates a hash of the extracted content and filters out previously seen hashes to detect changes.
- **1.4 Change Handling & Notification**: When a new change is detected, saves a snapshot to Google Drive, logs the event in Google Sheets, and sends an email notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Variable Setup

**Overview:**  
This block triggers the workflow on a recurring schedule and defines the URL of the webpage to monitor.

**Nodes Involved:**  
- Schedule Trigger  
- Variables

**Node Details:**

- **Schedule Trigger**  
  - *Type & Role:* Schedule Trigger node; initiates workflow execution automatically based on a time interval.  
  - *Configuration:* Runs on a daily interval (default daily schedule).  
  - *Inputs:* None (start node).  
  - *Outputs:* Connects to Variables node.  
  - *Edge Cases:* Misconfigured timezone may cause unexpected trigger times.

- **Variables**  
  - *Type & Role:* Set node; stores the URL to monitor as a workflow variable.  
  - *Configuration:* Sets a single string variable `url` with the value `https://x.com/en/tos`.  
  - *Inputs:* From Schedule Trigger.  
  - *Outputs:* Connects to Fetch Webpage node.  
  - *Edge Cases:* URL must be updated manually for different targets; no validation on URL format.

---

#### 2.2 Webpage Fetching & Content Extraction

**Overview:**  
Fetches the webpage content using HTTP request and extracts the relevant HTML segment for monitoring.

**Nodes Involved:**  
- Fetch Webpage  
- Extract Contents  
- Markdown

**Node Details:**

- **Fetch Webpage**  
  - *Type & Role:* HTTP Request node; retrieves the webpage content.  
  - *Configuration:*  
    - URL dynamically set from `url` variable.  
    - Sends a custom `User-Agent` header mimicking a modern browser to avoid blocking.  
  - *Inputs:* From Variables node.  
  - *Outputs:* Connects to Extract Contents node.  
  - *Edge Cases:*  
    - HTTP errors (404, 500, etc.) may cause failure.  
    - Pages behind authentication or with dynamic content may not be fetched correctly.  
    - Rate limiting or IP blocking by target server possible.

- **Extract Contents**  
  - *Type & Role:* HTML node; extracts specific HTML content using CSS selectors.  
  - *Configuration:*  
    - Extracts HTML content matching CSS selector `.ct07`.  
    - Returns raw HTML snippet under key `content`.  
  - *Inputs:* From Fetch Webpage.  
  - *Outputs:* Connects to Markdown node.  
  - *Edge Cases:*  
    - Selector must be updated if webpage structure changes.  
    - Empty or missing content if selector is incorrect.

- **Markdown**  
  - *Type & Role:* Markdown node; converts extracted HTML content to markdown format.  
  - *Configuration:*  
    - Converts trimmed HTML content from `content` field.  
  - *Inputs:* From Extract Contents.  
  - *Outputs:* Connects to Get Hash of Contents node.  
  - *Edge Cases:* Conversion errors if HTML is malformed.

---

#### 2.3 Content Hashing & Change Detection

**Overview:**  
Generates a hash of the extracted content and filters out hashes seen in previous executions to detect new changes.

**Nodes Involved:**  
- Get Hash of Contents  
- Only New Hashes

**Node Details:**

- **Get Hash of Contents**  
  - *Type & Role:* Cryptography node; generates a hash string from content.  
  - *Configuration:*  
    - Hash input is the markdown content (`data` property).  
    - Output property named `hash`.  
  - *Inputs:* From Markdown node.  
  - *Outputs:* Connects to Only New Hashes node.  
  - *Edge Cases:*  
    - Hashing failure if input is empty or invalid.  
    - Hash algorithm details not explicitly set (default used).

- **Only New Hashes**  
  - *Type & Role:* Remove Duplicates node; filters out hashes seen in previous workflow executions.  
  - *Configuration:*  
    - Deduplication based on `hash` property.  
    - History size set to 1 (only remembers last execution).  
  - *Inputs:* From Get Hash of Contents.  
  - *Outputs:* Connects to Take a Snapshot node if new hash detected; otherwise stops workflow.  
  - *Edge Cases:*  
    - If history is lost or reset, may resend notifications.  
    - Only tracks one previous hash, so rapid changes may be missed.

---

#### 2.4 Change Handling & Notification

**Overview:**  
When a new change is detected, saves a snapshot of the content to Google Drive, logs the change in Google Sheets, and sends an email notification.

**Nodes Involved:**  
- Take a Snapshot  
- Log Record  
- Notify of Change

**Node Details:**

- **Take a Snapshot**  
  - *Type & Role:* Google Drive node; uploads the markdown content as a file.  
  - *Configuration:*  
    - Filename constructed dynamically from domain extracted from URL plus hash, with `.md` extension.  
    - Content is the markdown data.  
    - Uploads to a specific Google Drive folder (`86. Monitor Webpage Changes`).  
  - *Inputs:* From Only New Hashes.  
  - *Outputs:* Connects to Log Record node.  
  - *Credentials:* Requires Google Drive OAuth2 credentials.  
  - *Edge Cases:*  
    - Upload failure due to auth errors or quota limits.  
    - Filename conflicts unlikely due to hash suffix.

- **Log Record**  
  - *Type & Role:* Google Sheets node; appends a new row logging the change event.  
  - *Configuration:*  
    - Columns: website URL, hash, date of change (ISO timestamp), Google Drive file link.  
    - Appends to a specific Google Sheet (`86. Webpage Changes Tracker`, Sheet1).  
  - *Inputs:* From Take a Snapshot.  
  - *Outputs:* Connects to Notify of Change node.  
  - *Credentials:* Requires Google Sheets OAuth2 credentials.  
  - *Edge Cases:*  
    - Failure if sheet ID or permissions are incorrect.  
    - Data format must match schema.

- **Notify of Change**  
  - *Type & Role:* Gmail node; sends an email notification about the detected change.  
  - *Configuration:*  
    - Recipient: `jim@example.com` (hardcoded).  
    - Subject includes the monitored URL.  
    - Email body includes site URL, timestamp, hash, and Google Drive link.  
    - Email type: plain text.  
  - *Inputs:* From Log Record.  
  - *Outputs:* None (end node).  
  - *Credentials:* Requires Gmail OAuth2 credentials.  
  - *Edge Cases:*  
    - Email sending failure due to auth or quota limits.  
    - Recipient address must be updated for actual use.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                      | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                  |
|---------------------|---------------------|------------------------------------|-----------------------|----------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger    | Initiates workflow on schedule     | None                  | Variables            |                                                                                                              |
| Variables           | Set                 | Stores URL to monitor               | Schedule Trigger      | Fetch Webpage        |                                                                                                              |
| Fetch Webpage        | HTTP Request        | Fetches webpage content             | Variables             | Extract Contents     | ## 1. Fetch a Webpage Contents [Read more about the HTTP request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Extract Contents     | HTML                | Extracts specific HTML content      | Fetch Webpage         | Markdown             |                                                                                                              |
| Markdown             | Markdown            | Converts HTML to markdown           | Extract Contents      | Get Hash of Contents | ## 2. Use Hashing to Detect Changes [Learn more about the cryptography node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.crypto/) |
| Get Hash of Contents | Cryptography        | Generates hash of content           | Markdown              | Only New Hashes      |                                                                                                              |
| Only New Hashes      | Remove Duplicates   | Filters previously seen hashes      | Get Hash of Contents  | Take a Snapshot      |                                                                                                              |
| Take a Snapshot      | Google Drive        | Uploads content snapshot            | Only New Hashes       | Log Record           |                                                                                                              |
| Log Record           | Google Sheets       | Logs change event                   | Take a Snapshot       | Notify of Change     |                                                                                                              |
| Notify of Change     | Gmail               | Sends email notification            | Log Record            | None                 | ## 3. Notify when Changes Occur [Read more about the Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail) |
| Sticky Note          | Sticky Note         | Informational note                  | None                  | None                 | ## 1. Fetch a Webpage Contents [Read more about the HTTP request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Sticky Note1         | Sticky Note         | Informational note                  | None                  | None                 | ## 2. Use Hashing to Detect Changes [Learn more about the cryptography node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.crypto/) |
| Sticky Note2         | Sticky Note         | Informational note                  | None                  | None                 | ## 3. Notify when Changes Occur [Read more about the Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail) |
| Sticky Note3         | Sticky Note         | Workflow overview and usage notes   | None                  | None                 | ## Try it out - Full workflow description and usage instructions with community links                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to daily (default)  
   - Position: start of the workflow

2. **Create Variables node**  
   - Type: Set  
   - Add string variable `url` with value `https://x.com/en/tos` (update as needed)  
   - Connect Schedule Trigger → Variables

3. **Create Fetch Webpage node**  
   - Type: HTTP Request  
   - Set URL to `={{ $json.url }}` (dynamic from Variables)  
   - Add header: `User-Agent` = `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36`  
   - Connect Variables → Fetch Webpage

4. **Create Extract Contents node**  
   - Type: HTML  
   - Operation: Extract HTML content  
   - Add extraction value: key=`content`, CSS selector=`.ct07`, return value=`html`  
   - Connect Fetch Webpage → Extract Contents

5. **Create Markdown node**  
   - Type: Markdown  
   - Input HTML: `={{ $json.content.trim() }}`  
   - Connect Extract Contents → Markdown

6. **Create Get Hash of Contents node**  
   - Type: Cryptography  
   - Value to hash: `={{ $json.data }}` (markdown content)  
   - Output property: `hash`  
   - Connect Markdown → Get Hash of Contents

7. **Create Only New Hashes node**  
   - Type: Remove Duplicates  
   - Operation: Remove items seen in previous executions  
   - Deduplication value: `={{ $json.hash }}`  
   - History size: 1  
   - Connect Get Hash of Contents → Only New Hashes

8. **Create Take a Snapshot node**  
   - Type: Google Drive  
   - Operation: Create file from text  
   - Filename: `={{ $('Variables').item.json.url.extractDomain().replace('.','_') + $json.hash + '.md' }}`  
   - Content: `={{ $json.data }}`  
   - Drive: My Drive  
   - Folder: Select or enter folder ID for snapshots (e.g., `86. Monitor Webpage Changes`)  
   - Connect Only New Hashes → Take a Snapshot  
   - Configure Google Drive OAuth2 credentials

9. **Create Log Record node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Google Sheet ID for logging (e.g., `86. Webpage Changes Tracker`)  
   - Sheet Name: `gid=0` or sheet name  
   - Columns mapping:  
     - `website`: `={{ $('Variables').first().json.url }}`  
     - `hash`: `={{ $('Get Hash of Contents').first().json.hash }}`  
     - `date of change`: `={{ $now.toISO() }}`  
     - `gdrive`: `=https://drive.google.com/file/d/{{ $json.id }}/view?usp=sharing`  
   - Connect Take a Snapshot → Log Record  
   - Configure Google Sheets OAuth2 credentials

10. **Create Notify of Change node**  
    - Type: Gmail  
    - Recipient: `jim@example.com` (update as needed)  
    - Subject: `=Change detected for {{ $('Variables').first().json.url }}`  
    - Message body:  
      ```
      site: {{ $('Variables').first().json.url }}
      date: {{ $now.toISO() }}
      hash: {{ $json.hash }}
      contents: {{ $json.gdrive }}
      ```  
    - Email type: Text  
    - Connect Log Record → Notify of Change  
    - Configure Gmail OAuth2 credentials

11. **Add Sticky Notes** (optional)  
    - Add informational sticky notes near relevant nodes with provided content and links for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for monitoring publicly accessible webpages such as Terms of Service, government policies, or competitor pages.                                                                                                  | Workflow purpose                                                                                                 |
| If the webpage is not publicly accessible, consider replacing the HTTP Request node with a web scraping service that supports authentication.                                                                                           | Usage note                                                                                                       |
| To track multiple URLs, modify the Variables node to accept an array and adjust the HTML node selectors accordingly.                                                                                                                    | Customization tip                                                                                                |
| Ensure the Scheduled Trigger node’s timezone is set correctly to match your desired monitoring schedule.                                                                                                                                | Usage note                                                                                                       |
| Google Sheets, Drive, and Gmail OAuth2 credentials must be configured and authorized before running the workflow.                                                                                                                       | Credential requirement                                                                                           |
| Community support is available via the [n8n Discord](https://discord.com/invite/XPKeKXeB7d) and [n8n Forum](https://community.n8n.io/).                                                                                                  | Support links                                                                                                    |
| Documentation links for key nodes: HTTP Request ([docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)), Cryptography ([docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.crypto/)), Gmail ([docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail)) | Node documentation                                                                                               |

---

This structured reference document provides a comprehensive understanding of the "Webpage Change Detection & Alerts with Google Suite and Hash Tracking" workflow, enabling users and AI agents to reproduce, modify, and troubleshoot it effectively.