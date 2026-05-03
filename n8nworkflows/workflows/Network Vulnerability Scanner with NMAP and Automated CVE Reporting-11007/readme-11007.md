Network Vulnerability Scanner with NMAP and Automated CVE Reporting

https://n8nworkflows.xyz/workflows/network-vulnerability-scanner-with-nmap-and-automated-cve-reporting-11007


# Network Vulnerability Scanner with NMAP and Automated CVE Reporting

### 1. Workflow Overview

This workflow automates network vulnerability scanning using Nmap, enriches scan results with CVE data from the National Vulnerability Database (NVD) API, and generates professional, password-protected PDF reports. It targets network security teams and automated security operations use cases, enabling scheduled, manual, or API-driven scans with detailed vulnerability analysis and distribution via Telegram and email.

The workflow is logically organized into the following blocks:

- **1.1 Triggers and Input Reception:** Multiple entry points allow starting scans via webhook, web form, manual trigger, or schedule.
- **1.2 Preparation and Dependency Checks:** Setting scan parameters and verifying/installing required helper tools.
- **1.3 Nmap Scan Execution and Parsing:** Running the Nmap scan with specified parameters and parsing XML results into structured JSON services.
- **1.4 CVE Enrichment:** Querying the NVD CVE API for vulnerabilities related to detected services using CPE identifiers with rate limiting.
- **1.5 Data Aggregation and Report Preparation:** Aggregating enriched data by host, calculating statistics, and preparing a structured report.
- **1.6 Report Generation and Storage:** Creating an HTML report with detailed vulnerability information, saving it, and converting it to a password-protected PDF.
- **1.7 Report Distribution:** Sending the PDF report via Telegram and email to configured recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggers and Input Reception

**Overview:**  
This block provides flexible ways to start the scan workflow: via external webhook API (for automation), web UI form (for user input), manual trigger (for testing), or scheduled execution (cron job).

**Nodes Involved:**  
- Webhook  
- On form submission (formTrigger)  
- Manual Trigger  
- Schedule Trigger

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Receives POST requests at path `/vuln-scan` with JSON payload containing target network.  
  - Config: HTTP POST method, immediate response "Process started!" to caller.  
  - Inputs: External HTTP request  
  - Outputs: Passes JSON payload to parameter settings node.  
  - Failure Types: Incorrect payload format, HTTP errors, but minimal as it responds immediately.

- **On form submission**  
  - Type: Form Trigger  
  - Role: Web UI form allowing users to input network targets interactively.  
  - Config: Form field "network" required; path `/target`.  
  - Outputs: Passes form data to parameter settings.  
  - Failure Types: Missing required field; user input validation needed externally.

- **Manual Trigger**  
  - Type: Manual Trigger  
  - Role: Allows manual workflow start for testing or ad-hoc runs.  
  - Outputs: Triggers a preset target scan.  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Executes scan automatically daily at 01:00.  
  - Outputs: Triggers a preset target scan.

---

#### 2.2 Preparation and Dependency Checks

**Overview:**  
Sets initial scan parameters including network target, timestamp, and report password. Checks if `nmap-helper` tool is installed to convert Nmap XML to JSON; if missing, installs it. Sets optional Nmap scan parameters.

**Nodes Involved:**  
- Pre-Set-Target  
- 1. Set Parameters  
- Need to Add Helper? (If)  
- Add NMAP-HELPER  
- Optional Params Setter

**Node Details:**

- **Pre-Set-Target**  
  - Type: Set  
  - Role: Sets default network target for manual or scheduled scans (default: `scanme.nmap.org`).  
  - Outputs: Passes target to parameter setting node.  

- **1. Set Parameters**  
  - Type: Set  
  - Role: Aggregates all scan parameters: network target (from input JSON), current timestamp (formatted `yyyyMMdd_HHmmss`), and report password (`Almafa123456` by default).  
  - Inputs: From triggers or Pre-Set-Target.  
  - Outputs: Passes parameters to helper check node.  
  - Failure: Missing or malformed network input may cause scan failure.

- **Need to Add Helper?**  
  - Type: If  
  - Role: Checks if `nmap-helper` binary exists at `/tmp/nmap-helper/target/release/nmap-helper`.  
  - True Output: Runs Add NMAP-HELPER node.  
  - False Output: Skips installation, proceeds to Optional Params Setter.  
  - Failure: File system access issues can cause false negatives.

- **Add NMAP-HELPER**  
  - Type: Execute Command  
  - Role: Installs required dependencies (`git`, `make`, `rust`, `cargo`), clones `nmap-helper` GitHub repo, builds and installs it.  
  - Outputs: Proceeds to Optional Params Setter.  
  - Failure: Network connectivity, build errors, or missing apk packages can cause failure.

- **Optional Params Setter**  
  - Type: Set  
  - Role: Sets additional Nmap scan parameters such as timing (`-T4`), parallelism, and retries.  
  - Outputs: Passes parameters to Nmap scan execution.

---

#### 2.3 Nmap Scan Execution and Parsing

**Overview:**  
Runs the Nmap scan command with full port scan, version detection, and default scripts. Converts XML output to JSON using the helper tool and parses the JSON to extract service details per host.

**Nodes Involved:**  
- 2. Execute Nmap Scan  
- 3. Parse NMAP JSON to Services  
- 4. Save Scan Data to File

**Node Details:**

- **2. Execute Nmap Scan**  
  - Type: Execute Command  
  - Role: Runs shell commands to install dependencies, execute Nmap with parameters, correct XML case sensitivity, convert XML to JSON via `nmap-helper`, and outputs JSON scan data.  
  - Key Config:  
    - Command includes Nmap options: `-sV` (service/version), `-sC` (scripts), `-p-` (all ports), `-sS` (SYN scan), timing and parallelism from parameters.  
    - Output file names include timestamp for uniqueness.  
  - Outputs: JSON scan results to next node.  
  - Failure: Command errors, Nmap installation missing, permission issues, or malformed parameters.

- **3. Parse NMAP JSON to Services**  
  - Type: Code  
  - Role: Parses JSON output, extracts all open services per host with details: IP, port, protocol, service name, product, version, CPE normalized to `cpe:2.3:` format if available.  
  - Outputs: Array of service objects as JSON and binary file.  
  - Failure: JSON parsing errors, unexpected Nmap output format.

- **4. Save Scan Data to File**  
  - Type: Read/Write File  
  - Role: Saves parsed scan JSON data to `/tmp/scan_data_{timestamp}.json` for persistence.  
  - Inputs: Parsed services from previous node.  
  - Outputs: Passes data to CVE enrichment.  
  - Failure: File system permission or disk space issues.

---

#### 2.4 CVE Enrichment

**Overview:**  
For each detected service, queries the NVD CVE API using the CPE identifier to retrieve up to 10 vulnerabilities. Handles missing CPEs, rate limits API calls (1 second delay), and collects CVE IDs, summaries, CVSS scores, severities, publish dates, and references.

**Nodes Involved:**  
- 5. CVE Enrichment Loop  
- 6. Save Enriched Data

**Node Details:**

- **5. CVE Enrichment Loop**  
  - Type: Code  
  - Role: Iterates over each service; if CPE present, queries NVD API endpoint `https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=...&resultsPerPage=10&isVulnerable`.  
  - Handles exceptions by adding error info to output.  
  - Applies a 1-second delay between requests to respect API rate limits.  
  - Outputs enriched data including vulnerability details and metadata.  
  - Failure: HTTP errors, API downtime, rate limit exceeded, malformed CPEs, or JSON parsing errors.

- **6. Save Enriched Data**  
  - Type: Read/Write File  
  - Role: Saves enriched CVE data JSON to `/tmp/enriched_data_{timestamp}.json`.  
  - Inputs: Enriched CVE data from code node.  
  - Outputs: Passes to aggregation node.  
  - Failure: File write errors.

---

#### 2.5 Data Aggregation and Report Preparation

**Overview:**  
Aggregates all enriched vulnerability data, groups findings by IP address, organizes services per host, summarizes vulnerability counts by severity, and prepares the final report JSON structure including metadata, statistics, and executive summary.

**Nodes Involved:**  
- 7. Aggregate All Data  
- 8. Prepare Report Structure

**Node Details:**

- **7. Aggregate All Data**  
  - Type: Aggregate  
  - Role: Collects all enriched data items into a single JSON structure including original services.  
  - Inputs: Enriched data saved file.  
  - Outputs: Passes aggregated data to report preparation code node.  
  - Failure: Data aggregation errors or empty input.

- **8. Prepare Report Structure**  
  - Type: Code  
  - Role:  
    - Extracts enriched vulnerability data from nested arrays.  
    - Groups by host IP, enumerates services and associated vulnerabilities.  
    - Calculates total counts of hosts, services, vulnerabilities, and breakdown by severity (critical, high, medium, low, info).  
    - Constructs executive summary text and full report JSON with metadata, statistics, and detailed host data.  
  - Outputs: Final report JSON to HTML report generation.  
  - Failure: Unexpected data structure, missing fields, or empty data sets.

---

#### 2.6 Report Generation and Storage

**Overview:**  
Generates a professional, styled HTML report from the aggregated data, then saves the HTML and converts it to a password-protected PDF using PrinceXML. Finally, the PDF is read back for distribution.

**Nodes Involved:**  
- 9. Generate HTML Report  
- 10. Save Report to File  
- 11. Read Report for Output  
- 12. Convert the Report to PDF from HTML  
- 13. Read Report PDF for Output

**Node Details:**

- **9. Generate HTML Report**  
  - Type: Code  
  - Role:  
    - Builds a comprehensive HTML report including: header, metadata, executive summary, statistics, severity breakdown, per-host service tables, vulnerability cards with CVE details and references, and recommendations.  
    - Uses inline CSS for clean, modern styling.  
    - Includes severity badges with color coding.  
  - Outputs: HTML string and binary file buffer.  
  - Failure: Code errors, malformed input data, or large data causing performance issues.

- **10. Save Report to File**  
  - Type: Read/Write File  
  - Role: Saves generated HTML report to `/tmp/vulnerability_report_{timestamp}.html`.  
  - Outputs: Passes file path to next read node.  
  - Failure: File system issues.

- **11. Read Report for Output**  
  - Type: Read/Write File  
  - Role: Reads saved HTML report from disk for conversion.  
  - Outputs: Passes HTML file to PDF conversion.  
  - Failure: File not found or read permissions.

- **12. Convert the Report to PDF from HTML**  
  - Type: Execute Command  
  - Role:  
    - Downloads and installs PrinceXML (PDF rendering tool).  
    - Converts HTML report to PDF with print media settings.  
    - Applies PDF encryption with user password from parameters (`report_password`).  
    - Sets CSS DPI for crisp rendering.  
  - Outputs: PDF file on disk.  
  - Failure: Network download failure, PrinceXML incompatibility, command execution errors.

- **13. Read Report PDF for Output**  
  - Type: Read/Write File  
  - Role: Reads generated PDF report from disk into binary data for distribution.  
  - Outputs: Passes PDF binary to distribution nodes.  
  - Failure: File access or missing PDF file.

---

#### 2.7 Report Distribution

**Overview:**  
Sends the final PDF vulnerability report to configured recipients via Telegram chat and SMTP email.

**Nodes Involved:**  
- 14/a. Send Report in Telegram  
- 14/b. Send Report in Email with SMTP (example POSTFIX)

**Node Details:**

- **14/a. Send Report in Telegram**  
  - Type: Telegram  
  - Role: Sends the PDF report as a document to a specified Telegram chat ID.  
  - Key Config: Chat ID configured in the node, binary data sent from PDF read node.  
  - Credentials: Telegram API token configured in n8n credentials.  
  - Failure: Invalid token, chat ID, network errors.

- **14/b. Send Report in Email with SMTP**  
  - Type: Email Send  
  - Role: Emails the PDF report as attachment with customized subject and body referencing the scanned network.  
  - Key Config: SMTP credentials configured, recipient and sender emails set in parameters.  
  - Failure: SMTP authentication errors, invalid email addresses, attachment size limits.

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                            | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                                        |
|---------------------------------|---------------------|------------------------------------------|--------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Webhook                         | Webhook             | API entry point for scan start           | -                              | 1. Set Parameters                     |                                                                                                                                  |
| On form submission              | Form Trigger        | Web UI form input                        | -                              | 1. Set Parameters                     |                                                                                                                                  |
| Manual Trigger                 | Manual Trigger      | Manual workflow start                     | -                              | Pre-Set-Target                       |                                                                                                                                  |
| Schedule Trigger                | Schedule Trigger    | Scheduled daily scan                      | -                              | Pre-Set-Target                       | Sticky Note - Triggers: Four ways to start scans: webhook API, web form, manual, scheduled.                                       |
| Pre-Set-Target                 | Set                 | Set default network target                | Manual Trigger, Schedule Trigger | 1. Set Parameters                     |                                                                                                                                  |
| 1. Set Parameters              | Set                 | Aggregate scan parameters                  | Webhook, On form submission, Pre-Set-Target | Need to Add Helper?                  | Sticky Note - Overview: Workflow purpose, setup steps, NVD query rate limits, etc.                                               |
| Need to Add Helper?            | If                  | Check for nmap-helper binary              | 1. Set Parameters               | Add NMAP-HELPER, Optional Params Setter | Sticky Note - Dependency: Installs nmap-helper if missing from GitHub.                                                            |
| Add NMAP-HELPER               | Execute Command     | Install nmap-helper tool                   | Need to Add Helper? (true)      | Optional Params Setter                |                                                                                                                                  |
| Optional Params Setter        | Set                 | Set optional Nmap scan parameters          | Need to Add Helper? (false), Add NMAP-HELPER | 2. Execute Nmap Scan               | Sticky Note - Scanning: Full port scan, timing and parallelism config.                                                           |
| 2. Execute Nmap Scan          | Execute Command     | Run Nmap scan and convert XML to JSON      | Optional Params Setter          | 3. Parse NMAP JSON to Services        |                                                                                                                                  |
| 3. Parse NMAP JSON to Services | Code                | Extract open services details from JSON    | 2. Execute Nmap Scan            | 4. Save Scan Data to File             |                                                                                                                                  |
| 4. Save Scan Data to File      | Read/Write File     | Save parsed scan data to file               | 3. Parse NMAP JSON to Services  | 5. CVE Enrichment Loop                |                                                                                                                                  |
| 5. CVE Enrichment Loop         | Code                | Query NVD API for CVE info per service      | 4. Save Scan Data to File       | 6. Save Enriched Data                 | Sticky Note - CVE: Queries NVD API, returns CVE IDs, CVSS scores, rate-limited 1 req/sec.                                         |
| 6. Save Enriched Data          | Read/Write File     | Save CVE-enriched service data              | 5. CVE Enrichment Loop          | 7. Aggregate All Data                 |                                                                                                                                  |
| 7. Aggregate All Data          | Aggregate           | Aggregate all enriched results               | 6. Save Enriched Data           | 8. Prepare Report Structure          |                                                                                                                                  |
| 8. Prepare Report Structure    | Code                | Group data by host, calculate statistics    | 7. Aggregate All Data           | 9. Generate HTML Report              | Sticky Note - Report Gen: Groups by host, stats, HTML report generation.                                                          |
| 9. Generate HTML Report        | Code                | Generate professional styled HTML report    | 8. Prepare Report Structure     | 10. Save Report to File               |                                                                                                                                  |
| 10. Save Report to File        | Read/Write File     | Save HTML report to disk                      | 9. Generate HTML Report         | 11. Read Report for Output            |                                                                                                                                  |
| 11. Read Report for Output     | Read/Write File     | Read saved HTML report                        | 10. Save Report to File         | 12. Convert the Report to PDF from HTML |                                                                                                                                  |
| 12. Convert the Report to PDF from HTML | Execute Command     | Convert HTML to password-protected PDF using PrinceXML | 11. Read Report for Output      | 13. Read Report PDF for Output        | Sticky Note - PDF: Uses PrinceXML to create password-protected PDF.                                                                |
| 13. Read Report PDF for Output | Read/Write File     | Read generated PDF report                      | 12. Convert the Report to PDF   | 14/a. Send Report in Telegram, 14/b. Send Report in Email |                                                                                                                                  |
| 14/a. Send Report in Telegram  | Telegram            | Send PDF report to Telegram chat               | 13. Read Report PDF for Output  | -                                   | Sticky Note - Distribution: Update Telegram credentials and chat ID before use.                                                  |
| 14/b. Send Report in Email with SMTP (example POSTFIX) | Email Send          | Email PDF report via SMTP                        | 13. Read Report PDF for Output  | -                                   | Sticky Note - Distribution: Update SMTP credentials and email addresses before use.                                              |
| Sticky Note (various)          | Sticky Note         | Contextual notes on dependencies, CVE API, PDF, triggers, scanning, distribution | -                              | -                                   | Multiple notes provide useful URLs and instructions, e.g., CPE Search, NVD API, PrinceXML guide.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**

   - Add a **Webhook** node:
     - HTTP Method: POST
     - Path: `vuln-scan`
     - Response Data: "Process started!"
   
   - Add a **Form Trigger** node:
     - Path: `target`
     - Form Field: `network` (string, required)
   
   - Add a **Manual Trigger** node.
   
   - Add a **Schedule Trigger** node:
     - Schedule: Daily at 01:00.

2. **Set Default Target for Manual and Scheduled Runs:**

   - Add a **Set** node named `Pre-Set-Target`:
     - Assign `network` = `scanme.nmap.org`

   - Connect **Manual Trigger** and **Schedule Trigger** to this node.

3. **Aggregate Scan Parameters:**

   - Add a **Set** node named `1. Set Parameters`:
     - Assign `network` from input JSON: `={{ $json.network }}{{ $json.body.network }}`
     - Assign `timestamp` = `={{ $now.format('yyyyMMdd_HHmmss') }}`
     - Assign `report_password` = `"Almafa123456"` (default, editable)
   
   - Connect **Webhook**, **On form submission**, and **Pre-Set-Target** to this node.

4. **Check for nmap-helper Installation:**

   - Add an **If** node named `Need to Add Helper?`:
     - Condition: Check if file `/tmp/nmap-helper/target/release/nmap-helper` exists.
     - True branch: Proceed to installation.
     - False branch: Skip to parameter setting.

5. **Install nmap-helper if Needed:**

   - Add an **Execute Command** node `Add NMAP-HELPER`:
     - Commands:
       ```
       apk add --no-cache git make rust cargo
       cd /tmp
       git clone https://github.com/net-shaper/nmap-helper.git
       cd nmap-helper
       make
       make install
       ```
   
   - Connect **Need to Add Helper?** true branch to this node.

6. **Set Optional Nmap Scan Parameters:**

   - Add a **Set** node `Optional Params Setter`:
     - Assign `setter` = `"-"`
     - Assign `max-retries` = `"max-retries 2"`
     - Assign `min-parallelism` = `"min-parallelism 40"`
     - Assign `max-parallelism` = `"max-parallelism 250"`
     - Assign `timing` = `"-T4"`
   
   - Connect **Add NMAP-HELPER** and **Need to Add Helper?** false branch to this node.

7. **Execute Nmap Scan:**

   - Add an **Execute Command** node `2. Execute Nmap Scan`:
     - Command:
       ```
       installdependencies=$(apk add util-linux-misc ncurses nmap nmap-scripts)
       runnmapscan=$(script -q -c "nmap -sV -sC -p- -sS {{ $json.timing }} {{ $json.setter }}{{ $json.setter }}{{ $json['min-parallelism'] }} {{ $json.setter }}{{ $json.setter }}{{ $json['max-parallelism'] }} {{ $json.setter }}{{ $json.setter }}{{ $json['max-retries'] }} -oX /tmp/nmap_{{ $('1. Set Parameters').item.json.timestamp }}.xml {{ $('1. Set Parameters').item.json.network }}" /dev/null)
       correctxml=$(sed -i 's/type="user"/type="USER"/g' /tmp/nmap_{{ $('1. Set Parameters').item.json.timestamp }}.xml)
       convertxmltojson=$(/tmp/nmap-helper/target/release/nmap-helper convert /tmp/nmap_{{ $('1. Set Parameters').item.json.timestamp }}.xml -o /tmp/nmap_{{ $('1. Set Parameters').item.json.timestamp }}.json)
       tput reset
       cat /tmp/nmap_{{ $('1. Set Parameters').item.json.timestamp }}.json
       ```
   
   - Connect **Optional Params Setter** to this node.

8. **Parse Nmap JSON to Services:**

   - Add a **Code** node `3. Parse NMAP JSON to Services` with the provided JavaScript code that extracts open services, normalizes CPEs, and returns services array as JSON and binary.

   - Connect **2. Execute Nmap Scan** to this node.

9. **Save Parsed Scan Data:**

   - Add a **Read/Write File** node `4. Save Scan Data to File`:
     - Operation: Write
     - File Name: `/tmp/scan_data_{{ $('1. Set Parameters').item.json.timestamp }}.json`
     - Data Property: `=data`
   
   - Connect **3. Parse NMAP JSON to Services** to this node.

10. **Perform CVE Enrichment:**

    - Add a **Code** node `5. CVE Enrichment Loop` with the provided JavaScript code:
      - Iterates services, queries NVD API for each CPE with 1-second delay.
      - Handles missing CPEs and errors gracefully.
      - Outputs enriched CVE data JSON and binary.

    - Connect **4. Save Scan Data to File** to this node.

11. **Save Enriched CVE Data:**

    - Add a **Read/Write File** node `6. Save Enriched Data`:
      - Operation: Write
      - File Name: `/tmp/enriched_data_{{ $('1. Set Parameters').item.json.timestamp }}.json`
      - Data Property: `=data`

    - Connect **5. CVE Enrichment Loop** to this node.

12. **Aggregate Enriched Data:**

    - Add an **Aggregate** node `7. Aggregate All Data`:
      - Aggregate All Item Data
      - Include fields: `enriched, services`

    - Connect **6. Save Enriched Data** to this node.

13. **Prepare Report Structure:**

    - Add a **Code** node `8. Prepare Report Structure` with the provided JavaScript:
      - Groups enriched data by host IP
      - Calculates statistics and builds report JSON including metadata and executive summary.

    - Connect **7. Aggregate All Data** to this node.

14. **Generate HTML Report:**

    - Add a **Code** node `9. Generate HTML Report`:
      - Uses provided JavaScript to create styled HTML report with severity badges, tables, and detailed vulnerability sections.

    - Connect **8. Prepare Report Structure** to this node.

15. **Save HTML Report:**

    - Add a **Read/Write File** node `10. Save Report to File`:
      - Operation: Write
      - File Name: `/tmp/vulnerability_report_{{ $('1. Set Parameters').item.json.timestamp }}.html`
      - Data Property: `=data`

    - Connect **9. Generate HTML Report** to this node.

16. **Read HTML Report for PDF Conversion:**

    - Add a **Read/Write File** node `11. Read Report for Output`:
      - Operation: Read
      - File Selector: `/tmp/vulnerability_report_{{ $('1. Set Parameters').item.json.timestamp }}.html`

    - Connect **10. Save Report to File** to this node.

17. **Convert HTML to Password-Protected PDF:**

    - Add an **Execute Command** node `12. Convert the Report to PDF from HTML`:
      - Commands:
        ```
        apk add wget
        cd /tmp
        wget https://www.princexml.com/download/prince-16.1-alpine3.20-x86_64.tar.gz
        tar -xf prince-16.1-alpine3.20-x86_64.tar.gz

        /tmp/prince-16.1-alpine3.20-x86_64/lib/prince/bin/prince /tmp/vulnerability_report_{{ $('1. Set Parameters').item.json.timestamp }}.html -o /tmp/vulnerability_report_{{ $('1. Set Parameters').item.json.timestamp }}.pdf --media=print --encrypt --user-password={{ $('1. Set Parameters').item.json.report_password }} --css-dpi=128
        ```
      - Converts HTML report to PDF with encryption.

    - Connect **11. Read Report for Output** to this node.

18. **Read PDF Report:**

    - Add a **Read/Write File** node `13. Read Report PDF for Output`:
      - Operation: Read
      - File Selector: `/tmp/vulnerability_report_{{ $('1. Set Parameters').item.json.timestamp }}.pdf`
      - Data Property: `=vulnerability_report`

    - Connect **12. Convert the Report to PDF from HTML** to this node.

19. **Send Report via Telegram:**

    - Add a **Telegram** node `14/a. Send Report in Telegram`:
      - Operation: Send Document
      - Chat ID: Your target Telegram chat ID (e.g., `-123456789012`)
      - Binary Property Name: `=vulnerability_report`
      - Requires Telegram API credentials configured in n8n.

    - Connect **13. Read Report PDF for Output** to this node.

20. **Send Report via Email:**

    - Add an **Email Send** node `14/b. Send Report in Email with SMTP`:
      - To Email: `report.receiver@example.com` (edit as needed)
      - From Email: `report.creator@example.com`
      - Subject: `Automatic NMAP Report for '{{ $('1. Set Parameters').item.json.network }}'`
      - Text: `The requested Automatic NMAP Scan finished, we attached the Report for '{{ $('1. Set Parameters').item.json.network }}'`
      - Attachments: Binary `vulnerability_report`
      - SMTP credentials configured in n8n.

    - Connect **13. Read Report PDF for Output** to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow scans network with Nmap, enriches findings from NVD CVE API, generates password-protected PDF reports, and sends via Telegram and Email. | Sticky Note - Overview                                                                           |
| Four triggers supported: webhook API, web form UI, manual trigger, scheduled trigger.                                        | Sticky Note - Triggers                                                                           |
| Verifies if `nmap-helper` tool is installed; installs from GitHub if missing.                                               | Sticky Note - Dependency                                                                         |
| Nmap scanning configured with full port scan, service/version detection, timing and parallelism parameters.                | Sticky Note - Scanning                                                                           |
| Queries NVD CVE API for each service with CPE, returning CVE IDs, CVSS scores, severity, references; rate-limited at 1 req/sec. | Sticky Note - CVE                                                                                |
| Aggregates enriched data by host, calculates statistics, generates detailed, styled HTML report.                           | Sticky Note - Report Gen                                                                         |
| Converts HTML report to password-protected PDF using PrinceXML; password set in workflow parameters.                       | Sticky Note - PDF                                                                                |
| Sends PDF reports via Telegram and email; requires updating credentials and recipients before use.                         | Sticky Note - Distribution                                                                       |
| CPE Search: https://nvd.nist.gov/products/cpe/search                                                                       | Sticky Note (CVE Enrichment)                                                                     |
| NVD API endpoint example: https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName={CPE Tag}&resultsPerPage=10&isVulnerable | Sticky Note (CVE Enrichment)                                                                     |
| PrinceXML documentation for HTML to PDF conversion: https://www.princexml.com/doc/intro-userguide/                         | Sticky Note7                                                                                     |

---

This detailed documentation enables understanding, reproduction, customization, and troubleshooting of the Network Vulnerability Scanner workflow with Nmap and automated CVE reporting. It covers all nodes, logic flows, configuration points, and potential failure scenarios to facilitate deployment and extension.