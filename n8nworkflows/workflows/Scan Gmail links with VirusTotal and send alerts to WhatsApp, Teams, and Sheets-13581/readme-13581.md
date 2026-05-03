Scan Gmail links with VirusTotal and send alerts to WhatsApp, Teams, and Sheets

https://n8nworkflows.xyz/workflows/scan-gmail-links-with-virustotal-and-send-alerts-to-whatsapp--teams--and-sheets-13581


# Scan Gmail links with VirusTotal and send alerts to WhatsApp, Teams, and Sheets

# Workflow Reference: Automatically Detect Suspicious Links in Gmail Messages

This document provides a technical breakdown of an n8n workflow designed to automate the security scanning of URLs found within incoming Gmail messages. It leverages the VirusTotal API for threat intelligence and distributes alerts across multiple communication and logging platforms.

---

### 1. Workflow Overview

The workflow monitors a Gmail account for new messages, extracts all embedded URLs, and filters them to remove trusted domains. Each remaining URL is submitted to VirusTotal for analysis. Once analyzed, the results are parsed and simultaneously sent to a WhatsApp contact, a Microsoft Teams channel, and logged into a Google Sheet for auditing.

#### Logical Blocks:
*   **1.1 Email Monitoring & URL Extraction:** Triggers on new emails and uses custom logic to identify and isolate links from the email body.
*   **1.2 Filtering & Iteration:** Loops through the list of extracted URLs, applying filters to skip known safe domains (e.g., Google) and ensuring the data is valid.
*   **1.3 Security Analysis:** Interfaces with the VirusTotal API to submit URLs for scanning and retrieve detailed security reports.
*   **1.4 Alerting & Logging:** Distributes the findings to external platforms (WhatsApp, Teams, and Google Sheets).

---

### 2. Block-by-Block Analysis

#### 2.1 Email Monitoring & URL Extraction
*   **Overview:** This block detects incoming emails and prepares the raw text for processing.
*   **Nodes Involved:** 
    *   `Gmail Trigger`
    *   `Code (Extracts URLs from email content)`
    *   `Code (Converts the array of links into individual)`
*   **Node Details:**
    *   **Gmail Trigger:** A polling or webhook-based node (Version 1.3) that triggers on new messages. Requires Gmail OAuth2 credentials.
    *   **Code (Extracts URLs):** Uses a JavaScript regex to find strings matching URL patterns within the `body` of the email.
    *   **Code (Converts array):** Transforms the list of URLs into individual n8n items to facilitate looping.

#### 2.2 Filtering & Iteration
*   **Overview:** Manages the flow of data to ensure only relevant and non-empty URLs are sent to the security scanner.
*   **Nodes Involved:**
    *   `Loop Over Items`
    *   `If (Checks if URLs contain "google.com" using regex)`
    *   `If (Checks if URLs are not empty)`
*   **Node Details:**
    *   **Loop Over Items:** Standard `Split in Batches` node (Version 3) to process one URL at a time.
    *   **If (Google Filter):** Uses a regular expression to identify "google.com" links. If a match is found (True), it returns to the loop to skip the scan; if False, it proceeds to the next check.
    *   **If (Empty Check):** A safety check (Version 2.3) to ensure the URL string is not null or empty before hitting the API.

#### 2.3 Security Analysis
*   **Overview:** Handles the asynchronous process of submitting a URL and fetching its reputation report.
*   **Nodes Involved:**
    *   `VirusTotal (Submits URLs for analysis)`
    *   `Analysis report from VirusTotal`
*   **Node Details:**
    *   **VirusTotal (Submits):** An HTTP Request node (Version 4.3) using VirusTotal API credentials to POST the URL to the `/urls` endpoint.
    *   **Analysis report:** A secondary HTTP Request (Version 4.1) that retrieves the analysis results using the ID provided by the submission node.
    *   **Failure Modes:** API Rate limits (VirusTotal free tier has strict limits), Invalid API Key, or timeouts.

#### 2.4 Alerting & Logging
*   **Overview:** Formats the security data and notifies the relevant stakeholders.
*   **Nodes Involved:**
    *   `Code (VirusTotal report to extract relevant information)`
    *   `Rapiwa (WhatsApp notification)`
    *   `Logs the analysis results to a Google Sheet`
    *   `Sends a Microsoft Teams notification`
*   **Node Details:**
    *   **Code (Extract Info):** Parses the complex VirusTotal JSON response to extract specific metrics like "malicious" and "suspicious" counts.
    *   **Rapiwa:** Sends a WhatsApp message via the Rapiwa API.
    *   **Google Sheets:** Appends a row to a specific spreadsheet (Version 4.7) containing the URL, timestamp, and scan results.
    *   **Microsoft Teams:** Sends a message or card to a predefined channel using a webhook or OAuth2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gmail Trigger | `n8n-nodes-base.gmailTrigger` | Trigger | None | Code (Extracts URLs) | |
| Code (Extracts URLs...) | `n8n-nodes-base.code` | Data Extraction | Gmail Trigger | Code (Converts array) | |
| Code (Converts array...) | `n8n-nodes-base.code` | Data Formatting | Code (Extracts URLs) | Loop Over Items | |
| Loop Over Items | `n8n-nodes-base.splitInBatches` | Flow Control | Code (Converts array), If (Empty), If (Google Filter) | If (Google Filter) | |
| If (Google Filter) | `n8n-nodes-base.if` | Whitelisting | Loop Over Items | Loop Over Items (True), If (Empty) (False) | |
| If (Empty) | `n8n-nodes-base.if` | Validation | If (Google Filter) | VirusTotal (True), Loop Over Items (False) | |
| VirusTotal (Submits...) | `n8n-nodes-base.httpRequest` | External API | If (Empty) | Analysis report | |
| Analysis report... | `n8n-nodes-base.httpRequest` | External API | VirusTotal (Submits) | Code (Extract report info) | |
| Code (Extract report info) | `n8n-nodes-base.code` | Data Parsing | Analysis report | Rapiwa, Google Sheets, Teams | |
| Rapiwa (WhatsApp) | `n8n-nodes-rapiwa.rapiwa` | Notification | Code (Extract report info) | None | |
| Logs to Google Sheet | `n8n-nodes-base.googleSheets` | Logging | Code (Extract report info) | None | |
| Sends a Microsoft Teams... | `n8n-nodes-base.microsoftTeams` | Notification | Code (Extract report info) | None | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Gmail Trigger** node. Connect your Gmail account via OAuth2 and set the event to "On New Email".
2.  **Extraction Logic:** 
    *   Add a **Code** node. Use JavaScript to extract URLs from the Gmail body: `const urls = item.json.textAsHtml.match(/https?:\/\/[^\s"<>]+/g) || [];`.
    *   Add a second **Code** node to return these as individual items: `return urls.map(url => ({ json: { url } }));`.
3.  **Looping:** Add a **Split in Batches** node.
4.  **Filtering:** 
    *   Add an **If** node to check if the URL contains "google.com" (or other trusted domains) using the `contains` operator or regex.
    *   Add another **If** node to verify the URL string length is greater than 0.
5.  **VirusTotal Integration:**
    *   Create a VirusTotal account and get an API Key.
    *   Add an **HTTP Request** node. Method: `POST`, URL: `https://www.virustotal.com/api/v3/urls`. Add a Header `x-apikey`.
    *   Add a second **HTTP Request** node. Method: `GET`. Use the analysis ID from the previous node to fetch the report: `https://www.virustotal.com/api/v3/analyses/{{$node["VirusTotal"].json["data"]["id"]}}`.
6.  **Data Processing:** Add a **Code** node to simplify the VirusTotal JSON. Map paths like `data.attributes.stats` to simpler keys like `malicious_count`.
7.  **Output Setup:**
    *   **Google Sheets:** Create a sheet with headers (Date, URL, Status). Add the Google Sheets node, select "Append Row," and map the parsed data.
    *   **WhatsApp:** Set up a Rapiwa node with your API key and target phone number.
    *   **Teams:** Set up a Microsoft Teams node. Choose "Send Message" and select your Team/Channel.
8.  **Connectivity:** Ensure the "False" path of the filters and the final output of the notification nodes loop back to the **Split in Batches** node until all items are processed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| VirusTotal API Documentation | [VirusTotal API v3 Guide](https://docs.virustotal.com/reference/overview) |
| n8n Gmail Integration Guide | [Gmail Node Setup](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) |
| Security Warning | Ensure your VirusTotal API Key is stored securely in n8n Credentials and not hardcoded. |
| Rate Limiting | The VirusTotal free API tier allows 4 requests per minute; adjust the "Batch Interval" in the Loop node if needed. |