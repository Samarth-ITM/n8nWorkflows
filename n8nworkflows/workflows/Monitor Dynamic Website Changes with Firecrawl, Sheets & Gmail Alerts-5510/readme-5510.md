Monitor Dynamic Website Changes with Firecrawl, Sheets & Gmail Alerts

https://n8nworkflows.xyz/workflows/monitor-dynamic-website-changes-with-firecrawl--sheets---gmail-alerts-5510


# Monitor Dynamic Website Changes with Firecrawl, Sheets & Gmail Alerts

### 1. Workflow Overview

This workflow monitors a specified dynamic website for content changes using Firecrawl’s web scraping API. It stores scraped content snapshots in Google Sheets, compares the latest scrape with the previous one, and sends an email alert via Gmail if changes are detected. The workflow is triggered externally via a webhook, making it suitable for automated periodic checks or integration with external schedulers.

**Logical Blocks:**

- **1.1 Input Reception:** Receives an HTTP request to trigger the monitoring process.
- **1.2 Web Scraping:** Calls Firecrawl API to scrape the target webpage, retrieving markdown and HTML formats.
- **1.3 Timestamp & Content Update:** Adds a timestamp and updates the current scraped content in Google Sheets.
- **1.4 Content Comparison:** Reads both last and current content from Google Sheets and compares them.
- **1.5 Change Handling:**
  - If unchanged: Responds immediately with no change noted.
  - If changed: Reads fresh data, extracts differences, sends an email alert, appends to log, updates last content, then responds.
- **1.6 Response:** Returns appropriate HTTP responses to webhook calls depending on whether content changed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Waits for an incoming HTTP request on a defined webhook path to start the workflow.
- **Nodes Involved:** Webhook
- **Node Details:**

  - **Webhook**
    - Type: Webhook Trigger
    - Configuration: Listens on path `/example_path` (customizable). Response mode set to “responseNode” meaning final response handled by Respond nodes.
    - Connections: Output → Firecrawl HTTP Request node
    - Edge Cases: Unreachable webhook endpoint if n8n instance is down or network misconfigured; malformed requests; unauthorized access (no auth configured here).
    - Version: 1

#### 1.2 Web Scraping

- **Overview:** Calls Firecrawl API with target URL to scrape content in markdown and HTML.
- **Nodes Involved:** Firecrawl HTTP Request
- **Node Details:**

  - **Firecrawl HTTP Request**
    - Type: HTTP Request
    - Configuration: POST to Firecrawl API with JSON body specifying the target URL (`https://we-hang.com/pages/n8n-automation-expert-challenge`) and formats `markdown` and `html`.
    - Authentication: Bearer token via configured credential named “Bearer Auth account.”
    - On Error: Continue workflow even if error (allows graceful failure).
    - Connections: Output → Get Timestamp
    - Edge Cases: API rate limits; invalid or expired token; unreachable API; malformed API response.
    - Version: 4.2

#### 1.3 Timestamp & Content Update

- **Overview:** Attaches a current millisecond timestamp to the scraped content and writes it into the “comparison” Google Sheets tab’s “current” columns.
- **Nodes Involved:** Get Timestamp, Update Current Content
- **Node Details:**

  - **Get Timestamp**
    - Type: Code (JavaScript)
    - Configuration: Iterates items, marks “high-port” if relevant (legacy), generates current timestamp in milliseconds.
    - Returns: Updated items with data and timestamp.
    - Connections: Output → Update Current Content
    - Edge Cases: Possible runtime errors in code logic; invalid input format.
    - Version: 2

  - **Update Current Content**
    - Type: Google Sheets (Update Operation)
    - Configuration: Updates row 2 in “comparison” sheet with:
      - `current_content` = scraped markdown content
      - `current_timestamp` = timestamp from previous node
    - Matching by row_number=2.
    - On Error: Continue workflow.
    - Connections: Output → Read Current and Latest Content
    - Credentials: OAuth2 Google Sheets account.
    - Edge Cases: Sheet ID or tab not found; OAuth token expiration; permission denied.
    - Version: 4.6

#### 1.4 Content Comparison

- **Overview:** Reads row 2 from “comparison” sheet to get both last and current content, then compares them to detect changes.
- **Nodes Involved:** Read Current and Latest Content, Is Equal?
- **Node Details:**

  - **Read Current and Latest Content**
    - Type: Google Sheets (Read Operation)
    - Configuration: Reads row 2 from “comparison” sheet, returning first match.
    - Credentials: Google Sheets OAuth2.
    - Connections: Output → Is Equal?
    - Edge Cases: Missing or malformed data; OAuth issues.
    - Version: 4

  - **Is Equal?**
    - Type: If node (Condition)
    - Configuration: Compares `last_content` and `current_content` strings for equality.
    - True branch → content unchanged → Respond Unchanged node.
    - False branch → content changed → Read Latest and Latest Content node.
    - Edge Cases: Null or empty content; data type mismatches.
    - Version: 1

#### 1.5 Change Handling

- **Overview:** Upon detecting changes, refreshes data, extracts content differences, sends notification email, appends historical log entry, updates “last” columns, and provides webhook response.
- **Nodes Involved:** Respond Unchanged, Read Latest and Latest Content, Extract Differences, Gmail, Append Log Row, Update Latest Content, Respond Changed
- **Node Details:**

  - **Respond Unchanged**
    - Type: Respond to Webhook
    - Configuration: Sends 200 OK response indicating no change.
    - Connections: None (end of unchanged branch).
    - Version: 1

  - **Read Latest and Latest Content**
    - Type: Google Sheets (Read Operation)
    - Configuration: Reads row 2 from “comparison” sheet again to ensure freshest data for downstream processing.
    - Credentials: Google Sheets OAuth2.
    - Connections: Output → Extract Differences, Append Log Row, Update Latest Content (parallel).
    - Version: 4

  - **Extract Differences**
    - Type: Code (JavaScript)
    - Configuration: Splits `last_content` and `current_content` strings by commas, finds new vs missing segments, attaches timestamp.
    - Returns: Object with `newContent`, `missingContent`, and timestamp.
    - Connections: Output → Gmail
    - Edge Cases: Unexpected string format; empty arrays; errors in splitting.
    - Version: 2

  - **Gmail**
    - Type: Gmail (Send Email)
    - Configuration: Sends plain-text email to `example@gmail.com` with subject “Relevante Inhaltsänderung auf Website” and message describing first detected change with timestamp.
    - Credentials: Gmail OAuth2 account.
    - Connections: Output → Respond Changed
    - Edge Cases: Email quota exceeded; invalid recipient; network issues.
    - Version: 2.1

  - **Append Log Row**
    - Type: Google Sheets (Append Operation)
    - Configuration: Appends new row to “Log” sheet (gid=0) with timestamp and full current content to maintain audit trail.
    - Credentials: Google Sheets OAuth2.
    - Connections: Output → Respond Changed
    - Edge Cases: Permission denied; quota issues.
    - Version: 4

  - **Update Latest Content**
    - Type: Google Sheets (Update Operation)
    - Configuration: Updates row 2 in “comparison” sheet, moving current content and timestamp into the “last_” columns for next comparison.
    - Credentials: Google Sheets OAuth2.
    - Connections: Output → Respond Changed
    - Edge Cases: Sheet access issues.
    - Version: 4.6

  - **Respond Changed**
    - Type: Respond to Webhook
    - Configuration: Sends 200 OK response confirming change was processed.
    - Connections: None (end of changed branch).
    - Version: 1

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                              | Input Node(s)               | Output Node(s)                              | Sticky Note                                                                                                  |
|---------------------------|-------------------------|----------------------------------------------|-----------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook Trigger         | Starts workflow on HTTP request               |                             | Firecrawl HTTP Request                       | Trigger node – waits for an HTTP request on /webhook/n8n-challenge and starts the workflow.                   |
| Firecrawl HTTP Request    | HTTP Request            | Scrapes website content via Firecrawl API    | Webhook                     | Get Timestamp                               | Calls Firecrawl API to scrape the target web page in Markdown and HTML. Uses Bearer-token authentication.     |
| Get Timestamp             | Code                    | Adds current timestamp and flags high port   | Firecrawl HTTP Request      | Update Current Content                       | JavaScript Code node – attaches a millisecond timestamp to the scraped content.                               |
| Update Current Content    | Google Sheets (Update)  | Updates current content and timestamp in sheet | Get Timestamp               | Read Current and Latest Content              | Writes freshly scraped content into row 2 of the comparison sheet.                                            |
| Read Current and Latest Content | Google Sheets (Read)    | Reads last and current content for comparison | Update Current Content       | Is Equal?                                   | Loads the same row 2, giving last_content and current_content.                                                |
| Is Equal?                 | If                      | Compares last and current content             | Read Current and Latest Content | Respond Unchanged (true), Read Latest and Latest Content (false) | IF node – checks whether the scraped content changed.                                                        |
| Respond Unchanged         | Respond to Webhook       | Returns 200 OK if unchanged                    | Is Equal? (true branch)      |                                             | Returns a 200 OK response to the original webhook caller when no change was detected.                          |
| Read Latest and Latest Content | Google Sheets (Read)    | Reads freshest content data post-change       | Is Equal? (false branch)     | Extract Differences, Append Log Row, Update Latest Content | Reads the current sheet row again on the changed path for freshest values.                                   |
| Extract Differences       | Code                    | Determines new vs missing content segments     | Read Latest and Latest Content | Gmail                                       | Splits last & current content into arrays, determines differences, attaches timestamp.                       |
| Gmail                     | Gmail                   | Sends email notification about detected change | Extract Differences          | Respond Changed                             | Sends a plain-text email detailing what changed to configured email address.                                 |
| Append Log Row            | Google Sheets (Append)  | Logs change with timestamp and full current content | Read Latest and Latest Content | Respond Changed                             | Adds a new row to Log sheet for historical audit trail.                                                      |
| Update Latest Content     | Google Sheets (Update)  | Moves current content into last_content fields | Read Latest and Latest Content | Respond Changed                             | Updates last_content and last_timestamp for next comparison.                                                 |
| Respond Changed           | Respond to Webhook       | Confirms webhook caller that change processed | Gmail, Append Log Row, Update Latest Content |                                         | Returns a success response to the original HTTP call confirming that a change was processed.                  |
| 📋 SETUP OVERVIEW          | Sticky Note             | Workflow purpose and flow summary              |                             |                                             | 🎯 WORKFLOW PURPOSE: monitors webpage, emails changes; key components: Firecrawl, Sheets, Gmail, Webhook.     |
| 🔐 CREDENTIALS SETUP       | Sticky Note             | Credentials setup instructions                  |                             |                                             | ⚙️ REQUIRED CREDENTIALS: Firecrawl API, Google Sheets, Gmail OAuth2 setup instructions.                       |
| 📊 GOOGLE SHEETS SETUP      | Sticky Note             | Google Sheets spreadsheet and tab structure    |                             |                                             | Spreadsheet structure: Log and comparison sheets, column and row 2 setup instructions.                        |
| ⚙️ CONFIGURATION           | Sticky Note             | Workflow customization parameters               |                             |                                             | Customization: target URL, email settings, webhook path, scraping format.                                     |
| 🧪 TESTING & ACTIVATION     | Sticky Note             | Testing instructions and activation tips        |                             |                                             | Steps for manual test, webhook test, troubleshooting.                                                        |
| 🤖 AUTOMATION SETUP         | Sticky Note             | Scheduling and external trigger options          |                             |                                             | Scheduling: cron jobs, external calls, n8n cron trigger alternative, external monitoring layers.             |
| 🔧 MAINTENANCE & MONITORING | Sticky Note             | Monitoring and error handling recommendations    |                             |                                             | Regular log review, error notifications, node error continuations, execution history monitoring.             |
| 🔒 SECURITY & BEST PRACTICES| Sticky Note             | Security and best practice guidelines             |                             |                                             | API key security, email security, webhook security, data privacy considerations.                             |
| 🔍 COMMON ISSUES & SOLUTIONS| Sticky Note             | Troubleshooting common errors                      |                             |                                             | Firecrawl errors, Google Sheets issues, Email problems, Webhook problems and solutions.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - Path: `example_path` (or your preferred path)
   - Response Mode: `responseNode`
   - No authentication configured here (consider adding if needed)
   - Connect output to next node

2. **Add HTTP Request Node (Firecrawl)**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.firecrawl.dev/scrape` (replace with actual Firecrawl endpoint if different)
   - Authentication: HTTP Bearer Token, credential named “Bearer Auth account”
   - Body Content Type: JSON
   - JSON Body:
     ```json
     {
       "url": "https://we-hang.com/pages/n8n-automation-expert-challenge",
       "formats": ["markdown","html"]
     }
     ```
   - Set On Error to “Continue”
   - Connect webhook node output to this node

3. **Add Code Node (Get Timestamp)**
   - Type: Code (JavaScript)
   - Code:
     ```js
     const items = $input.all();

     const updatedItems = items.map((item) => {
       if (item.json.data.port > 5000) {
         item.json.data["high-port"] = true;
       }
       return item.json;
     });

     const timestamp = new Date().getTime();

     return { updatedItems, timestamp };
     ```
   - On Error: Continue
   - Connect Firecrawl HTTP Request output to this node

4. **Add Google Sheets Node (Update Current Content)**
   - Operation: Update
   - Sheet Name: `comparison`
   - Document ID: Your Google Sheet ID
   - Matching Column: `row_number` = 2
   - Map columns:
     - `current_content` = `{{ $json.updatedItems[0].data.markdown }}`
     - `current_timestamp` = `{{ $json.timestamp }}`
   - Credentials: Google Sheets OAuth2 account
   - On Error: Continue
   - Connect Get Timestamp output to this node

5. **Add Google Sheets Node (Read Current and Latest Content)**
   - Operation: Read
   - Sheet Name: `comparison`
   - Document ID: Same Google Sheet ID
   - Return First Match Only
   - Credentials: Google Sheets OAuth2 account
   - Connect Update Current Content output to this node

6. **Add If Node (Is Equal?)**
   - Condition: String equality check
     - `{{$json.last_content}}` equals `{{$json.current_content}}`
   - Execute Once: True
   - Connect Read Current and Latest Content output to this node

7. **Add Respond to Webhook Node (Respond Unchanged)**
   - For True branch of If node
   - Sends a simple 200 OK response indicating no change
   - Connect Is Equal? True branch here

8. **Add Google Sheets Node (Read Latest and Latest Content)**
   - Operation: Read
   - Sheet Name: `comparison`
   - Document ID: Same as above
   - Return First Match Only
   - Credentials: Google Sheets OAuth2
   - Connect Is Equal? False branch here

9. **Add Code Node (Extract Differences)**
   - Type: Code (JavaScript)
   - Code:
     ```js
     const items = $input.all().map((item) => item.json);

     const result = items.map((item) => {
       const lastContent = item.last_content.split(",");
       const currentContent = item.current_content.split(",");
       const newContent = currentContent.filter((i) => !lastContent.includes(i));
       const missingContent = lastContent.filter((i) => !currentContent.includes(i));
       return {
         ...item,
         newContent,
         missingContent,
         timestamp: new Date().getTime(),
       };
     });

     return result;
     ```
   - On Error: Continue
   - Connect Read Latest and Latest Content output to this node

10. **Add Gmail Node**
    - Operation: Send Email
    - Recipient: `example@gmail.com` (replace with your actual email)
    - Subject: `Relevante Inhaltsänderung auf Website`
    - Message (plain text):
      ```
      Guten Tag, 

      Es gab eine Änderung an der beobachteten Website:

      Um "{{ $json.timestamp }}" wurde "{{ $json.missingContent[0] }}" durch "{{ $json.newContent[0] }}" ersetzt.

      Viele Grüße
      ```
    - Credentials: Gmail OAuth2 account
    - On Error: Continue
    - Connect Extract Differences output to Gmail

11. **Add Google Sheets Node (Append Log Row)**
    - Operation: Append
    - Sheet Name: `Log` (gid=0)
    - Document ID: Same Google Sheet ID
    - Columns to append:
      - `timestamp` = `{{ $json.current_timestamp }}`
      - `content` = `{{ $json.current_content }}`
    - Credentials: Google Sheets OAuth2
    - On Error: Continue
    - Connect Read Latest and Latest Content output to this node (parallel to Extract Differences)

12. **Add Google Sheets Node (Update Latest Content)**
    - Operation: Update
    - Sheet Name: `comparison`
    - Document ID: Same Google Sheet ID
    - Matching Column: `row_number` = 2
    - Columns to update:
      - `last_content` = `{{ $json.current_content }}`
      - `last_timestamp` = `{{ $json.current_timestamp }}`
    - Credentials: Google Sheets OAuth2
    - On Error: Continue
    - Connect Read Latest and Latest Content output (parallel to Append Log Row and Extract Differences)

13. **Add Respond to Webhook Node (Respond Changed)**
    - Sends a 200 OK confirming change processed
    - Connect outputs of Gmail, Append Log Row, and Update Latest Content nodes to this Respond node

14. **Activate the workflow**

15. **Configure Credentials**
    - Set up Firecrawl API Bearer token credential named “Bearer Auth account”
    - Set up Google Sheets OAuth2 credential named “Google Sheets account”
    - Set up Gmail OAuth2 credential named “Gmail account”

16. **Set Google Sheets**
    - Create Google Sheets with two tabs: `Log` (gid=0) and `comparison` (gid=725909638)
    - `comparison` tab columns: last_timestamp, last_content, current_timestamp, current_content, row_number
    - Set row 2, column `row_number` = 2 (static row for updates)
    - `Log` tab columns: timestamp and content

17. **Customize parameters**
    - Replace Firecrawl URL in HTTP Request node with your target website
    - Replace Gmail recipient email in Gmail node
    - Adjust webhook path as desired

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 🎯 WORKFLOW PURPOSE: Monitors a webpage for content changes and sends email notifications using Firecrawl, Google Sheets, Gmail, and webhook triggers.                         | Sticky note “📋 SETUP OVERVIEW”                                                                  |
| ⚙️ Required credentials setup instructions for Firecrawl API, Google Sheets API, and Gmail API including OAuth2 configuration and where to input them in n8n.                  | Sticky note “🔐 CREDENTIALS SETUP”                                                              |
| 📋 Spreadsheet structure explained with two tabs: “Log” for audit trail and “comparison” for current vs last content storage; includes column names and row setup.            | Sticky note “📊 GOOGLE SHEETS SETUP”                                                            |
| 🔧 Customization tips: update Firecrawl target URL, Gmail recipient, webhook path, and scraping format (currently markdown + html).                                           | Sticky note “⚙️ CONFIGURATION”                                                                  |
| 🧪 Testing steps: manual execution, webhook testing via curl/Postman, and troubleshooting tips including credential and permission checks.                                    | Sticky note “🧪 TESTING & ACTIVATION”                                                           |
| 🤖 Scheduling options: external cron jobs calling webhook, replacing webhook with n8n Cron Trigger node, or external monitoring services like UptimeRobot.                   | Sticky note “🤖 AUTOMATION SETUP”                                                               |
| 📈 Monitoring best practices: review execution history, check logs in Google Sheets, monitor errors with “continue on error” enabled for graceful degradation.                | Sticky note “🔧 MAINTENANCE & MONITORING”                                                      |
| 🛡️ Security recommendations: store credentials securely, avoid hardcoding API keys, use HTTPS, consider webhook authentication, and monitor email/bounce issues.              | Sticky note “🔒 SECURITY & BEST PRACTICES”                                                     |
| ⚠️ Common issues and solutions for Firecrawl API errors, Google Sheets access problems, Gmail quota or spam issues, and webhook activation/URL correctness.                  | Sticky note “🔍 COMMON ISSUES & SOLUTIONS”                                                      |

---

This comprehensive documentation provides the full logic, configuration details, edge cases, and reproduction steps for the “Monitor Dynamic Website Changes with Firecrawl, Sheets & Gmail Alerts” workflow, enabling both expert users and AI agents to understand, replicate, and maintain the solution effectively.