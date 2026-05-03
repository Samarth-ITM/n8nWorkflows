Malicious File Detection & Response: Wazuh to VirusTotal with Slack Alerts

https://n8nworkflows.xyz/workflows/malicious-file-detection---response--wazuh-to-virustotal-with-slack-alerts-5997


# Malicious File Detection & Response: Wazuh to VirusTotal with Slack Alerts

### 1. Workflow Overview

This workflow automates the detection and response process for potentially malicious files identified by Wazuh alerts. It integrates threat intelligence from VirusTotal to validate file hashes and enrich alerts with detailed reputation and detection data. It then categorizes files as safe or suspicious and escalates suspicious detections by notifying analysts through Slack, creating ServiceNow incidents, and emailing a formatted threat summary.

Logical blocks:

- **1.1 Alert Ingestion & IOC Extraction:** Receives Wazuh file integrity alerts via webhook and extracts key indicators of compromise (IOCs) such as file hashes and metadata.
- **1.2 VirusTotal Enrichment & Threat Summary:** Queries VirusTotal API with extracted SHA256 hash to obtain threat intelligence, generates a comprehensive file threat summary in HTML.
- **1.3 Alert Escalation & Notification:** Filters alerts based on threat status, sends Slack notifications, creates ServiceNow incidents for suspicious files, and emails the file summary to security analysts.

---

### 2. Block-by-Block Analysis

#### 2.1 Alert Ingestion & IOC Extraction

**Overview:**  
This block receives incoming alerts from Wazuh via an HTTP webhook, processes the payload to extract relevant file indicators and alert metadata, and prepares the data for threat validation.

**Nodes Involved:**  
- Wazuh Alert (Webhook)  
- Extract IOCs (Code)  

**Node Details:**  

- **Wazuh Alert**  
  - Type: Webhook  
  - Role: Entry point for file integrity alerts from Wazuh. Accepts POST requests on path `/file_validation`.  
  - Key Config: HTTP method POST, webhook path `file_validation`.  
  - Input: External HTTP request with Wazuh alert JSON payload.  
  - Output: Raw alert JSON passed to next node.  
  - Edge Cases: Malformed webhook payload, network errors, unauthorized requests (authentication not configured here).  

- **Extract IOCs**  
  - Type: Code (JavaScript)  
  - Role: Parses Wazuh alert JSON to extract hashes (MD5, SHA1, SHA256), file path, alert description, agent name, and alert severity level.  
  - Key Expressions: Extracts nested fields from `body.syscheck`, `body.rule`, and `body.agent`. Outputs a simplified JSON object with these fields plus full alert for traceability.  
  - Input: Data from Wazuh Alert node.  
  - Output: JSON with fields: `md5`, `sha1`, `sha256`, `file_path`, `description`, `agent`, `level`, and `full_alert`.  
  - Edge Cases: Missing or null fields in alert payload; defaults are assigned for missing descriptions/agent names.  

---

#### 2.2 VirusTotal Enrichment & Threat Summary

**Overview:**  
This block enriches the extracted file hash data by querying the VirusTotal API to obtain reputation scores, detection statistics, threat labels, and tags. It then formats this data into a human-readable HTML summary.

**Nodes Involved:**  
- VirusTotal File Hash Validation (HTTP Request)  
- Generate File Summary (Code)  
- file summary display (HTML)  

**Node Details:**  

- **VirusTotal File Hash Validation**  
  - Type: HTTP Request  
  - Role: Calls VirusTotal API to validate the SHA256 file hash and retrieve detailed file analysis data.  
  - Key Config: URL constructed dynamically using extracted SHA256 hash. Uses VirusTotal API credentials for authentication.  
  - Input: JSON containing `sha256` from Extract IOCs.  
  - Output: VirusTotal API response JSON with file attributes and analysis stats.  
  - Edge Cases: API rate limits, invalid/missing SHA256 hash, network timeouts, API authentication errors. Configured to continue on error to prevent workflow breakage.  

- **Generate File Summary**  
  - Type: Code (JavaScript)  
  - Role: Processes VirusTotal response to extract key attributes: reputation, threat classification, detection stats (malicious, suspicious, harmless, undetected), tags, file magic signature, and meaningful name.  
  - Key Expressions: Determines overall status as "Suspicious" if any malicious or suspicious detections exist, otherwise "Safe". Formats tags as HTML spans for display.  
  - Input: JSON from VirusTotal File Hash Validation.  
  - Output: JSON containing a `summary` object with all extracted and computed fields, including a localized timestamp.  
  - Edge Cases: Missing fields in VirusTotal response default to safe values or placeholders.  

- **file summary display**  
  - Type: HTML  
  - Role: Renders the generated file summary in a styled HTML format suitable for email or display.  
  - Key Config: Uses embedded HTML template with CSS for dark theme styling and dynamic content placeholders bound to the `summary` JSON fields.  
  - Input: JSON summary from Generate File Summary node.  
  - Output: HTML string passed to the email node.  
  - Edge Cases: Rendering issues if summary fields are missing or malformed; mitigated by default placeholders.  

---

#### 2.3 Alert Escalation & Notification

**Overview:**  
This block routes the alert based on the threat status. For suspicious files, it sends Slack alerts and creates ServiceNow incidents. It also sends an email with the formatted file summary to security analysts.

**Nodes Involved:**  
- Filter Suspicious Files (Switch)  
- Slack File Alert (Slack)  
- Create File Incident (ServiceNow)  
- Gmail1 (Gmail)  

**Node Details:**  

- **Filter Suspicious Files**  
  - Type: Switch  
  - Role: Routes workflow execution based on the `summary.Status` field. Only "Suspicious" status triggers the escalation path.  
  - Key Config: Checks if `summary.Status` equals "Suspicious".  
  - Input: JSON summary from Generate File Summary.  
  - Output: Two branches — suspicious (true) and safe (false, implicitly ignored here).  
  - Edge Cases: Unexpected status values; configured strictly for "Suspicious".  

- **Slack File Alert**  
  - Type: Slack  
  - Role: Sends a formatted alert message to a specific Slack channel notifying about the malicious file detection.  
  - Key Config: OAuth2 authentication, channel ID hardcoded, message dynamically constructed with file name, partial SHA256, status, and threat label.  
  - Input: Suspicious alert summary from Filter Suspicious Files.  
  - Output: Slack message confirmation.  
  - Edge Cases: Slack API rate limits, authentication expiration, channel permission issues.  

- **Create File Incident**  
  - Type: ServiceNow  
  - Role: Creates an incident ticket in ServiceNow to track the malicious file detection case.  
  - Key Config: Basic authentication credentials for ServiceNow, short description includes truncated SHA256, file name, status, and threat classification.  
  - Input: Suspicious alert summary from Filter Suspicious Files.  
  - Output: ServiceNow incident creation response.  
  - Edge Cases: ServiceNow API failures, auth errors, field validation errors.  

- **Gmail1**  
  - Type: Gmail  
  - Role: Emails the HTML-formatted file threat summary to a dedicated security analyst inbox.  
  - Key Config: OAuth2 Gmail credentials, recipient fixed as `aman@amvic.in`, subject "Alert", message body uses HTML from file summary display node.  
  - Input: HTML summary from file summary display node.  
  - Output: Email send confirmation.  
  - Edge Cases: Gmail API quota, auth token expiration, invalid recipient address.  

---

### 3. Summary Table

| Node Name                  | Node Type        | Functional Role                           | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                                  |
|----------------------------|------------------|-----------------------------------------|---------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Wazuh Alert                | Webhook          | Receives Wazuh file integrity alerts    | External HTTP request     | Extract IOCs                       | ## Alert Ingestion & Threat Intelligence - Receives file integrity alerts via webhook.                        |
| Extract IOCs               | Code             | Extracts hashes and metadata from alert | Wazuh Alert               | VirusTotal File Hash Validation     | ## Alert Ingestion & Threat Intelligence - Extracts SHA256, MD5, filename, path, and agent info.              |
| VirusTotal File Hash Validation | HTTP Request  | Queries VirusTotal with file SHA256     | Extract IOCs              | Generate File Summary              | ## VirusTotal Enrichment & Threat Summary - Validates file hash with VirusTotal API.                          |
| Generate File Summary      | Code             | Creates detailed threat summary         | VirusTotal File Hash Validation | file summary display, Filter Suspicious Files | ## VirusTotal Enrichment & Threat Summary - Gathers reputation, detection stats, threat label, and tags.      |
| file summary display       | HTML             | Formats summary into styled HTML        | Generate File Summary     | Gmail1                            | ## VirusTotal Enrichment & Threat Summary - Generates a readable HTML summary with file context.              |
| Filter Suspicious Files    | Switch           | Routes alerts based on threat status    | Generate File Summary     | Slack File Alert, Create File Incident | ## Alert Escalation & Analyst Notification - Routes alerts based on threat level.                             |
| Slack File Alert           | Slack            | Sends Slack notifications for suspicious files | Filter Suspicious Files | -                                  | ## Alert Escalation & Analyst Notification - Sends Slack alert if suspicious.                                |
| Create File Incident       | ServiceNow       | Creates incident tickets for suspicious files | Filter Suspicious Files | -                                  | ## Alert Escalation & Analyst Notification - Creates ServiceNow ticket if suspicious.                        |
| Gmail1                     | Gmail            | Sends email with file threat summary    | file summary display      | -                                  | ## Alert Escalation & Analyst Notification - Emails formatted threat summary to analyst inbox.               |
| Sticky Note                | Sticky Note      | Documentation note                      | -                         | -                                  | ## Alert Ingestion & Threat Intelligence - Receives file integrity alerts via webhook and extracts metadata. |
| Sticky Note1               | Sticky Note      | Documentation note                      | -                         | -                                  | ## Alert Escalation & Analyst Notification - Escalates suspicious alerts via Slack, ServiceNow, and email.   |
| Sticky Note2               | Sticky Note      | Documentation note                      | -                         | -                                  | ## VirusTotal Enrichment & Threat Summary - Validates file hash and generates threat summary.                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Wazuh Alert"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `file_validation`  
   - Purpose: Receive Wazuh file integrity alerts as JSON payload.  

2. **Add "Extract IOCs" Code Node**  
   - Type: Code (JavaScript)  
   - Connect input from "Wazuh Alert".  
   - Script extracts: `md5`, `sha1`, `sha256`, `file_path`, `description`, `agent`, `level` from alert JSON.  
   - Outputs simplified JSON for downstream processing.  

3. **Add HTTP Request Node "VirusTotal File Hash Validation"**  
   - Connect input from "Extract IOCs".  
   - Method: GET  
   - URL: `https://www.virustotal.com/api/v3/files/{{ $json.sha256 }}` (Use expression to insert SHA256).  
   - Authentication: Use VirusTotal API credentials (predefined OAuth2 or API Key).  
   - On Error: Continue regular output to avoid workflow stop on API issues.  

4. **Add Code Node "Generate File Summary"**  
   - Connect input from "VirusTotal File Hash Validation".  
   - Script parses VirusTotal response: extracts reputation, detection stats, tags, threat labels, magic signature, meaningful name.  
   - Adds a `Status` field: "Suspicious" if malicious or suspicious detections > 0, else "Safe".  
   - Creates timestamp localized to Asia/Kolkata timezone.  

5. **Add HTML Node "file summary display"**  
   - Connect input from "Generate File Summary".  
   - Paste the HTML template that formats the file summary with CSS styling and dynamic placeholders bound to `summary` fields.  

6. **Add Gmail Node "Gmail1"**  
   - Connect input from "file summary display".  
   - Credentials: Gmail OAuth2 account.  
   - Recipient: `aman@amvic.in` (set as sendTo).  
   - Subject: "Alert"  
   - Body: Use the HTML content from previous node with `={{ $json.html }}` expression.  

7. **Add Switch Node "Filter Suspicious Files"**  
   - Connect input from "Generate File Summary".  
   - Rule: Check if `{{$json.summary.Status}} === 'Suspicious'`.  
   - True branch proceeds to escalation nodes.  

8. **Add Slack Node "Slack File Alert"**  
   - Connect input from "Filter Suspicious Files" true output.  
   - Authentication: OAuth2 Slack account.  
   - Channel: Select target channel ID (e.g., `"C0913JPTZBJ"`).  
   - Message: Construct dynamic alert with file name, truncated SHA256, status, and threat label.  

9. **Add ServiceNow Node "Create File Incident"**  
   - Connect input from "Filter Suspicious Files" true output.  
   - Resource: Incident  
   - Operation: Create  
   - Authentication: Basic Auth credentials for ServiceNow instance.  
   - Short Description: Dynamic text including file name, first 12 chars of SHA256, status, and threat classification.  

10. **Connect all nodes according to the above flow:**  
    - Wazuh Alert → Extract IOCs → VirusTotal File Hash Validation → Generate File Summary →  
      - file summary display → Gmail1  
      - Filter Suspicious Files → (True) Slack File Alert & Create File Incident  

11. **Credential Configuration:**  
    - VirusTotal API credentials (API Key or OAuth2) for HTTP Request node.  
    - Gmail OAuth2 credentials for Gmail node.  
    - Slack OAuth2 credentials for Slack node.  
    - ServiceNow Basic Auth credentials for ServiceNow node.  

12. **Set Node Versions:**  
    - Prefer recent stable versions: e.g., Webhook v2, HTTP Request v4, Slack v2.3, Gmail v2.1.  

13. **Test the Workflow:**  
    - Trigger webhook with sample Wazuh alert payload.  
    - Verify VirusTotal API response, HTML summary generation, Slack notification, ServiceNow incident creation, and email delivery.  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow uses a clean, dark-themed HTML template for threat summaries designed for readability and quick analysis. | Embedded in “file summary display” HTML node.                                                      |
| Slack channel is preconfigured; ensure correct channel ID and OAuth2 token with permissions to post messages.    | Slack File Alert node configuration.                                                               |
| ServiceNow integration requires valid API credentials and incident creation permissions.                         | ServiceNow node configuration, Basic Auth credentials.                                             |
| Gmail node uses OAuth2; ensure token refresh and correct sender email are configured.                             | Gmail1 node configuration.                                                                          |
| VirusTotal API rate limits may impact throughput; errors are handled gracefully to avoid workflow failure.        | HTTP Request node “onError” set to continue.                                                       |
| Time zone for timestamps is set to “Asia/Kolkata” for local relevance; customize if needed.                      | Generate File Summary node JavaScript code.                                                        |

---

This document fully describes the "Malicious File Detection & Response: Wazuh to VirusTotal with Slack Alerts" workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents.