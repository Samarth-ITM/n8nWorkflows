Parse DMARC reports, save them in database and notify on DKIM or SPF error

https://n8nworkflows.xyz/workflows/parse-dmarc-reports--save-them-in-database-and-notify-on-dkim-or-spf-error-2369


# Parse DMARC reports, save them in database and notify on DKIM or SPF error

### 1. Workflow Overview

This workflow automates the processing of DMARC (Domain-based Message Authentication, Reporting & Conformance) aggregate reports received via email. It is designed primarily for postmasters or email server administrators who need to monitor DKIM (DomainKeys Identified Mail) and SPF (Sender Policy Framework) authentication results for their domains. The workflow extracts, parses, and transforms DMARC reports sent as zipped XML attachments, stores structured data into a MySQL/MariaDB database, and sends notifications only when SPF or DKIM authentication failures are detected.

Logical blocks grouped by functionality:

- **1.1 Input Reception:** Monitors an email inbox for incoming DMARC report emails and fetches attachments.
- **1.2 Report Extraction and Parsing:** Unzips the received report, extracts XML data, and parses the XML into JSON format.
- **1.3 Data Normalization and Mapping:** Handles cases of multiple report entries, normalizes keys, and maps JSON fields to a database-compatible structure.
- **1.4 Date Formatting:** Converts date fields into MySQL/MariaDB compatible datetime strings.
- **1.5 Database Storage:** Inserts parsed and formatted DMARC report data into the database.
- **1.6 DKIM/SPF Failure Detection and Notification:** Checks for DKIM or SPF failures and triggers notifications via Slack and email if failures are present.
- **1.7 Informational Annotations:** Sticky notes provide explanations and reminders about setup and functionality.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block monitors the configured IMAP mailbox for new emails containing DMARC reports and downloads any attachments for further processing.

- **Nodes Involved:**  
  - Email Trigger (IMAP)

- **Node Details:**  
  - **Email Trigger (IMAP)**  
    - Type: `emailReadImap`  
    - Role: Watches an IMAP mailbox and downloads emails and their attachments.  
    - Configuration: Downloads attachments (`downloadAttachments: true`), no additional filters set.  
    - Credentials: Uses an IMAP account credential for authentication.  
    - Input: None (trigger node)  
    - Output: Email data with attachments (binary property `attachment_0`).  
    - Edge cases: Email server connection issues, authentication errors, empty inbox, emails without attachments.  
    - Notes: Critical to configure mailbox properly to receive DMARC reports (rua tag).

---

#### 2.2 Report Extraction and Parsing

- **Overview:**  
  This block decompresses the zipped DMARC report attached to the email, extracts the XML report file, and parses it into JSON.

- **Nodes Involved:**  
  - Unzip File  
  - Extract XML data  
  - Parse XML data to JSON  
  - Sticky Note1 (informational)

- **Node Details:**  
  - **Unzip File**  
    - Type: `compression`  
    - Role: Decompresses zipped email attachment to access the XML report inside.  
    - Configuration: Uses binary property `attachment_0` from incoming email.  
    - Input: Email Trigger (IMAP) output  
    - Output: Binary property `file_0` containing extracted XML file.  
    - Edge cases: Invalid zip files, corrupted attachments, no zipped attachment found.  
  - **Extract XML data**  
    - Type: `extractFromFile`  
    - Role: Extracts XML content from decompressed file binary data.  
    - Configuration: Operates on binary property `file_0`.  
    - Input: Unzip File output  
    - Output: XML content as binary for parsing.  
    - Edge cases: Non-XML files, empty or malformed files.  
  - **Parse XML data to JSON**  
    - Type: `xml`  
    - Role: Converts XML content into JSON format for easier processing in n8n.  
    - Configuration: Default XML parsing options.  
    - Input: Extract XML data output  
    - Output: JSON object representing the DMARC report structure.  
    - Edge cases: Malformed XML, parsing errors.  
  - **Sticky Note1**  
    - Content: Explains the preparation step of converting email data to JSON.

---

#### 2.3 Data Normalization and Mapping

- **Overview:**  
  This block handles cases where multiple DMARC records exist in one report, normalizes JSON keys for consistency, and maps JSON fields into a structure suitable for database insertion.

- **Nodes Involved:**  
  - If multiple records to parse  
  - Split Out For Separate Entries  
  - Rename Keys  
  - Rename column for consistency  
  - Map fields for DB input and parse  
  - Sticky Note2 (mapping explanation)

- **Node Details:**  
  - **If multiple records to parse**  
    - Type: `if`  
    - Role: Checks if the JSON contains multiple DMARC record entries.  
    - Condition: Checks if `$json.feedback.record[0]` exists.  
    - Input: Parse XML data to JSON output  
    - Output: Two branches: true (multiple records), false (single record).  
    - Edge cases: Unexpected JSON structure, missing keys.  
  - **Split Out For Separate Entries**  
    - Type: `splitOut`  
    - Role: Splits array of DMARC records into individual items for separate processing.  
    - Field to split: `feedback.record`  
    - Input: If multiple records to parse (true branch)  
    - Output: Individual DMARC record entries.  
  - **Rename Keys**  
    - Type: `renameKeys`  
    - Role: Renames key `feedback.record` to `fbr` for shorter and consistent referencing.  
    - Input: Split Out For Separate Entries output  
    - Output: JSON with renamed keys.  
  - **Rename column for consistency**  
    - Type: `set`  
    - Role: For the single record case (false branch of If multiple records), assigns `feedback.record` to `fbr` field to normalize downstream processing.  
    - Input: If multiple records to parse (false branch)  
    - Output: JSON with `fbr` field assigned  
  - **Map fields for DB input and parse**  
    - Type: `set`  
    - Role: Extracts and maps relevant fields from JSON to flat fields matching database columns; also converts sub-objects to JSON strings for storage.  
    - Fields mapped include org_name, domain, date_range_begin/end, policy details, source IP, mail count, DKIM/SPF evaluation results, identifiers, and auth results.  
    - Input: Converged output from Rename Keys and Rename column for consistency.  
    - Output: Structured JSON ready for date formatting and DB input.  
    - Edge cases: Missing fields, null values handled by conditional expressions.  
  - **Sticky Note2**  
    - Content: Explains handling multiple entries and mapping data to the database structure.

---

#### 2.4 Date Formatting

- **Overview:**  
  Converts DMARC report date fields from timestamps or raw format to MySQL/MariaDB-compatible datetime strings.

- **Nodes Involved:**  
  - Begin format date  
  - End date format  
  - Sticky Note3

- **Node Details:**  
  - **Begin format date**  
    - Type: `dateTime`  
    - Role: Formats `date_range_begin` field using a custom format (`yyyy-MM-dd hh:mm:ss`).  
    - Input: Map fields for DB input and parse output  
    - Output: JSON with formatted `date_range_begin`  
  - **End date format**  
    - Type: `dateTime`  
    - Role: Formats `date_range_end` field similarly.  
    - Input: Begin format date output  
    - Output: JSON with formatted `date_range_end`  
  - **Sticky Note3**  
    - Content: Explains purpose of translating date formats.

---

#### 2.5 Database Storage

- **Overview:**  
  Inserts the parsed and formatted DMARC report data into the configured MySQL/MariaDB database table.

- **Nodes Involved:**  
  - Input into database

- **Node Details:**  
  - **Input into database**  
    - Type: `mySql`  
    - Role: Inserts the mapped DMARC report data as a new row in the `dmarc` table.  
    - Configuration: Uses `defineBelow` mode to specify column-value pairs explicitly.  
    - Fields inserted: full_data (JSON string), org_name, date_range_begin, date_range_end, domain, policy_published (JSON string), source_ip, mail_count, evaluated_disposition, evaluated_dkim, evaluated_spf, identifiers (JSON string or null), auth_results (JSON string or null).  
    - Credentials: Uses MySQL account credential.  
    - Input: End date format output  
    - Output: Database insertion result with detailed output enabled.  
    - Edge cases: Database connectivity issues, constraint violations, data type mismatches.

---

#### 2.6 DKIM/SPF Failure Detection and Notification

- **Overview:**  
  Evaluates if the DMARC report indicates any DKIM or SPF failures and sends notifications accordingly via Slack and email.

- **Nodes Involved:**  
  - If issue with DKIM or SPF  
  - Slack Post Message On Channel (disabled)  
  - Send Error Notification Email (disabled)  
  - Sticky Note4

- **Node Details:**  
  - **If issue with DKIM or SPF**  
    - Type: `if`  
    - Role: Checks if either `evaluated_dkim` or `evaluated_spf` values are not equal to "pass".  
    - Condition: Logical OR between DKIM and SPF evaluation status.  
    - Input: Input into database output  
    - Output: True branch triggers notifications, false branch ends workflow silently.  
  - **Slack Post Message On Channel**  
    - Type: `slack`  
    - Role: Posts a message to a specified Slack channel reporting the DKIM/SPF failure details.  
    - Disabled: Yes (requires enabling before use).  
    - Configuration: Uses OAuth2 Slack credential; message template includes domain, mail count, disposition, DKIM and SPF statuses.  
    - Input: If issue with DKIM or SPF (true branch)  
    - Edge cases: Slack API rate limits, authentication issues.  
  - **Send Error Notification Email**  
    - Type: `emailSend`  
    - Role: Sends an email notification with failure details.  
    - Disabled: Yes (requires enabling).  
    - Configuration: Simple text email with subject "DMARC problem".  
    - Input: If issue with DKIM or SPF (true branch)  
    - Edge cases: SMTP server errors, invalid email address.  
  - **Sticky Note4**  
    - Content: Explains notification logic and purpose.

---

#### 2.7 Informational Annotations

- **Nodes Involved:**  
  - Sticky Note (positioned at workflow start)

- **Details:**  
  - Provides high-level summary of the workflow steps and setup reminders such as configuring the email input mailbox and notification channels (Slack, email).

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                           | Input Node(s)                    | Output Node(s)                              | Sticky Note                                                                                     |
|-----------------------------|-----------------------|-----------------------------------------|---------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------|
| Email Trigger (IMAP)         | emailReadImap          | Monitors email inbox, downloads DMARC reports | None                            | Unzip File                                  |                                                                                                |
| Unzip File                  | compression            | Decompresses zipped DMARC report         | Email Trigger (IMAP)            | Extract XML data                            |                                                                                                |
| Extract XML data            | extractFromFile        | Extracts XML content from decompressed file | Unzip File                     | Parse XML data to JSON                      |                                                                                                |
| Parse XML data to JSON      | xml                    | Converts XML to JSON                      | Extract XML data                | If multiple records to parse                | Sticky Note1: Explains converting email data to JSON                                           |
| If multiple records to parse| if                     | Checks for multiple DMARC records        | Parse XML data to JSON          | Split Out For Separate Entries, Rename column for consistency |                                                                                                |
| Split Out For Separate Entries | splitOut             | Splits multiple records into single items | If multiple records to parse   | Rename Keys                                 |                                                                                                |
| Rename Keys                | renameKeys             | Normalizes JSON keys                      | Split Out For Separate Entries  | Map fields for DB input and parse           | Sticky Note2: Explains handling multiple entries and mapping data                             |
| Rename column for consistency| set                   | Normalizes single record JSON structure  | If multiple records to parse (false branch) | Map fields for DB input and parse           |                                                                                                |
| Map fields for DB input and parse | set               | Maps JSON fields to database schema      | Rename Keys, Rename column for consistency | Begin format date                           |                                                                                                |
| Begin format date           | dateTime               | Formats begin date for DB                 | Map fields for DB input and parse| End date format                             | Sticky Note3: Explains date formatting                                                        |
| End date format             | dateTime               | Formats end date for DB                   | Begin format date              | Input into database                         |                                                                                                |
| Input into database         | mySql                  | Inserts data into MySQL/MariaDB           | End date format               | If issue with DKIM or SPF                   |                                                                                                |
| If issue with DKIM or SPF   | if                     | Checks DKIM/SPF failure                    | Input into database            | Slack Post Message On Channel, Send Error Notification Email | Sticky Note4: Explains notification logic                                                      |
| Slack Post Message On Channel | slack (disabled)      | Sends Slack notification on failure       | If issue with DKIM or SPF      | None                                       |                                                                                                |
| Send Error Notification Email | emailSend (disabled)  | Sends email notification on failure       | If issue with DKIM or SPF      | None                                       |                                                                                                |
| Sticky Note                 | stickyNote             | Workflow overview and setup reminders     | None                         | None                                       | Explains workflow steps and setup reminders                                                  |
| Sticky Note1                | stickyNote             | Explains XML to JSON conversion            | None                         | None                                       | Explains preparation step                                                                     |
| Sticky Note2                | stickyNote             | Explains data mapping and multiple entries | None                         | None                                       | Explains mapping and multiple records handling                                               |
| Sticky Note3                | stickyNote             | Explains date formatting                    | None                         | None                                       | Explains date translation for DB insertion                                                  |
| Sticky Note4                | stickyNote             | Explains DKIM/SPF failure notification logic | None                         | None                                       | Explains notification logic                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Email Trigger (IMAP) node:**  
   - Type: `emailReadImap`  
   - Configure with your IMAP credentials (host, port, username, password, SSL as needed).  
   - Enable attachment download. Leave filters empty to fetch all emails.  
   - Position: start of workflow.

2. **Create a Compression node ("Unzip File"):**  
   - Type: `compression`  
   - Set `binaryPropertyName` to `attachment_0` (the first attachment from email trigger).  
   - Connect the Email Trigger output to this node input.

3. **Create Extract From File node ("Extract XML data"):**  
   - Type: `extractFromFile`  
   - Set operation to `xml`.  
   - Set `binaryPropertyName` to `file_0` (output from compression node).  
   - Connect Unzip File output to this node.

4. **Create XML node ("Parse XML data to JSON"):**  
   - Type: `xml`  
   - Default parsing options.  
   - Connect Extract XML data output to this node.

5. **Create an If node ("If multiple records to parse"):**  
   - Type: `if`  
   - Condition: Check if `$json.feedback.record[0]` exists (using expression).  
   - Connect Parse XML data to JSON output to this node.

6. **Create a Split Out node ("Split Out For Separate Entries"):**  
   - Type: `splitOut`  
   - Field to split: `feedback.record`  
   - Connect If multiple records to parse (true branch) output to this node.

7. **Create Rename Keys node ("Rename Keys"):**  
   - Type: `renameKeys`  
   - Rename `feedback.record` to `fbr`.  
   - Connect Split Out For Separate Entries output to this node.

8. **Create Set node ("Rename column for consistency"):**  
   - Type: `set`  
   - Assign field `fbr` = `$json.feedback.record`. Enable "include other fields".  
   - Connect If multiple records to parse (false branch) output to this node.

9. **Create Set node ("Map fields for DB input and parse"):**  
   - Type: `set`  
   - Map the following fields (use expressions):  
     - `full_data` = entire `feedback` object as JSON string.  
     - `org_name` = `feedback.report_metadata.org_name`  
     - `date_range_begin` = `feedback.report_metadata.date_range.begin`  
     - `date_range_end` = `feedback.report_metadata.date_range.end`  
     - `domain` = `feedback.policy_published.domain`  
     - `policy_published` = `feedback.policy_published` as JSON string  
     - `source_ip` = `fbr.row.source_ip`  
     - `mail_count` = `fbr.row.count`  
     - `evaluated_disposition` = `fbr.row.policy_evaluated.disposition`  
     - `evaluated_dkim` = `fbr.row.policy_evaluated.dkim`  
     - `evaluated_spf` = `fbr.row.policy_evaluated.spf`  
     - `identifiers` = `fbr.identifiers` as JSON string or null  
     - `auth_results` = `fbr.auth_results` as JSON string or null  
   - Connect both Rename Keys and Rename column for consistency outputs to this node (merge branches).

10. **Create DateTime node ("Begin format date"):**  
    - Type: `dateTime`  
    - Input field: `date_range_begin`  
    - Format: Custom `yyyy-MM-dd hh:mm:ss`  
    - Output field: `date_range_begin`  
    - Connect Map fields for DB input and parse output.

11. **Create DateTime node ("End date format"):**  
    - Type: `dateTime`  
    - Input field: `date_range_end`  
    - Format: Custom `yyyy-MM-dd hh:mm:ss`  
    - Output field: `date_range_end`  
    - Connect Begin format date output.

12. **Create MySQL node ("Input into database"):**  
    - Type: `mySql`  
    - Credentials: Set up your MySQL/MariaDB connection.  
    - Table: `dmarc`  
    - Data Mode: Define below, set column-to-value mappings corresponding to mapped fields (use expressions as in Map fields node).  
    - Connect End date format output.

13. **Create If node ("If issue with DKIM or SPF"):**  
    - Type: `if`  
    - Condition: Logical OR:  
      - `$json.evaluated_dkim != 'pass'`  
      - `$json.evaluated_spf != 'pass'`  
    - Connect Input into database output.

14. **Create Slack node ("Slack Post Message On Channel") [optional]:**  
    - Type: `slack`  
    - Configure OAuth2 credentials for Slack.  
    - Channel: select or provide channel ID for notifications.  
    - Message template:  
      `DMARC evaluation failed for {{ $json.domain }} on {{ $json.mail_count }} mails with disposition: {{ $json.evaluated_disposition }}. DKIM: {{ $json.evaluated_dkim }} SPF: {{ $json.evaluated_spf }}`  
    - Connect If issue with DKIM or SPF (true branch).  
    - Enable this node when notifications are needed.

15. **Create Email Send node ("Send Error Notification Email") [optional]:**  
    - Type: `emailSend`  
    - Configure SMTP credentials for your email account.  
    - Subject: `DMARC problem`  
    - Text body: same template as Slack message.  
    - Connect If issue with DKIM or SPF (true branch).  
    - Enable when email notifications are desired.

16. **Add Sticky Notes for documentation:**  
    - Add notes describing overview, parsing, mapping, date formatting, and notification logic as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow automatically parses DMARC aggregate reports received via email and triggers notifications only on DKIM/SPF failures.                                  | Workflow purpose & usage                                    |
| Remember to configure your IMAP email account to receive DMARC reports via the `rua` tag in your DNS DMARC record.                                              | Setup reminder                                              |
| Slack and email notification nodes are disabled by default; enable and configure credentials as needed.                                                         | Notification setup                                          |
| Date fields are converted to `yyyy-MM-dd hh:mm:ss` format for compatibility with MySQL/MariaDB datetime columns.                                                | Date formatting detail                                     |
| The workflow handles multiple DMARC records per report by splitting and processing each separately.                                                              | Data normalization detail                                  |
| For more information on DMARC, DKIM, and SPF, see official documentation: https://dmarc.org/                                                                    | External resource                                           |
| The database schema must include columns matching mapped fields, with appropriate JSON/text columns for full_data, policy_published, identifiers, auth_results. | Database schema requirements                               |

---

This completes the comprehensive analysis and documentation of the "Parse DMARC reports" workflow.