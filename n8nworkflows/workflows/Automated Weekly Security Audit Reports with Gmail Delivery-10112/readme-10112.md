Automated Weekly Security Audit Reports with Gmail Delivery

https://n8nworkflows.xyz/workflows/automated-weekly-security-audit-reports-with-gmail-delivery-10112


# Automated Weekly Security Audit Reports with Gmail Delivery

### 1. Workflow Overview

This workflow automates the generation and delivery of detailed weekly security audit reports for an n8n instance. It is designed to analyze the security posture of workflows, credentials, and instance settings, then send a richly formatted HTML email summary to a specified recipient. The workflow includes the following logical blocks:

- **1.1 Schedule Trigger (Weekly):** Initiates the workflow automatically every week at a defined time.
- **1.2 Configuration Setup:** Defines key variables such as recipient email, project name, server URL, and language preference.
- **1.3 Security Audit Generation:** Calls the n8n API to generate comprehensive security audit data.
- **1.4 Data Deduplication:** Extracts unique workflow IDs from audit results to avoid redundant processing.
- **1.5 Execution Status Retrieval:** Fetches the most recent execution status of each workflow to enrich the audit report with runtime data.
- **1.6 Language-Based Report Formatting:** Routes audit data to either a French or English formatter that produces Markdown and HTML reports, including risk analysis.
- **1.7 Email Dispatch:** Sends the formatted HTML report via Gmail OAuth2 to the configured recipient.

The design ensures modularity, enabling easy customization of scheduling, localization, and email delivery options.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger (Weekly)

- **Overview:** Automatically triggers the workflow every Monday at 6 AM to start the audit process.
- **Nodes Involved:**  
  - `Schedule Trigger (Weekly)`
- **Node Details:**
  - Type: Cron Trigger  
  - Configuration: Set to trigger weekly at 06:00 hours.  
  - Input: None (trigger node)  
  - Output: Triggers `Set Config Variables` node.  
  - Edge Cases: Ensure server time zone aligns with expected run time; misconfiguration can lead to missed runs.  
  - Sticky Note: Explains how to modify the schedule for different frequencies.

#### 2.2 Configuration Setup

- **Overview:** Assigns essential configuration variables that govern email recipient, project context, server URL, and report language.
- **Nodes Involved:**  
  - `Set Config Variables`
- **Node Details:**
  - Type: Set Node  
  - Configuration:  
    - `email_to`: Target email address for report delivery (default "abc@xyz.com").  
    - `project_name`: Friendly name for the project or instance (default "N8N-main").  
    - `server_url`: Base URL of the n8n server without trailing slash (must be set correctly).  
    - `Language`: Report language preference ("EN" or "FR").  
  - Input: Trigger from `Schedule Trigger`  
  - Output: Passes variables downstream to `Generate a security audit`.  
  - Edge Cases: Incorrect or missing URL format can break report links; invalid email leads to delivery failure.  
  - Sticky Note: Emphasizes editing these variables before first run.

#### 2.3 Security Audit Generation

- **Overview:** Invokes the n8n API to generate a comprehensive security audit report covering credentials, node risks, and instance settings.
- **Nodes Involved:**  
  - `Generate a security audit`
- **Node Details:**
  - Type: n8n API Node (Audit resource)  
  - Configuration: Uses n8n API credentials with audit permission.  
  - Input: Configuration variables from `Set Config Variables`.  
  - Output: Raw audit data fed to `Filter duplicate WorkflowID`.  
  - Edge Cases: API authentication failures, permission issues, or server downtime.  
  - Sticky Note: Instructions on creating and using an API key credential are provided.

#### 2.4 Data Deduplication

- **Overview:** Processes the audit's "Nodes Risk Report" section to extract unique workflows, removing duplicates to optimize subsequent processing.
- **Nodes Involved:**  
  - `Filter duplicate WorkflowID`
- **Node Details:**
  - Type: Code Node (JavaScript)  
  - Configuration:  
    - Extracts unique workflow objects by `workflowId`.  
    - Returns array of unique workflows with counts and node types.  
  - Input: Audit data JSON from `Generate a security audit`.  
  - Output: Unique workflows passed to `Get last executions`.  
  - Edge Cases: Missing or malformed `location` array throws error.  
  - Sticky Note: Indicates automatic deduplication, no user configuration needed.

#### 2.5 Execution Status Retrieval

- **Overview:** For each unique workflow, retrieves the most recent execution to show success/failure and timing info in the report.
- **Nodes Involved:**  
  - `Get last executions`
- **Node Details:**
  - Type: n8n API Node (Execution resource)  
  - Configuration:  
    - Limits to 1 latest execution per workflow ID.  
    - Uses the same n8n API credential as audit generation.  
  - Input: Unique workflows from `Filter duplicate WorkflowID`.  
  - Output: Execution data forwarded to `If Language`.  
  - Edge Cases: No executions found if workflows never ran; API permission errors possible.  
  - Sticky Note: Explains importance of execution status enrichment.

#### 2.6 Language-Based Report Formatting

- **Overview:** Routes data based on selected language to generate either French or English audit reports with Markdown and HTML content, including risk assessments and actionable insights.
- **Nodes Involved:**  
  - `If Language`  
  - `Format Audit Report - FR`  
  - `Format Audit Report - EN`
- **Node Details:**

  - **If Language:**
    - Type: If Node  
    - Configuration: Checks if `Language` variable equals "FR".  
    - Input: Execution data from `Get last executions`.  
    - Output: Routes to French formatter if true, else English formatter.  
    - Edge Cases: Case sensitivity; unrecognized language defaults to English.

  - **Format Audit Report - FR:**
    - Type: Code Node (JavaScript)  
    - Configuration:  
      - Parses audit data and execution info.  
      - Aggregates statistics: total credentials, nodes, community nodes.  
      - Groups nodes by workflow with icon mappings.  
      - Formats detailed Markdown and HTML reports with clickable workflow links and execution status badges (success/failure).  
      - Calculates overall risk level (Low/Moderate/High) with colored emoji indicators.  
      - Constructs email subject line reflecting risk level.  
    - Input: Audit and execution data.  
    - Output: JSON with `markdown`, `html`, `emailSubject`, and risk metadata for email sending.  
    - Edge Cases: Missing data sections, malformed inputs, date/time localization issues.  
    - Sticky Note: Describes report contents in French.

  - **Format Audit Report - EN:**
    - Type: Code Node (JavaScript)  
    - Configuration: Same logic as French formatter but localized to English language and date formatting.  
    - Input/Output: Identical to FR node but localized content.  
    - Edge Cases: Same as French formatter.  
    - Sticky Note: Same functionality, English localization.

#### 2.7 Email Dispatch

- **Overview:** Sends the fully formatted audit report via Gmail in rich HTML format to the configured email address.
- **Nodes Involved:**  
  - `Send Gmail (HTML)`
- **Node Details:**
  - Type: Gmail Node  
  - Configuration:  
    - Recipient email taken dynamically from `email_to` variable.  
    - Subject and HTML body sourced from formatter output.  
    - Uses Gmail OAuth2 credential.  
  - Input: Formatted data from either `Format Audit Report - FR` or `Format Audit Report - EN`.  
  - Output: Sends email; terminal node.  
  - Edge Cases: OAuth2 token expiration, invalid email address, Gmail API quota limits.  
  - Sticky Note: Suggests alternative email nodes (SMTP, Outlook).

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                          | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|----------------------|----------------------------------------|--------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger (Weekly)  | Cron Trigger         | Initiates weekly workflow execution    | None                     | Set Config Variables         | Automatically runs every Monday at 6 AM; change schedule in node settings if needed          |
| Set Config Variables       | Set Node             | Defines email, project, URL, language  | Schedule Trigger (Weekly) | Generate a security audit    | Configuration - edit before first run; controls entire workflow variables                   |
| Generate a security audit  | n8n API Node         | Calls n8n API to generate security audit | Set Config Variables      | Filter duplicate WorkflowID  | Requires n8n API credential with audit permission                                           |
| Filter duplicate WorkflowID | Code Node           | Extracts unique workflows from audit   | Generate a security audit | Get last executions          | Deduplication of workflows; no config needed                                               |
| Get last executions        | n8n API Node         | Fetches latest execution per workflow  | Filter duplicate WorkflowID | If Language                 | Retrieves execution status; requires n8n API credential                                    |
| If Language                | If Node              | Routes to language-specific formatter  | Get last executions       | Format Audit Report - FR / EN | Routes based on Language variable "FR" or else English                                    |
| Format Audit Report - FR   | Code Node            | Formats French audit report and metrics | If Language (true path)   | Send Gmail (HTML)            | Creates Markdown & HTML report with risk levels, links, and summary                        |
| Format Audit Report - EN   | Code Node            | Formats English audit report and metrics | If Language (false path)  | Send Gmail (HTML)            | Same as FR formatter but in English                                                        |
| Send Gmail (HTML)          | Gmail Node           | Sends formatted report via Gmail       | Format Audit Report - FR/EN | None                      | Sends rich HTML email; requires Gmail OAuth2 credential; can be replaced with other email nodes |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger (Weekly)` node:**
   - Type: Cron Trigger  
   - Set trigger time to every Monday at 06:00 (hour: 6, mode: everyWeek).

2. **Create `Set Config Variables` node:**
   - Type: Set Node  
   - Define variables:  
     - `email_to`: Set to recipient email address (e.g., "your.email@domain.com").  
     - `project_name`: Set your project or instance name.  
     - `server_url`: Set your n8n server URL without trailing slash (e.g., "https://n8n.yourdomain.com").  
     - `Language`: Set "EN" or "FR" for report language.  
   - Connect output of `Schedule Trigger (Weekly)` to this node.

3. **Create `Generate a security audit` node:**
   - Type: n8n API Node  
   - Resource: `audit`  
   - Operation: `generate`  
   - Use n8n API credentials with audit access (create API key via Settings → API).  
   - Connect output of `Set Config Variables` to this node.

4. **Create `Filter duplicate WorkflowID` node:**
   - Type: Code Node (JavaScript)  
   - Paste provided code that extracts unique workflows from `"Nodes Risk Report"` → `sections[0].location`.  
   - Connect output of `Generate a security audit` to this node.

5. **Create `Get last executions` node:**
   - Type: n8n API Node  
   - Resource: `execution`  
   - Operation: `list` or equivalent for fetching executions  
   - Limit to 1 result per workflow ID.  
   - Use same n8n API credential as for audit generation.  
   - Configure filter to use `workflowId` dynamically from input JSON.  
   - Connect output of `Filter duplicate WorkflowID` to this node.

6. **Create `If Language` node:**
   - Type: If Node  
   - Condition: Compare `Language` variable from `Set Config Variables`.  
   - Condition: Equals "FR" (case-sensitive, loose validation).  
   - Connect output of `Get last executions` to this node.

7. **Create `Format Audit Report - FR` node:**
   - Type: Code Node (JavaScript)  
   - Paste French formatting code for report generation (Markdown + HTML + metadata).  
   - Connect "true" output of `If Language` to this node.

8. **Create `Format Audit Report - EN` node:**
   - Type: Code Node (JavaScript)  
   - Paste English formatting code for report generation (Markdown + HTML + metadata).  
   - Connect "false" output of `If Language` to this node.

9. **Create `Send Gmail (HTML)` node:**
   - Type: Gmail Node  
   - Configure OAuth2 Gmail credentials (set up in Google Cloud Console, enable Gmail API).  
   - Set recipient to expression referencing `email_to` from `Set Config Variables`:  
     `={{ $('Set Config Variables').first().json.email_to }}`  
   - Set subject to `={{ $json.emailSubject }}` from formatter output.  
   - Set HTML message content to `={{ $json.html }}`.  
   - Connect outputs of both `Format Audit Report - FR` and `Format Audit Report - EN` to this node.

10. **Activate the workflow:**
    - Test manually to verify configuration and email delivery.  
    - Activate for automated weekly execution.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Before first run, ensure `server_url` has no trailing slash and `Language` is exactly "EN" or "FR".         | Sticky Note11 - Important Notes                                                                     |
| Create an n8n API key with audit permissions via Settings → API → Permissions.                               | Sticky Note2, Sticky Note11                                                                         |
| Setup Gmail OAuth2 credential via Google Cloud Console with Gmail API enabled.                              | Sticky Note8, Sticky Note11                                                                         |
| Typical execution time is 10-20 seconds; email arrives within 1 minute.                                    | Sticky Note13                                                                                       |
| Customize schedule trigger for different frequencies (daily, monthly, custom cron).                        | Sticky Note7, Sticky Note1                                                                           |
| Risk thresholds can be adjusted by editing JavaScript conditions inside the formatter nodes.               | Sticky Note9                                                                                        |
| Email delivery node can be replaced by SMTP, Outlook, or other email nodes if preferred.                   | Sticky Note8                                                                                        |
| Workflow links in reports depend on correct `server_url` format; broken links indicate URL misconfiguration.| Sticky Note12                                                                                       |
| If reports are empty or missing execution data, verify API permissions and that workflows have been executed at least once. | Sticky Note12                                                                                       |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n integration and automation tool. It strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.