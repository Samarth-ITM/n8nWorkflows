Convert DMARC Reports to Readable Tables with Gmail Automation

https://n8nworkflows.xyz/workflows/convert-dmarc-reports-to-readable-tables-with-gmail-automation-9584


# Convert DMARC Reports to Readable Tables with Gmail Automation

### 1. Workflow Overview

This workflow automates the processing of DMARC (Domain-based Message Authentication, Reporting, and Conformance) reports received via Gmail. DMARC reports are typically sent as compressed XML files (zip or gz) by major email providers like Gmail and Yahoo, but their raw format is difficult to read. The workflow extracts these attachments, decompresses and parses the XML data, converts it into a human-readable HTML table summarizing key report details, and then sends this table back via email. This enables users to easily monitor DMARC reports without manual unpacking or parsing.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled Email Retrieval:** Triggered daily at a set time, it connects to Gmail and fetches recent DMARC report emails with attachments.
- **1.2 Attachment Extraction and Decompression:** Downloads and extracts the compressed XML attachments from emails.
- **1.3 XML Parsing and JSON Conversion:** Converts the extracted XML content into JSON format for easier manipulation.
- **1.4 HTML Table Generation:** Builds a styled HTML table summarizing DMARC report data such as source IPs, counts, DKIM and SPF evaluation results.
- **1.5 Email Sending:** Sends the formatted summary table via Gmail to a specified recipient.

The workflow includes sticky notes that describe each logical block, offering helpful contextual guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Email Retrieval

- **Overview:**  
  This block triggers the workflow daily at a specified hour, then fetches up to 5 recent DMARC report emails from Gmail received in the last 24 hours. It filters emails from known DMARC sources and downloads attachments for processing.

- **Nodes Involved:**  
  - `cron`  
  - `dmarc`

- **Node Details:**  

  - **cron**  
    - *Type & Role:* Schedule Trigger node; initiates workflow execution daily.  
    - *Configuration:* Set to trigger once a day at 13:00 (1 PM) server time.  
    - *Key Variables:* None directly; triggers downstream nodes.  
    - *Connections:* Output connects to `dmarc` node.  
    - *Failure Modes:* Could fail if server time zone changes or node is disabled.  
    - *Version:* 1.2.

  - **dmarc**  
    - *Type & Role:* Gmail node; retrieves emails matching DMARC report criteria.  
    - *Configuration:*  
      - Operation: `getAll` to fetch multiple emails.  
      - Limit: 5 emails maximum.  
      - Filter Query: emails from `noreply@dmarc.yahoo.com` or `noreply-dmarc-support@google.com` received in last 24 hours.  
      - Options: Download attachments enabled.  
    - *Credentials:* Uses OAuth2 Gmail credentials named `shownotes`.  
    - *Connections:* Output connects to `unzip` node.  
    - *Failure Modes:* Authentication errors, Gmail quota limits, or no matching emails found.  
    - *Version:* 2.1.

---

#### 1.2 Attachment Extraction and Decompression

- **Overview:**  
  This block takes the compressed attachments from the fetched emails, decompresses them regardless of whether they are ZIP or GZ format, and extracts the XML files contained inside.

- **Nodes Involved:**  
  - `unzip`  
  - `xml`

- **Node Details:**  

  - **unzip**  
    - *Type & Role:* Compression node; decompresses binary attachment data.  
    - *Configuration:*  
      - Uses the binary property named `attachment_0` (first attachment from email) as input.  
      - Automatically detects compression format (zip/gz).  
    - *Connections:* Output binary file connects to `xml` node.  
    - *Failure Modes:* Corrupted attachments, unsupported compression types, empty attachments.  
    - *Version:* 1.1.

  - **xml**  
    - *Type & Role:* Extract From File node; extracts XML content from decompressed binary data.  
    - *Configuration:*  
      - Operation: `xml` extraction.  
      - Binary property name: `file_0` (output from unzip node).  
    - *Connections:* Outputs parsed XML data to `xml2json` node.  
    - *Failure Modes:* Malformed XML, missing file, extraction errors.  
    - *Version:* 1.

---

#### 1.3 XML Parsing and JSON Conversion

- **Overview:**  
  Converts the extracted XML into JSON format to enable easy access and manipulation of DMARC report data.

- **Nodes Involved:**  
  - `xml2json`

- **Node Details:**  

  - **xml2json**  
    - *Type & Role:* XML node; converts XML string to JSON data structure.  
    - *Configuration:* Default XML to JSON conversion options.  
    - *Connections:* Outputs JSON to the `set` node.  
    - *Failure Modes:* Invalid XML input, conversion errors.  
    - *Version:* 1.

---

#### 1.4 HTML Table Generation

- **Overview:**  
  This block creates a clean, styled HTML table presenting key DMARC report details like the reporting organization, domain, DMARC policy, source IP addresses, count of emails, and DKIM/SPF evaluation results.

- **Nodes Involved:**  
  - `set`

- **Node Details:**  

  - **set**  
    - *Type & Role:* Set node; constructs an HTML table string as a workflow variable.  
    - *Configuration:*  
      - Assigns a new string field named `table` with a multi-line HTML template.  
      - Template uses JavaScript expressions to dynamically insert JSON data from previous nodes.  
      - Handles cases where there are multiple or single DMARC records.  
    - *Key Expressions:*  
      - Uses `$json["feedback"]["report_metadata"]["org_name"]` for organization name.  
      - Uses `$json["feedback"]["policy_published"]["domain"]` for domain.  
      - Iterates over `$json["feedback"]["record"]` array to build rows.  
    - *Connections:* Outputs to `send` node.  
    - *Failure Modes:* Missing JSON fields, malformed data structures, expression evaluation errors.  
    - *Version:* 3.4.

---

#### 1.5 Email Sending

- **Overview:**  
  Sends the generated HTML table as an email message to a fixed recipient with a subject line dynamically filled with the reporting organization and domain.

- **Nodes Involved:**  
  - `send`

- **Node Details:**  

  - **send**  
    - *Type & Role:* Gmail node; sends outgoing email with DMARC summary.  
    - *Configuration:*  
      - Send to: `cooper@shownotes.io` (hardcoded recipient email).  
      - Subject: Combines reporting org name and domain dynamically from JSON.  
      - Message: Uses the HTML table string from the previous node.  
      - Options: Attribution appended is disabled to keep email clean.  
    - *Credentials:* Uses OAuth2 Gmail credentials named `shownotes`.  
    - *Failure Modes:* Authentication failure, invalid recipient email, message size too large, Gmail rate limits.  
    - *Version:* 2.1.

---

### 3. Summary Table

| Node Name   | Node Type              | Functional Role                     | Input Node(s) | Output Node(s) | Sticky Note                                                                                      |
|-------------|------------------------|-----------------------------------|---------------|----------------|-------------------------------------------------------------------------------------------------|
| cron        | Schedule Trigger       | Daily trigger to start workflow   |               | dmarc          | Gmail and Yahoo send DMARC reports as zip/gz xml files that are hard to read. This workflow unpacks them, turns the data into a simple table, and emails you an easy-to-read report |
| dmarc       | Gmail                  | Fetch DMARC report emails          | cron          | unzip          | Get DMARC reports and download the attached reports.                                            |
| unzip       | Compression            | Decompress DMARC report attachment | dmarc         | xml            | Convert the XML to JSON. Handle both zip and gz formats. Create HTML table for email.           |
| xml         | Extract From File      | Extract XML content from binary    | unzip         | xml2json       | Convert the XML to JSON. Handle both zip and gz formats. Create HTML table for email.           |
| xml2json    | XML                    | Convert XML to JSON                | xml           | set            | Convert the XML to JSON. Handle both zip and gz formats. Create HTML table for email.           |
| set         | Set                    | Generate HTML table from JSON data | xml2json      | send           | Convert the XML to JSON. Handle both zip and gz formats. Create HTML table for email.           |
| send        | Gmail                  | Send email with DMARC report       | set           |                | Send report by email. Suggestion: You could add an Airtable node here if you want to keep records. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: `Schedule Trigger`  
   - Name: `cron`  
   - Set to trigger daily at 13:00 (1 PM).  
   - Save.

2. **Create a Gmail node to fetch DMARC emails**  
   - Type: `Gmail`  
   - Name: `dmarc`  
   - Operation: `getAll`  
   - Limit: 5 emails  
   - Filters: Query string: `from:(noreply@dmarc.yahoo.com OR noreply-dmarc-support@google.com) in:anywhere`  
   - Received after: Expression `={{ $now - 24 * 60 * 60 * 1000 }}` (last 24 hours)  
   - Options: Enable downloading attachments  
   - Credentials: Select or create Gmail OAuth2 credentials (named e.g. `shownotes`)  
   - Connect `cron` output to `dmarc` input.

3. **Create a Compression node to decompress attachments**  
   - Type: `Compression`  
   - Name: `unzip`  
   - Binary Property Name: `attachment_0` (first attachment from emails)  
   - Connect `dmarc` output to `unzip` input.

4. **Create an Extract From File node to extract XML**  
   - Type: `Extract From File`  
   - Name: `xml`  
   - Operation: `xml`  
   - Binary Property Name: `file_0` (output from `unzip`)  
   - Connect `unzip` output to `xml` input.

5. **Create an XML node to convert XML to JSON**  
   - Type: `XML`  
   - Name: `xml2json`  
   - Use default options for XML to JSON conversion  
   - Connect `xml` output to `xml2json` input.

6. **Create a Set node to build HTML table**  
   - Type: `Set`  
   - Name: `set`  
   - Add a new string field called `table`.  
   - Use the following HTML template with embedded JavaScript expressions to dynamically populate data (copy exact template below):  

   ```html
   <table border="1" cellpadding="4" cellspacing="0" style="width:100%; max-width:760px; border-collapse:collapse; margin:12px 0;">
   <tr>
       <th colspan="4" align="left" style="text-align:left; padding:6px 8px;">
         <strong>{{$json["feedback"]["report_metadata"]["org_name"]}}</strong> → 
         {{$json["feedback"]["policy_published"]["domain"]}}
         <code style="white-space:normal; word-break:break-all; font-family:Courier, monospace; margin-left:4px;">
           [v=DMARC1; p={{$json["feedback"]["policy_published"]["p"]}};]
         </code>
       </th>
     </tr>
     
     <!-- Records header -->
   <tr>
     <th align="left" style="text-align:left;">IP</th>
     <th align="left" style="text-align:left;">Count</th>
     <th align="left" style="text-align:left;">DKIM</th>
     <th align="left" style="text-align:left;">SPF</th>
   </tr>
   
     <!-- Records rows -->
     {{
       Array.isArray($json["feedback"]["record"])
         ? $json["feedback"]["record"].map(r => `
           <tr>
             <td><code>${r.row.source_ip}</code></td>
             <td style="text-align:right;">${r.row.count}</td>
             <td style="text-align:left;">${r.row.policy_evaluated.dkim}</td>
             <td style="text-align:left;">${r.row.policy_evaluated.spf}</td>
           </tr>
         `).join('')
         : `<tr>
             <td><code>${$json["feedback"]["record"]["row"]["source_ip"]}</code></td>
             <td style="text-align:left;">${$json["feedback"]["record"]["row"]["count"]}</td>
             <td style="text-align:left;">${$json["feedback"]["record"]["row"]["policy_evaluated"]["dkim"]}</td>
             <td style="text-align:left;">${$json["feedback"]["record"]["row"]["policy_evaluated"]["spf"]}</td>
           </tr>`
     }}
   </table>
   ```

   - Connect `xml2json` output to `set` input.

7. **Create a Gmail node to send email**  
   - Type: `Gmail`  
   - Name: `send`  
   - Operation: Send email  
   - Send To: `cooper@shownotes.io` (adjust as needed)  
   - Subject: Expression  

     ``` 
     ={{ $('xml2json').item.json.feedback.report_metadata.org_name }} DMARC Report for {{ $('xml2json').item.json.feedback.policy_published.domain }}
     ```  
   - Message: Expression `={{ $json.table }}` (HTML table from `set` node)  
   - Options: Disable "Append Attribution" to keep email clean  
   - Credentials: Use Gmail OAuth2 credentials as before (`shownotes`)  
   - Connect `set` output to `send` input.

8. **Activate the workflow**  
   - Ensure all nodes are connected and credentials set.  
   - Activate the workflow to enable daily runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Gmail and Yahoo send DMARC reports as zip/gz xml files that are hard to read. This workflow unpacks them, turns the data into a simple table, and emails you an easy-to-read report. | Sticky Note on `cron` node                                        |
| Get DMARC reports and download the attached reports.                                                                    | Sticky Note on `dmarc` node                                       |
| Convert the XML to JSON. Handle both zip and gz formats. Create HTML table for email.                                    | Sticky Notes on `unzip`, `xml`, `xml2json`, and `set` nodes      |
| Send report by email. Suggestion: You could add an Airtable node here if you want to keep records.                       | Sticky Note on `send` node                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.