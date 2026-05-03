IP Reputation Check & SOC Alerts with Splunk, VirusTotal and AlienVault

https://n8nworkflows.xyz/workflows/ip-reputation-check---soc-alerts-with-splunk--virustotal-and-alienvault-6037


# IP Reputation Check & SOC Alerts with Splunk, VirusTotal and AlienVault

### 1. Workflow Overview

This workflow automates the process of ingesting security alerts from Splunk related to IP addresses, enriching these alerts with threat intelligence data from VirusTotal and AlienVault, summarizing the findings, and notifying security operations center (SOC) analysts through multiple channels. It is designed for SOC teams to rapidly assess the reputation of suspicious IPs and respond accordingly.

The workflow is logically divided into three main blocks:

- **1.1 Alert Ingestion & IOC Extraction**  
  Receives alerts from Splunk, extracts the relevant IP and event details.

- **1.2 Threat Intelligence Enrichment & Summary Generation**  
  Queries VirusTotal and AlienVault for IP reputation data, merges results, and generates a detailed HTML summary.

- **1.3 Alert Routing & Analyst Notification**  
  Filters suspicious IPs to create incident tickets and send Slack alerts, and emails the summary report to the SOC inbox.

---

### 2. Block-by-Block Analysis

#### 2.1 Alert Ingestion & IOC Extraction

**Overview:**  
This block receives incoming alerts from Splunk via a webhook, extracts the IP address and event reason from the alert payload, and triggers threat intelligence lookups.

**Nodes Involved:**  
- Splunk Alert (Webhook)  
- Extract IOCs (Code)  
- VirusTotal IP reputation check (HTTP Request)  
- AlienVault Lookup (HTTP Request)  
- Merge Threat Data (Merge)

**Node Details:**

- **Splunk Alert**  
  - *Type:* Webhook  
  - *Role:* Entry point receiving POST alerts from Splunk containing IP-related security events.  
  - *Configuration:* HTTP POST method, unique path for triggering.  
  - *Input:* External Splunk alert payload.  
  - *Output:* JSON alert data forwarded to Extract IOCs node.  
  - *Failures:* Network issues, malformed payloads.  
  - *Notes:* Must be configured in Splunk to send alerts to this webhook URL.

- **Extract IOCs**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses Splunk alert JSON to extract `src_ip` and `reason`.  
  - *Configuration:* Uses optional chaining to safely extract `ip_address` and `description`.  
  - *Expressions:* `$input.first().json.body.result.src_ip` and `.reason`  
  - *Input:* JSON alert from Splunk Alert node.  
  - *Output:* JSON with fields: `ip_address`, `description`.  
  - *Failures:* Missing keys or unexpected JSON structure.  
  - *Edge cases:* Defaults to 'No IP found' or 'No reason found' if missing.

- **VirusTotal IP reputation check**  
  - *Type:* HTTP Request  
  - *Role:* Fetches IP reputation data from VirusTotal API using extracted IP.  
  - *Configuration:* GET request to `https://www.virustotal.com/api/v3/ip_addresses/{{ $json.ip_address }}`  
  - *Authentication:* Uses predefined VirusTotal API credentials.  
  - *Input:* JSON with `ip_address`.  
  - *Output:* VirusTotal API JSON response with IP data.  
  - *Failures:* API rate limiting, invalid API key, IP not found.  
  - *Notes:* Requires valid VirusTotal API key credential.

- **AlienVault Lookup**  
  - *Type:* HTTP Request  
  - *Role:* Queries AlienVault OTX API for IP threat info.  
  - *Configuration:* GET request to `https://otx.alienvault.com/api/v1/indicators/IPv4/{{ $json.ip_address }}`  
  - *Authentication:* Predefined AlienVault API credentials.  
  - *Input:* JSON with `ip_address`.  
  - *Output:* AlienVault API JSON response.  
  - *Failures:* API errors, rate limits, private IPs may return no data.  
  - *OnError:* Continue without failing workflow if API call errors.  
  - *Notes:* Requires AlienVault API credential.

- **Merge Threat Data**  
  - *Type:* Merge  
  - *Role:* Combines outputs from VirusTotal, AlienVault, and Extract IOCs nodes into a single data stream for processing.  
  - *Configuration:* Merges three inputs, mapping data by index.  
  - *Input:* Three streams: VirusTotal, AlienVault, Extract IOCs.  
  - *Output:* Single merged JSON object array.  
  - *Failures:* Mismatched inputs or missing data.  
  - *Notes:* Ensures all data sources are correlated per IP.

---

#### 2.2 Threat Intelligence Enrichment & Summary Generation

**Overview:**  
Processes merged threat data to build a structured summary, including reputation scores, analysis statistics, WHOIS info, and pulse counts. Generates an HTML summary report with styling suitable for SOC analysts.

**Nodes Involved:**  
- Process Intel Data (Code)  
- Generate IP Summary (Code)  
- IP summary display (HTML)  

**Node Details:**

- **Process Intel Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Identifies and separates VirusTotal, AlienVault, and metadata from merged data.  
  - *Configuration:* Iterates over items, assigns json fields to dedicated variables.  
  - *Input:* Merged data from previous block.  
  - *Output:* JSON object containing `virustotal`, `alienvault`, `description`, and `ip_address`.  
  - *Failures:* Missing or malformed data.  
  - *Notes:* Prepares data structure for summary generation.

- **Generate IP Summary**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts relevant fields from VirusTotal and AlienVault data, computes suspiciousness, and formats tags and pulse info for display.  
  - *Configuration:*  
    - Extracts VT tags as comma-separated string and HTML spans.  
    - Calculates malicious and suspicious counts.  
    - Extracts AlienVault pulse info and WHOIS links.  
    - Determines overall status (`Safe` or `Suspicious`).  
    - Adds generation timestamp in Asia/Kolkata timezone.  
  - *Input:* Output from Process Intel Data node.  
  - *Output:* JSON summary with nested VirusTotal and AlienVault objects, status, and timestamp.  
  - *Failures:* Unexpected data schema, missing attributes.  
  - *Notes:* Implements core business logic for threat status.

- **IP summary display**  
  - *Type:* HTML  
  - *Role:* Renders a styled HTML report of the IP threat summary using injected data from the Generate IP Summary node.  
  - *Configuration:*  
    - Custom CSS for dark theme and card layout.  
    - Uses expressions to insert dynamic data such as IP, reputation, WHOIS, tags, and timestamps.  
    - Conditional CSS classes based on threat status.  
    - Generates tag badges for VirusTotal tags and AlienVault pulse names.  
  - *Input:* JSON summary from Generate IP Summary.  
  - *Output:* HTML content for email or display.  
  - *Failures:* Expression errors, missing data causing empty fields.  
  - *Notes:* Designed for SOC analyst consumption, included in email body.

---

#### 2.3 Alert Routing & Analyst Notification

**Overview:**  
Based on threat status, routes suspicious IP alerts to the SOC team via Slack and ServiceNow incident creation, and sends an email with the full HTML summary regardless of status.

**Nodes Involved:**  
- Filter Suspicious IPs (Switch)  
- Create IP Incident (ServiceNow)  
- Slack IP Alert (Slack)  
- Gmail (Email)

**Node Details:**

- **Filter Suspicious IPs**  
  - *Type:* Switch  
  - *Role:* Checks if the IP status is "Suspicious" to determine alert routing path.  
  - *Configuration:* Condition: `{{$json.summary.Status}} == 'Suspicious'`  
  - *Input:* Summary JSON from Generate IP Summary.  
  - *Output:* True branch: suspicious IPs; False branch: no further action.  
  - *Failures:* Expression errors or missing status field.  
  - *Notes:* Critical for controlling alert escalation.

- **Create IP Incident**  
  - *Type:* ServiceNow (Create Incident)  
  - *Role:* Creates a new incident ticket for suspicious IPs in ServiceNow.  
  - *Configuration:*  
    - Resource: Incident  
    - Operation: Create  
    - Short description includes IP and description from summary.  
  - *Authentication:* Basic Auth credential for ServiceNow.  
  - *Input:* Suspicious IP summary from switch node.  
  - *Output:* ServiceNow response with incident details.  
  - *Failures:* Authentication errors, API downtime.  
  - *Notes:* Enables SOC case management integration.

- **Slack IP Alert**  
  - *Type:* Slack  
  - *Role:* Sends alert messages to a Slack channel for suspicious IPs.  
  - *Configuration:*  
    - OAuth2 authentication.  
    - Channel ID hardcoded to specific SOC channel.  
    - Message includes IP, status, and event description with emoji indicator.  
  - *Input:* Suspicious IP summary from switch node.  
  - *Output:* Slack message delivery confirmation.  
  - *Failures:* OAuth token expiration, channel access issues.  
  - *Notes:* Provides real-time alerting to SOC team.

- **Gmail**  
  - *Type:* Gmail (Send Email)  
  - *Role:* Sends an HTML email report with the full IP threat summary to SOC email inbox.  
  - *Configuration:*  
    - Recipient: rajneesh@haxsecurity.com  
    - Subject: "[New Alert] IP Reputation Check Summary"  
    - Message body: Injected HTML from IP summary display node.  
  - *Authentication:* Gmail OAuth2 credential.  
  - *Input:* HTML formatted summary from IP summary display node.  
  - *Output:* Email sent confirmation.  
  - *Failures:* OAuth token expiration, SMTP issues, invalid recipient.  
  - *Notes:* Email always sent regardless of suspiciousness.

---

### 3. Summary Table

| Node Name                   | Node Type       | Functional Role                                | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                     |
|-----------------------------|-----------------|-----------------------------------------------|-------------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Splunk Alert                | Webhook         | Entry point - receives Splunk alerts          | -                                   | Extract IOCs                       | 📥 Alert Ingestion & IOC Extraction - Receives alert, extracts IP and reason, sends to VT, AV |
| Extract IOCs                | Code            | Extracts IP and description from alert        | Splunk Alert                        | VirusTotal IP reputation check, AlienVault Lookup, Merge Threat Data | 📥 Alert Ingestion & IOC Extraction - Receives alert, extracts IP and reason, sends to VT, AV |
| VirusTotal IP reputation check | HTTP Request    | Queries VirusTotal API for IP reputation      | Extract IOCs                       | Merge Threat Data                  | 📥 Alert Ingestion & IOC Extraction - Receives alert, extracts IP and reason, sends to VT, AV |
| AlienVault Lookup           | HTTP Request    | Queries AlienVault OTX for IP info             | Extract IOCs                       | Merge Threat Data                  | 📥 Alert Ingestion & IOC Extraction - Receives alert, extracts IP and reason, sends to VT, AV |
| Merge Threat Data           | Merge           | Combines VirusTotal, AlienVault, and IOC data | VirusTotal IP reputation check, AlienVault Lookup, Extract IOCs | Process Intel Data                | 🧪 Threat Enrichment & Summary Generation - Queries VT & AV, merges data, creates summary    |
| Process Intel Data          | Code            | Separates and organizes threat intel data     | Merge Threat Data                  | Generate IP Summary               | 🧪 Threat Enrichment & Summary Generation - Queries VT & AV, merges data, creates summary    |
| Generate IP Summary         | Code            | Builds summarized threat report and status    | Process Intel Data                 | IP summary display, Filter Suspicious IPs | 🧪 Threat Enrichment & Summary Generation - Queries VT & AV, merges data, creates summary    |
| IP summary display          | HTML            | Generates styled HTML summary for email       | Generate IP Summary                | Gmail                           | 🚨 Alert Routing & Analyst Notification - Sends Slack alert, creates incident, emails report |
| Filter Suspicious IPs       | Switch          | Routes suspicious IPs for alerting             | Generate IP Summary                | Create IP Incident, Slack IP Alert | 🚨 Alert Routing & Analyst Notification - Sends Slack alert, creates incident, emails report |
| Create IP Incident          | ServiceNow      | Creates incident ticket for suspicious IP     | Filter Suspicious IPs              | -                               | 🚨 Alert Routing & Analyst Notification - Sends Slack alert, creates incident, emails report |
| Slack IP Alert              | Slack           | Sends Slack alert for suspicious IP           | Filter Suspicious IPs              | -                               | 🚨 Alert Routing & Analyst Notification - Sends Slack alert, creates incident, emails report |
| Gmail                      | Gmail           | Emails HTML summary report to SOC inbox       | IP summary display                | -                               | 🚨 Alert Routing & Analyst Notification - Sends Slack alert, creates incident, emails report |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Splunk Alert"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., e645d98e-f80c-47e5-b96e-762c96f3db76)  
   - Purpose: Receive Splunk security alert JSON payload.

2. **Create Code Node: "Extract IOCs"**  
   - Type: Code (JavaScript)  
   - Code: Extract `src_ip` and `reason` from incoming JSON:  
     ```js
     const body = $input.first().json.body;
     return [{
       json: {
         ip_address: body?.result?.src_ip || 'No IP found',
         description: body?.result?.reason || 'No reason found'
       }
     }];
     ```  
   - Connect input from "Splunk Alert".

3. **Create HTTP Request Node: "VirusTotal IP reputation check"**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://www.virustotal.com/api/v3/ip_addresses/{{ $json.ip_address }}`  
   - Authentication: Use VirusTotal API credentials (predefined)  
   - Connect input from "Extract IOCs".

4. **Create HTTP Request Node: "AlienVault Lookup"**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://otx.alienvault.com/api/v1/indicators/IPv4/{{ $json.ip_address }}`  
   - Authentication: AlienVault API credentials (predefined)  
   - On Error: Continue execution to avoid failure on API errors  
   - Connect input from "Extract IOCs".

5. **Create Merge Node: "Merge Threat Data"**  
   - Type: Merge  
   - Number of Inputs: 3 (VirusTotal, AlienVault, Extract IOCs)  
   - Mode: Merge by index  
   - Connect inputs from:  
     - "VirusTotal IP reputation check" (input 1)  
     - "AlienVault Lookup" (input 2)  
     - "Extract IOCs" (input 3)

6. **Create Code Node: "Process Intel Data"**  
   - Type: Code (JavaScript)  
   - Code: Identify VirusTotal, AlienVault, and metadata JSON objects and return structured data.  
   - Connect input from "Merge Threat Data".

7. **Create Code Node: "Generate IP Summary"**  
   - Type: Code (JavaScript)  
   - Code: Extract attributes, compute threat status, format tags and pulse names, and timestamp summary.  
   - Connect input from "Process Intel Data".

8. **Create HTML Node: "IP summary display"**  
   - Type: HTML  
   - Paste provided styled HTML with embedded expressions using `{{ $json.summary... }}` for dynamic content.  
   - Connect input from "Generate IP Summary".

9. **Create Switch Node: "Filter Suspicious IPs"**  
   - Type: Switch  
   - Condition: If `{{$json.summary.Status}}` equals `"Suspicious"`  
   - Connect input from "Generate IP Summary".

10. **Create ServiceNow Node: "Create IP Incident"**  
    - Type: ServiceNow  
    - Operation: Create Incident  
    - Authentication: Basic Auth to ServiceNow instance  
    - Short description: `"IP: {{ $json.summary.VirusTotal.IP }} \n{{ $json.summary.VirusTotal.Description }}"`  
    - Connect input from "Filter Suspicious IPs" (True branch).

11. **Create Slack Node: "Slack IP Alert"**  
    - Type: Slack  
    - Authentication: OAuth2 with Slack credentials  
    - Channel: SOC monitoring channel (e.g., `C0913JPTZBJ`)  
    - Message:  
      ```
      {{$json.summary.VirusTotal.Status === 'Safe' ? '✅' : '🚨'}} IP {{$json.summary.VirusTotal.IP}}
      Status: *{{$json.summary.Status}}*
      Event: {{$json.summary.VirusTotal.Description}}
      ```  
    - Connect input from "Filter Suspicious IPs" (True branch).

12. **Create Gmail Node: "Gmail"**  
    - Type: Gmail (Send Email)  
    - Authentication: Gmail OAuth2 credentials  
    - Send To: `rajneesh@haxsecurity.com`  
    - Subject: `[New Alert] IP Reputation Check Summary`  
    - Message: Use `{{$json.html}}` from the HTML node output  
    - Connect input from "IP summary display".

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Security Operations Centers (SOC) can integrate this workflow with Splunk alerting and ServiceNow incident management. | Workflow designed for SOC automation and incident response.   |
| Custom styled HTML summary provides a dark-themed, analyst-friendly report.                      | HTML node styling within the workflow.                         |
| Slack and Gmail credentials require prior OAuth2 setup with appropriate permissions.            | Credential configuration in n8n UI.                            |
| VirusTotal and AlienVault API accounts must be created and API keys generated for authentication. | VirusTotal: https://www.virustotal.com/gui/join              |
| AlienVault OTX API documentation: https://otx.alienvault.com/api |  
| ServiceNow Basic Auth credentials require valid user with incident creation permissions.         | Setup in ServiceNow administration portal.                     |

---

**Disclaimer:** This document is generated exclusively from an n8n automated workflow export. It adheres strictly to content policies and contains no illegal or protected data. All data processed is legal and publicly accessible.