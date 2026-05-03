Phishing Analysis - URLScan.io and VirusTotal

https://n8nworkflows.xyz/workflows/phishing-analysis---urlscan-io-and-virustotal-1992


# Phishing Analysis - URLScan.io and VirusTotal

### 1. Workflow Overview

This n8n workflow automates the phishing analysis process by scanning URLs extracted from Microsoft Outlook emails for potential threats using URLScan.io and VirusTotal. It targets cybersecurity teams aiming to monitor incoming emails for indicators of compromise (IOCs), specifically suspicious URLs, and report findings through Slack.

Logical blocks:

- **1.1 Trigger and Email Retrieval**: Initiates workflow execution (manual or scheduled) and fetches unread emails from Outlook.
- **1.2 Email Processing and IOC Extraction**: Marks emails as read to prevent reprocessing, splits emails into batches, and extracts URLs as potential IOCs.
- **1.3 IOC Filtering**: Filters out emails without URLs.
- **1.4 Parallel Phishing URL Analysis**: Concurrently scans each URL with URLScan.io and VirusTotal.
- **1.5 URLScan.io Result Handling**: Handles URLScan.io scanning, manages errors by waiting and retrying, then retrieves scan reports.
- **1.6 VirusTotal Result Handling**: Initiates VirusTotal scans and retrieves reports.
- **1.7 Results Merging and Filtering**: Merges URLScan.io and VirusTotal reports and filters out empty results.
- **1.8 Reporting**: Sends a summarized analysis report to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Email Retrieval

**Overview:**  
Starts the workflow either manually or on a schedule, then fetches unread emails from Outlook for analysis.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Schedule Trigger  
- Get all unread messages (Microsoft Outlook node)

**Node Details:**  

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Allows manual workflow execution  
  - Config: Default (no parameters)  
  - Inputs: None  
  - Outputs: Connected to "Get all unread messages"  
  - Failures: None expected  
  - Notes: Enables on-demand phishing analysis.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow based on schedule  
  - Config: Default interval (can be customized)  
  - Inputs: None  
  - Outputs: Connected to "Get all unread messages"  
  - Failures: None expected  
  - Notes: Enables automated daily or recurring scans.

- **Get all unread messages**  
  - Type: Microsoft Outlook  
  - Role: Fetches unread emails from Outlook inbox  
  - Config: Operation = getAll; Filter: "isRead eq false" (only unread emails)  
  - Inputs: Trigger nodes output  
  - Outputs: Connects to "Mark as read"  
  - Failures: Possible authentication errors, API limits, or connectivity issues  
  - Credentials: Microsoft Outlook OAuth2  
  - Notes: Current config correctly fetches unread emails, though description notes a past misconfiguration retrieving read emails.

---

#### 1.2 Email Processing and IOC Extraction

**Overview:**  
Marks emails as read, splits email batch for individual processing, and extracts URLs from email bodies as potential IOCs.

**Nodes Involved:**  
- Mark as read (Microsoft Outlook)  
- Split In Batches  
- Find indicators of compromise (Python Code)  
- Has URL? (If node)

**Node Details:**  

- **Mark as read**  
  - Type: Microsoft Outlook  
  - Role: Updates message status to read to avoid duplicate processing  
  - Config: Operation = update; Set isRead = true; messageId dynamically from email  
  - Inputs: "Get all unread messages" output  
  - Outputs: Connects to "Split In Batches"  
  - Failures: Auth errors, invalid messageId, API limits  
  - Credentials: Microsoft Outlook OAuth2  

- **Split In Batches**  
  - Type: Split In Batches  
  - Role: Processes emails one-by-one sequentially  
  - Config: Batch size = 1 (process one email per iteration)  
  - Inputs: "Mark as read" output  
  - Outputs: Two outputs — main flow to "Find indicators of compromise" and alternative to "Not empty?" for filtering  
  - Failures: None expected  
  - Notes: Enables iterative, controlled processing for large batches.

- **Find indicators of compromise**  
  - Type: Code (Python)  
  - Role: Extracts URLs (IOCs) from email body content using ioc-finder library  
  - Config: Installs `ioc-finder` if missing, parses `body.content` from email JSON, returns list of URLs found  
  - Inputs: Single email from "Split In Batches"  
  - Outputs: List of objects with `domain` key for each URL found  
  - Failures: Python errors, missing or malformed email body, empty URLs  
  - Notes: Critical node for IOC detection.

- **Has URL?**  
  - Type: If Node  
  - Role: Checks if extracted URL (`domain`) is not empty  
  - Config: Condition: `domain` is not empty string  
  - Inputs: "Find indicators of compromise" output  
  - Outputs: If true, proceeds to scanning; else, loops back to process next email  
  - Failures: None expected.

---

#### 1.3 IOC Filtering

**Overview:**  
Filters out emails without URLs to avoid unnecessary scanning.

**Nodes Involved:**  
- Has URL?

**Node Details:**  
(Described above; no further nodes involved)

---

#### 1.4 Parallel Phishing URL Analysis

**Overview:**  
Scans each identified URL with URLScan.io and VirusTotal concurrently to assess phishing threat.

**Nodes Involved:**  
- URLScan: Scan URL (URLScan.io node)  
- VirusTotal: Scan URL (HTTP Request)

**Node Details:**  

- **URLScan: Scan URL**  
  - Type: URLScan.io node  
  - Role: Submits URL for scanning on URLScan.io service  
  - Config: URL dynamically from `domain`; continues on failure to prevent workflow halt  
  - Inputs: "Has URL?" true output  
  - Outputs: Connects to "No error?" node for error handling  
  - Credentials: URLScan.io API key required  
  - Failures: API key errors, network issues, invalid URL  
  - Notes: Uses `continueOnFail` to allow workflow to proceed on errors.

- **VirusTotal: Scan URL**  
  - Type: HTTP Request  
  - Role: Sends URL to VirusTotal API for scanning  
  - Config: POST to `https://www.virustotal.com/api/v3/urls`; URL parameter from `domain`; uses VirusTotal API credentials  
  - Inputs: "Has URL?" true output  
  - Outputs: Connects to "VirusTotal: Get report"  
  - Credentials: VirusTotal API key required  
  - Failures: Auth errors, rate limiting, invalid URL.

---

#### 1.5 URLScan.io Result Handling

**Overview:**  
Checks for errors from URLScan submission, waits if necessary, and retrieves scan results.

**Nodes Involved:**  
- No error? (If node)  
- Wait 1 Minute (Wait node)  
- URLScan: Get report (URLScan.io node)  
- Merge Reports (Merge node)

**Node Details:**  

- **No error?**  
  - Type: If Node  
  - Role: Verifies absence of errors in URLScan submission response  
  - Config: Checks if `error` string is empty; if not empty, triggers wait & retry path  
  - Inputs: "URLScan: Scan URL" output  
  - Outputs: True to "Merge Reports"; False to "Wait 1 Minute"  
  - Failures: None expected.

- **Wait 1 Minute**  
  - Type: Wait  
  - Role: Pauses workflow to allow URLScan.io to complete scanning  
  - Config: Wait for 60 seconds  
  - Inputs: "No error?" false output  
  - Outputs: Connects to "URLScan: Get report"  
  - Failures: None expected.

- **URLScan: Get report**  
  - Type: URLScan.io node  
  - Role: Retrieves scan report from URLScan.io using `scanId`  
  - Config: `scanId` dynamically from previous URLScan submission result  
  - Inputs: "Wait 1 Minute" output  
  - Outputs: Connects to "Merge Reports"  
  - Credentials: URLScan.io API key  
  - Failures: Invalid scanId, API errors.

- **Merge Reports**  
  - Type: Merge  
  - Role: Combines URLScan.io and VirusTotal reports by position for consolidated view  
  - Config: Combination mode = mergeByPosition  
  - Inputs: Two inputs — from "No error?" true output (URLScan result) and "VirusTotal: Get report" output  
  - Outputs: Connects to "Split In Batches" (loop continuation)  
  - Failures: Mismatched input counts, data format issues.

---

#### 1.6 VirusTotal Result Handling

**Overview:**  
Retrieves VirusTotal scan report for the submitted URL.

**Nodes Involved:**  
- VirusTotal: Get report (HTTP Request)  
- Merge Reports (Merge node)

**Node Details:**  

- **VirusTotal: Get report**  
  - Type: HTTP Request  
  - Role: Retrieves analysis results from VirusTotal API for the URL scan  
  - Config: GET to `https://www.virustotal.com/api/v3/analyses/{{ $json.data.id }}`; uses VirusTotal API credentials  
  - Inputs: "VirusTotal: Scan URL" output  
  - Outputs: Connects to second input of "Merge Reports"  
  - Failures: API errors, invalid scan ID, rate limiting.

- **Merge Reports**  
  - (Described above; merges URLScan and VirusTotal reports)

---

#### 1.7 Results Merging and Filtering

**Overview:**  
Filters out empty or incomplete results before reporting.

**Nodes Involved:**  
- Not empty? (Filter node)

**Node Details:**  

- **Not empty?**  
  - Type: Filter  
  - Role: Checks that `data` field in merged report is not empty, ensuring valid analysis data before reporting  
  - Config: Condition: `data` is not empty string  
  - Inputs: From "Split In Batches" (after merging)  
  - Outputs: Connects to "sends slack message"  
  - Failures: None expected.

---

#### 1.8 Reporting

**Overview:**  
Sends a detailed phishing analysis summary to a Slack channel for cybersecurity team awareness.

**Nodes Involved:**  
- sends slack message (Slack node)

**Node Details:**  

- **sends slack message**  
  - Type: Slack  
  - Role: Posts formatted message summarizing email and URL scan results to Slack channel  
  - Config:  
    - Channel: "test-giulio-public"  
    - Message contains: Email Subject, Sender, Date, URLScan report URL (if available), VirusTotal report URL, and a verdict summary (malicious + suspicious counts over total checks)  
  - Inputs: From "Not empty?" filter node  
  - Outputs: None (end node)  
  - Credentials: Slack Bot Token (OAuth)  
  - Failures: Slack auth errors, channel not found, rate limits.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                             | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                      |
|----------------------------|--------------------|---------------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger     | Workflow manual start trigger                | None                             | Get all unread messages           | Enables manual execution of the workflow.                                                                       |
| Schedule Trigger           | Schedule Trigger   | Scheduled workflow start trigger             | None                             | Get all unread messages           | Allows automatic periodic execution.                                                                             |
| Get all unread messages    | Microsoft Outlook  | Retrieve unread emails from Outlook inbox    | When clicking "Execute Workflow", Schedule Trigger | Mark as read                     | Fetches unread emails for analysis.                                                                              |
| Mark as read              | Microsoft Outlook  | Marks emails as read after retrieval          | Get all unread messages           | Split In Batches                 | Prevents reprocessing of emails; keeps inbox organized.                                                         |
| Split In Batches           | Split In Batches   | Processes emails one by one                    | Mark as read                     | Find indicators of compromise, Not empty? | Enables controlled iterative processing of emails.                                                               |
| Find indicators of compromise | Code (Python)      | Extracts URLs (IOCs) from email content       | Split In Batches                  | Has URL?                        | Uses ioc-finder library to detect URLs in email body.                                                           |
| Has URL?                   | If                 | Checks if URLs are found in email              | Find indicators of compromise     | URLScan: Scan URL, VirusTotal: Scan URL, Split In Batches (else) | Filters out emails without URLs to optimize processing.                                                         |
| URLScan: Scan URL          | URLScan.io         | Submits URL scan request to URLScan.io         | Has URL? (true path)              | No error?                       | Continues on failure to avoid halting workflow.                                                                  |
| VirusTotal: Scan URL       | HTTP Request       | Sends URL to VirusTotal for scanning           | Has URL? (true path)              | VirusTotal: Get report          | Initiates phishing scan in VirusTotal service.                                                                   |
| No error?                  | If                 | Checks for errors in URLScan submission response | URLScan: Scan URL                 | Merge Reports, Wait 1 Minute     | Determines next step based on presence of error in URLScan submission.                                          |
| Wait 1 Minute              | Wait               | Waits 60 seconds for URLScan.io to finish scan | No error? (false)                 | URLScan: Get report             | Ensures scan results are ready before retrieval.                                                                 |
| URLScan: Get report        | URLScan.io         | Retrieves URLScan.io scan report                | Wait 1 Minute                    | Merge Reports                   | Gets detailed URL scan results.                                                                                   |
| VirusTotal: Get report     | HTTP Request       | Retrieves VirusTotal scan report                | VirusTotal: Scan URL             | Merge Reports                   | Obtains detailed analysis from VirusTotal.                                                                        |
| Merge Reports              | Merge              | Combines URLScan.io and VirusTotal results      | URLScan: Get report, VirusTotal: Get report, No error? (true) | Split In Batches               | Consolidates results for comprehensive threat analysis.                                                          |
| Not empty?                 | Filter             | Filters out empty or incomplete scan results    | Split In Batches                 | sends slack message             | Prevents reporting on incomplete or empty data.                                                                   |
| sends slack message        | Slack              | Sends summarized phishing analysis to Slack    | Not empty?                      | None                           | Reports findings to cybersecurity team for action.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add **Manual Trigger** named "When clicking \"Execute Workflow\"" with default settings.  
   - Add **Schedule Trigger** named "Schedule Trigger" with your preferred schedule (e.g., daily at midnight).

2. **Configure Microsoft Outlook Node to Fetch Emails**  
   - Add Microsoft Outlook node named "Get all unread messages".  
   - Set operation to `getAll`.  
   - Under additional fields, apply filter: `isRead eq false` to fetch only unread emails.  
   - Connect outputs of both trigger nodes to this node.

3. **Mark Emails as Read**  
   - Add another Microsoft Outlook node named "Mark as read".  
   - Set operation to `update`.  
   - Set `messageId` parameter to `={{ $json.id }}` dynamically.  
   - Under updateFields, set `isRead` to `true`.  
   - Connect output of "Get all unread messages" to this node.

4. **Split Emails into Individual Items**  
   - Add Split In Batches node named "Split In Batches".  
   - Set batch size to 1 (to process emails one at a time).  
   - Connect output of "Mark as read" to this node.

5. **Extract URLs (IOCs) from Email Body**  
   - Add a Code node named "Find indicators of compromise".  
   - Set language to Python.  
   - Paste the following code:  
     ```python
     try:
       from ioc_finder import find_iocs
     except ImportError:
       import micropip
       await micropip.install("ioc-finder")
       from ioc_finder import find_iocs

     text = _input.first().json['body']['content']
     iocs = find_iocs(text)

     return [{"json": {"domain": item}} for item in iocs["urls"]]
     ```  
   - Connect output of "Split In Batches" to this node.

6. **Filter Emails with URLs**  
   - Add If node named "Has URL?".  
   - Set condition to check if `domain` is not empty string (`isNotEmpty`).  
   - Connect output of "Find indicators of compromise" to this node.  
   - True output proceeds to URL scanning nodes, false output connects back to "Split In Batches" to continue looping.

7. **URLScan.io: Submit URL for Scanning**  
   - Add URLScan.io node named "URLScan: Scan URL".  
   - Set operation to `scan`.  
   - Set URL parameter to `={{ $json.domain }}`.  
   - Enable "Continue on Fail" to true.  
   - Connect true output of "Has URL?" to this node.  
   - Provide URLScan.io API credentials.

8. **VirusTotal: Submit URL for Scanning**  
   - Add HTTP Request node named "VirusTotal: Scan URL".  
   - Set method to `POST`.  
   - Set URL to `https://www.virustotal.com/api/v3/urls`.  
   - Add query parameter `url` with value `={{ $json.domain }}`.  
   - Use predefined VirusTotal API credentials.  
   - Connect true output of "Has URL?" also to this node (parallel to URLScan.io submit).

9. **URLScan.io: Check for Errors**  
   - Add If node named "No error?".  
   - Check if `error` string is empty (operation: `isNotEmpty` on `$json.error`).  
   - Connect output of "URLScan: Scan URL" to this node.

10. **On No Error: Proceed to Merge**  
    - Connect true output of "No error?" to next node (Merge Reports).  
    - Connect false output to "Wait 1 Minute".

11. **Wait Before Retrieving URLScan Report**  
    - Add Wait node named "Wait 1 Minute".  
    - Set to wait 60 seconds.  
    - Connect false output of "No error?" to this node.

12. **Retrieve URLScan.io Report**  
    - Add URLScan.io node named "URLScan: Get report".  
    - Set operation to `get`.  
    - Set `scanId` parameter to `={{ $json.scanId }}`.  
    - Connect output of "Wait 1 Minute" to this node.  
    - Provide URLScan.io API credentials.

13. **Retrieve VirusTotal Report**  
    - Add HTTP Request node named "VirusTotal: Get report".  
    - Set method to `GET`.  
    - Set URL to `https://www.virustotal.com/api/v3/analyses/{{ $json.data.id }}`.  
    - Use VirusTotal API credentials.  
    - Connect output of "VirusTotal: Scan URL" to this node.

14. **Merge URLScan.io and VirusTotal Reports**  
    - Add Merge node named "Merge Reports".  
    - Set mode to `combine`, combination mode `mergeByPosition`.  
    - Connect outputs:  
      - From "URLScan: Get report" to input 1  
      - From "VirusTotal: Get report" to input 2  
      - From "No error?" true output to input 1 if applicable (ensures merging in both success and retry cases)

15. **Loop Continuation**  
    - Connect output of "Merge Reports" back to "Split In Batches" to continue processing next email.

16. **Filter Non-Empty Results**  
    - Add Filter node named "Not empty?".  
    - Condition: `data` field is not empty (`isNotEmpty`).  
    - Connect output of "Split In Batches" (second output) to this node.

17. **Send Slack Report**  
    - Add Slack node named "sends slack message".  
    - Configure channel (e.g., "test-giulio-public").  
    - Compose message text using expressions:  
      ```
      =*Email Analysis*

      Subject: {{ $('Microsoft Outlook').item.json["subject"] }}
      From: {{ $('Microsoft Outlook').item.json["sender"]["emailAddress"]["address"] }}
      Date: {{ $('Microsoft Outlook').item.json["sentDateTime"] }}

      Report:

      *URLScan Report URL:* {{ $('urlscan.io').item.json.result ? $('urlscan.io').item.json.result : "N/A" }}
      *Virustotal report:* https://www.virustotal.com/gui/url/{{ $json["data"]["id"].split("-")[1] }}
      *Virustotal Verdict:* {{ $json.data.attributes.stats.malicious + $json.data.attributes.stats.suspicious }} / {{ Object.keys($json.data.attributes.results).length }}
      ```  
    - Connect output of "Not empty?" node to this Slack node.  
    - Provide Slack Bot Token credentials.

18. **Test & Validate**  
    - Execute manually or wait for scheduled run.  
    - Monitor Slack for reports and verify email processing.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow is designed to be adaptable to other mail providers by replacing the Outlook nodes. | Sticky Note at Workflow Overview |
| The ioc-finder Python library is dynamically installed at runtime to extract URLs from email bodies. | Sticky Note describing IOC detection |
| URLScan.io node is configured to continue on failure to prevent workflow halting, with error handling via conditional nodes and wait steps. | Sticky Note about URLScan.io error handling |
| Slack reporting includes direct links to VirusTotal and URLScan.io reports for easy access to detailed threat information. | Sticky Note on final reporting |
| API keys and OAuth credentials for Microsoft Outlook, URLScan.io, VirusTotal, and Slack must be properly configured and tested. | General setup requirement |

---

This detailed reference document provides a complete understanding of the phishing analysis workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.