Automated DNS Records Lookup for Subdomains with HackerTarget API and Gmail Reports

https://n8nworkflows.xyz/workflows/automated-dns-records-lookup-for-subdomains-with-hackertarget-api-and-gmail-reports-6355


# Automated DNS Records Lookup for Subdomains with HackerTarget API and Gmail Reports

### 1. Workflow Overview

This workflow automates the process of enumerating subdomains for a target domain, performing DNS record lookups on each subdomain, formatting the results in Markdown, and sending a consolidated report via email. It is designed primarily for security and IT teams needing automated, periodic visibility into DNS configurations across subdomains.

**Target Use Cases:**
- Security monitoring for DNS misconfigurations or unauthorized changes.
- Compliance auditing aligned with frameworks like ISO 27001 and NIST CSF.
- Automated reporting for SOC and DevOps teams.
- Periodic DNS enumeration and record collection without manual effort.

**Logical Blocks:**

- **1.1 Input Reception and Domain Setup:**  
  Receives manual trigger to start the workflow and sets the target domain.

- **1.2 Subdomain Enumeration:**  
  Calls HackerTarget API to enumerate all subdomains of the target domain and parses raw output into structured data.

- **1.3 DNS Records Lookup:**  
  For each discovered subdomain, performs DNS lookups via HackerTarget API, parses the record data, and merges it with subdomain info.

- **1.4 Markdown Formatting:**  
  Formats the DNS records into a human-readable Markdown report per subdomain.

- **1.5 Report Aggregation and Delivery:**  
  Combines all subdomain Markdown reports into a single message and sends the report via Gmail.

- **1.6 Documentation and Guidance:**  
  Sticky notes provide explanations, instructions, and compliance context.


---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Domain Setup

**Overview:**  
Starts the workflow manually and defines the target domain to analyze.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- 🌐 Target Domain  
- Sticky Note1

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates execution manually.  
  - Config: No parameters, simple manual start.  
  - Connections: Output to "🌐 Target Domain" node.  
  - Failure Modes: None expected; user-triggered.

- **🌐 Target Domain**  
  - Type: Set node  
  - Role: Defines the domain for subdomain enumeration.  
  - Config: Sets a string parameter `domain` with default "example.com".  
  - Expressions: None dynamic; static default value.  
  - Connections: Outputs domain to "📡 Subdomain Enum".  
  - Failure Modes: If domain format invalid, downstream API calls may fail.  
  - Notes: Can be replaced by an external trigger or Cron node for automation.

- **Sticky Note1**  
  - Provides guidance to edit the target domain and notes option to automate via Cron.

---

#### 1.2 Subdomain Enumeration

**Overview:**  
Fetches subdomains using HackerTarget API and parses raw text response into structured JSON objects containing subdomain and IP.

**Nodes Involved:**  
- 📡 Subdomain Enum  
- 🧠 Parse Subdomains

**Node Details:**

- **📡 Subdomain Enum**  
  - Type: HTTP Request  
  - Role: Calls HackerTarget API endpoint `/hostsearch/?q=<domain>` to retrieve subdomains and IP addresses.  
  - Config: URL dynamically built using expression `{{$json["domain"]}}`.  
  - Response: Text format, raw CSV-like lines of `subdomain,ip`.  
  - Connections: Output to "🧠 Parse Subdomains".  
  - Failure Modes: Network errors, API rate limits, invalid domain input.

- **🧠 Parse Subdomains**  
  - Type: Code node (JavaScript)  
  - Role: Parses raw text output into array of JSON objects with keys `subdomain` and `ip`.  
  - Code Logic: Splits text by lines, filters empty lines, splits by comma, returns structured items.  
  - Connections: Outputs parallel to "🌐 DNS Records" and "🔗 Merge DNS + Subdomain" (initial).  
  - Failure Modes: Unexpected response format causing parse errors.

---

#### 1.3 DNS Records Lookup

**Overview:**  
Performs DNS record lookups for each subdomain and parses the resulting text response into categorized DNS record arrays.

**Nodes Involved:**  
- 🌐 DNS Records  
- 🧠 Parse DNS Records  
- 🔗 Merge DNS + Subdomain

**Node Details:**

- **🌐 DNS Records**  
  - Type: HTTP Request  
  - Role: Calls HackerTarget API `/dnslookup/?q=<subdomain>` to fetch DNS records.  
  - Config: URL dynamically set with `{{$json["subdomain"]}}`.  
  - Response: Text format stored under property `dns`.  
  - Connections: Output to "🧠 Parse DNS Records".  
  - Failure Modes: API limits, DNS query failure for specific subdomain, network timeouts.

- **🧠 Parse DNS Records**  
  - Type: Code node  
  - Role: Parses text DNS lookup into structured arrays for each DNS record type (A, AAAA, CNAME, TXT, MX, NS, SOA).  
  - Code Logic: Regex extracts lines like `TYPE: value`, accumulates values per type, defaults to "No records found." if empty.  
  - Connections: Output to second input of "🔗 Merge DNS + Subdomain".  
  - Failure Modes: Unexpected response format, empty responses.

- **🔗 Merge DNS + Subdomain**  
  - Type: Merge node  
  - Role: Combines parsed subdomain info and parsed DNS record data by matching on `subdomain`.  
  - Config: Mode "combine", join mode "keepEverything", matching on string field `subdomain`.  
  - Connections: Outputs merged data to "📝 Format DNS Markdown".  
  - Failure Modes: Mismatched keys leading to incomplete merges.

---

#### 1.4 Markdown Formatting

**Overview:**  
Formats DNS record arrays into a well-structured Markdown report snippet per subdomain.

**Nodes Involved:**  
- 📝 Format DNS Markdown  
- 🧩 Merge All Markdown

**Node Details:**

- **📝 Format DNS Markdown**  
  - Type: Code node (runOnceForEachItem mode)  
  - Role: Converts DNS record arrays into Markdown with headings per record type and lists of records.  
  - Logic: Iterates over standard DNS types, lists each record or states "No records found."  
  - Connections: Outputs Markdown snippets to "🧩 Merge All Markdown".  
  - Failure Modes: Missing properties causing undefined; defensive code handles missing arrays.

- **🧩 Merge All Markdown**  
  - Type: Code node  
  - Role: Aggregates all individual subdomain Markdown sections into one combined report string separated by horizontal rules (`---`).  
  - Connections: Outputs combined report to "Gmail".  
  - Failure Modes: Empty input array yields empty report.

---

#### 1.5 Report Aggregation and Delivery

**Overview:**  
Sends the compiled DNS Markdown report via Gmail to a specified recipient.

**Nodes Involved:**  
- Gmail  
- Sticky Note2

**Node Details:**

- **Gmail**  
  - Type: Gmail node  
  - Role: Sends the email with the DNS report.  
  - Config:  
    - Recipient: `security-team@example.com` (editable).  
    - Subject: Dynamic with prefix "🧠 DNS Report: All Subdomains".  
    - Message: HTML formatted email embedding the Markdown report inside `<pre>` tag for monospace formatting, plus footer with n8n branding link.  
  - Credentials: Gmail OAuth2 credential configured.  
  - Failure Modes: Authentication failure, quota limits, invalid recipient.

- **Sticky Note2**  
  - Explains purpose of Gmail node, suggests replacing with Slack, Notion, or webhook if needed, and reminds to update recipient.

---

#### 1.6 Documentation and Guidance

**Overview:**  
Provides detailed explanation of workflow purpose, benefits, compliance relevance, and usage recommendations.

**Nodes Involved:**  
- Sticky Note  

**Node Details:**

- **Sticky Note**  
  - Contains comprehensive multi-section content covering:  
    - Workflow purpose and workflow steps overview  
    - Benefits such as visibility, security, automation, audit readiness  
    - Compliance frameworks alignment with table  
    - Output destination and scheduling recommendations  
    - Optional future integrations  
    - Tags for categorization and searchability

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                                    | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                      |
|--------------------------|--------------------|---------------------------------------------------|-----------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Starts workflow execution manually                 |                             | 🌐 Target Domain              |                                                                                                |
| 🌐 Target Domain          | Set                | Defines target domain parameter                     | When clicking ‘Execute workflow’ | 📡 Subdomain Enum            | ✏️ Set your target domain here. Default is "example.com". Can connect to external trigger or Cron. |
| 📡 Subdomain Enum         | HTTP Request       | Retrieves subdomains from HackerTarget API         | 🌐 Target Domain             | 🧠 Parse Subdomains           |                                                                                                |
| 🧠 Parse Subdomains       | Code               | Parses raw CSV subdomain data into structured JSON | 📡 Subdomain Enum            | 🌐 DNS Records, 🔗 Merge DNS + Subdomain |                                                                                                |
| 🌐 DNS Records            | HTTP Request       | Fetches DNS records for each subdomain              | 🧠 Parse Subdomains          | 🧠 Parse DNS Records          |                                                                                                |
| 🧠 Parse DNS Records      | Code               | Parses DNS records text into categorized arrays     | 🌐 DNS Records               | 🔗 Merge DNS + Subdomain      |                                                                                                |
| 🔗 Merge DNS + Subdomain  | Merge              | Combines subdomain info with DNS record data        | 🧠 Parse Subdomains, 🧠 Parse DNS Records | 📝 Format DNS Markdown       |                                                                                                |
| 📝 Format DNS Markdown    | Code               | Converts DNS data to Markdown format per subdomain | 🔗 Merge DNS + Subdomain     | 🧩 Merge All Markdown         |                                                                                                |
| 🧩 Merge All Markdown     | Code               | Aggregates all Markdown snippets into one report    | 📝 Format DNS Markdown       | Gmail                        |                                                                                                |
| Gmail                    | Gmail              | Sends final DNS Markdown report by email            | 🧩 Merge All Markdown        |                              | 📧 Sends DNS Markdown report via email. Update recipient or replace with Slack, Notion, webhook. |
| Sticky Note              | Sticky Note        | Documentation and purpose explanation                |                             |                              | 🎯 Purpose, benefits, compliance, usage, scheduling, tags, and output destination details.       |
| Sticky Note1             | Sticky Note        | Guidance on setting target domain                    |                             |                              | ✏️ Set target domain, can automate with Cron or external triggers.                              |
| Sticky Note2             | Sticky Note        | Guidance on email sending node                        |                             |                              | 📧 Explains Gmail node usage, recipient update, and alternative integrations.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Add Manual Trigger Node**  
   - Node: Manual Trigger  
   - Purpose: Start workflow execution by manual click.  
   - No parameters needed.

2. **Add Set Node for Target Domain**  
   - Node: Set  
   - Name: `🌐 Target Domain`  
   - Add string field `domain` with default value `"example.com"`.  
   - Connect output of Manual Trigger to this node.

3. **Add HTTP Request Node for Subdomain Enumeration**  
   - Node: HTTP Request  
   - Name: `📡 Subdomain Enum`  
   - Method: GET  
   - URL: `https://api.hackertarget.com/hostsearch/?q={{$json["domain"]}}` (use expression to insert domain).  
   - Response Format: Text  
   - Connect output of `🌐 Target Domain` to this node.

4. **Add Code Node to Parse Subdomains**  
   - Node: Code  
   - Name: `🧠 Parse Subdomains`  
   - JavaScript code to:  
     - Retrieve raw text from previous node output.  
     - Split by newline, filter empty lines.  
     - Split each line by comma into `subdomain`, `ip`.  
     - Return array of JSON objects with these keys.  
   - Connect output of `📡 Subdomain Enum` to this node.

5. **Add HTTP Request Node for DNS Records Lookup**  
   - Node: HTTP Request  
   - Name: `🌐 DNS Records`  
   - Method: GET  
   - URL: `https://api.hackertarget.com/dnslookup/?q={{$json["subdomain"]}}` (use expression to insert subdomain).  
   - Response Format: Text, stored under property `dns`.  
   - Connect output of `🧠 Parse Subdomains` to this node.

6. **Add Code Node to Parse DNS Records**  
   - Node: Code  
   - Name: `🧠 Parse DNS Records`  
   - JavaScript code to:  
     - Extract `dns` text property from input.  
     - For each line matching `<TYPE>: <value>`, group values by type (A, AAAA, CNAME, TXT, MX, NS, SOA).  
     - If no records for a type, add "No records found."  
     - Return structured JSON with arrays per DNS type and subdomain.  
   - Connect output of `🌐 DNS Records` to this node.

7. **Add Merge Node to Combine Subdomain and DNS Data**  
   - Node: Merge  
   - Name: `🔗 Merge DNS + Subdomain`  
   - Mode: Combine  
   - Join Mode: Keep Everything  
   - Match Field: `subdomain` (string)  
   - Connect first input to `🧠 Parse Subdomains` output.  
   - Connect second input to `🧠 Parse DNS Records` output.

8. **Add Code Node to Format DNS Records as Markdown**  
   - Node: Code  
   - Name: `📝 Format DNS Markdown`  
   - Mode: Run Once For Each Item (process each merged item separately).  
   - JavaScript to:  
     - Access DNS record arrays for each type.  
     - Build Markdown string with headings per record type and bullet lists of records.  
     - If no records, state "No records found."  
     - Return JSON with `subdomain` and `dns_markdown`.  
   - Connect output of merge node here.

9. **Add Code Node to Merge All Markdown Reports**  
   - Node: Code  
   - Name: `🧩 Merge All Markdown`  
   - JavaScript to:  
     - Concatenate all `dns_markdown` strings from input items with separator `---`.  
     - Return single JSON with `full_report` field.  
   - Connect output of formatting node here.

10. **Add Gmail Node to Send Email**  
    - Node: Gmail  
    - Configure with valid Gmail OAuth2 credentials.  
    - Set recipient email (e.g., `security-team@example.com`).  
    - Subject: `🧠 DNS Report: All Subdomains` (expression allowed).  
    - Message: Use HTML `<pre>` tag embedding `{{$json["full_report"]}}` for monospace and preserve formatting.  
    - Connect output of merge markdown node here.

11. **Add Sticky Notes for Documentation**  
    - Add one explaining workflow purpose, benefits, compliance, usage, scheduling, and tags.  
    - Add one near target domain node advising how to change domain and automate trigger.  
    - Add one near Gmail node explaining email delivery and alternatives.

12. **Test Workflow**  
    - Trigger manually.  
    - Verify subdomains enumerated, DNS records fetched, Markdown formatted, and email sent successfully.  
    - Handle errors such as invalid domain or API rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| 🎯 Purpose: Automate DNS enumeration and reporting for security and compliance. Benefits include visibility, security monitoring, automation, audit readiness, and easy readability through Markdown formatting. Recommended to schedule via Cron for daily or weekly runs. Optional future integrations with Slack or SIEM systems.                                                                                                                                                                                     | Sticky Note (main workflow explanation)          |
| ✏️ Configure your target domain in the Set node. You can replace manual trigger with Cron or external webhook for scheduled runs.                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note1 (near target domain node)            |
| 📧 Final DNS Markdown report is sent by Gmail node. Update recipient email accordingly. You may replace Gmail with Slack, Notion, or webhook node depending on your notification preferences. Ensure Gmail OAuth2 credentials are properly configured.                                                                                                                                                                                                                                                                    | Sticky Note2 (near Gmail node)                     |
| HackerTarget API is used for subdomain enumeration and DNS lookups. Review API usage limits and reliability. Consider error handling for network issues or rate limiting.                                                                                                                                                                                                                                                                                                                                                | Workflow-wide consideration                        |
| Markdown formatting uses standard DNS record types: A, AAAA, CNAME, TXT, MX, NS, SOA. If no records are found for a type, the report explicitly states “No records found.” This improves clarity and audit-readiness.                                                                                                                                                                                                                                                                                                       | Code nodes for parsing and formatting              |
| Compliance Alignment includes frameworks such as ISO 27001 (A.12.6.1), NIST CSF (DE.CM-7), Essential Eight, SOC 2, CIS Controls (1 & 8) emphasizing vulnerability management and monitoring.                                                                                                                                                                                                                                                                                                                                 | Sticky Note (compliance table)                      |

---

This comprehensive documentation allows technical users and automation agents to understand, reproduce, and maintain the "Automated DNS Records Lookup for Subdomains with HackerTarget API and Gmail Reports" workflow confidently and efficiently.