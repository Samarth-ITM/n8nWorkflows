Monitor Compliance with GPT-4 Analysis of System Logs and Generate Audit Reports

https://n8nworkflows.xyz/workflows/monitor-compliance-with-gpt-4-analysis-of-system-logs-and-generate-audit-reports-11775


# Monitor Compliance with GPT-4 Analysis of System Logs and Generate Audit Reports

### 1. Workflow Overview

This workflow automates the process of monitoring organizational compliance by collecting diverse system logs and data, analyzing them with AI for potential compliance violations, and generating detailed audit reports. It targets use cases in IT security, compliance auditing, and risk management, where daily automated checks against standards like GDPR, SOC 2, and ISO 27001 are required.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Configuration Initialization:** Starts the workflow on a fixed daily schedule and sets key configuration parameters including API endpoints and compliance standards.
- **1.2 Data Collection:** Pulls logs and data from multiple systems—server logs, API gateway logs, HR system, and finance system—for the last 24 hours.
- **1.3 Data Aggregation and Normalization:** Combines all gathered data into a unified format with consistent timestamping and metadata.
- **1.4 AI-Powered Compliance Analysis:** Processes normalized data with a specialized AI agent to detect and classify compliance violations according to defined standards.
- **1.5 Violation Handling and Evidence Archiving:** Checks if any violations exist, archives evidence to storage, and prepares data for reporting.
- **1.6 Audit Report Generation and Distribution:** Generates a comprehensive HTML audit report summarizing findings, sends it via email, and creates corrective action tasks in a task management system.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Configuration Initialization

**Overview:**  
This block triggers the workflow every day at 2 AM and initializes all configuration parameters such as API endpoints and compliance standards used downstream.

**Nodes Involved:**  
- Schedule Compliance Audit  
- Workflow Configuration

**Node Details:**  

- **Schedule Compliance Audit**  
  - Type: Schedule Trigger  
  - Role: Automatically starts the workflow daily at 2 AM.  
  - Configuration: Interval set to trigger at hour 2 every day.  
  - Inputs: None  
  - Outputs: Workflow Configuration node  
  - Failure Modes: Misconfiguration of schedule or time zone issues might delay execution.

- **Workflow Configuration**  
  - Type: Set  
  - Role: Defines and assigns workflow-wide variables such as URLs for various logs, compliance standards, and email recipients.  
  - Configuration: Contains placeholder strings for all external API endpoints (server logs, API gateway logs, HR system, finance system, storage API, task management system) and compliance standards (GDPR, SOC 2, ISO 27001).  
  - Key Expressions: None, static assignment with placeholders.  
  - Inputs: From Schedule Compliance Audit  
  - Outputs: Multiple HTTP Request nodes for data pulling  
  - Edge Cases: If placeholders are not replaced with valid URLs or email addresses, requests will fail.

---

#### 2.2 Data Collection

**Overview:**  
This block fetches logs and data from multiple external sources for the last 24 hours, based on configured API URLs.

**Nodes Involved:**  
- Pull Server Logs  
- Pull API Gateway Logs  
- Pull HR System Data  
- Pull Finance System Data

**Node Details:**  

- **Pull Server Logs**  
  - Type: HTTP Request  
  - Role: Retrieves server logs for the last 24 hours using the configured API endpoint.  
  - Configuration: URL taken from Workflow Configuration; query parameters `startDate` and `endDate` set dynamically based on current time minus one day and now.  
  - Inputs: Workflow Configuration  
  - Outputs: Combine All Logs node  
  - Failure Modes: Network errors, invalid URL, API authentication or rate limiting issues, empty or malformed responses.

- **Pull API Gateway Logs**  
  - Same type and role as above but fetches API gateway logs.

- **Pull HR System Data**  
  - Fetches HR system data logs similarly.

- **Pull Finance System Data**  
  - Fetches finance system data logs similarly.

---

#### 2.3 Data Aggregation and Normalization

**Overview:**  
Combines all collected logs into a single array and normalizes them into a consistent structure with standardized fields.

**Nodes Involved:**  
- Combine All Logs  
- Normalize Log Data

**Node Details:**  

- **Combine All Logs**  
  - Type: Aggregate  
  - Role: Merges data from all four HTTP request nodes into one aggregated dataset.  
  - Configuration: Aggregates all item data into one output.  
  - Inputs: Pull Server Logs, Pull API Gateway Logs, Pull HR System Data, Pull Finance System Data  
  - Outputs: Normalize Log Data node  
  - Edge Cases: Null or empty inputs from any source can reduce dataset completeness.

- **Normalize Log Data**  
  - Type: Set  
  - Role: Normalizes the aggregated logs, adding standardized fields like `timestamp`, `source`, `logData`, and `auditDate`.  
  - Configuration:  
    - `timestamp`: Current ISO timestamp  
    - `source`: Takes existing source or defaults to "unknown"  
    - `logData`: Stores raw JSON log data from aggregate  
    - `auditDate`: Current date in `yyyy-MM-dd` format  
  - Inputs: Combine All Logs  
  - Outputs: AI Compliance Classifier  
  - Edge Cases: Failure in date formatting or missing `source` values.

---

#### 2.4 AI-Powered Compliance Analysis

**Overview:**  
Analyzes normalized log data with a specialized AI agent using GPT-4 to detect compliance violations, classify severity, provide evidence, and suggest corrective actions.

**Nodes Involved:**  
- AI Compliance Classifier  
- OpenAI GPT-4  
- Structured Violation Output  
- Check for Violations

**Node Details:**  

- **AI Compliance Classifier**  
  - Type: LangChain Agent  
  - Role: Sends the `logData` to an AI agent configured with a detailed system message outlining compliance standards and violation types to detect.  
  - Configuration:  
    - System message specifies analysis scope including GDPR, SOC 2, ISO 27001 violations, severity classification, evidence extraction, and corrective suggestions.  
    - Prompt type: Define  
    - Has output parser enabled.  
  - Inputs: Normalize Log Data  
  - Outputs: Check for Violations node, OpenAI GPT-4 (language model)  
  - Edge Cases: AI API failures, prompt misinterpretation, output parsing errors.

- **OpenAI GPT-4**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Executes GPT-4 model inference called by AI Compliance Classifier.  
  - Configuration: Model set to GPT-4 (gpt-4o variant), uses stored OpenAI credentials.  
  - Inputs: AI Compliance Classifier (language model)  
  - Outputs: Structured Violation Output  
  - Failure Modes: API rate limits, auth errors, model unavailability.

- **Structured Violation Output**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI output into a structured JSON schema specifying violations details and summary.  
  - Configuration: JSON schema defines required fields: `hasViolations` (boolean), `violations` (array of violation objects), and `summary` (string).  
  - Inputs: OpenAI GPT-4  
  - Outputs: AI Compliance Classifier (output parser)  
  - Edge Cases: Parsing errors if AI output does not match schema.

- **Check for Violations**  
  - Type: If  
  - Role: Checks if any violations were detected (`hasViolations === true`) and routes flow accordingly.  
  - Inputs: AI Compliance Classifier  
  - Outputs: Archive Evidence to Storage if true; else workflow ends or could be extended.  
  - Edge Cases: False negatives or positives in violation detection.

---

#### 2.5 Violation Handling and Evidence Archiving

**Overview:**  
Archives violation evidence and metadata into a storage system and prepares data for report generation.

**Nodes Involved:**  
- Archive Evidence to Storage  
- Prepare Audit Report Data

**Node Details:**  

- **Archive Evidence to Storage**  
  - Type: HTTP Request  
  - Role: Posts all relevant audit data including audit date, timestamp, violations, summary, and raw logs to an external storage API.  
  - Configuration:  
    - URL from Workflow Configuration  
    - Method: POST  
    - JSON body includes audit metadata and violation details dynamically extracted from previous nodes.  
  - Inputs: Check for Violations (true branch)  
  - Outputs: Prepare Audit Report Data  
  - Failure Modes: Network failures, invalid API endpoint, authorization errors.

- **Prepare Audit Report Data**  
  - Type: Set  
  - Role: Calculates summary metrics for the report such as total violations and counts by severity level.  
  - Configuration: Uses JavaScript expressions to:  
    - Count total violations  
    - Count Critical, High, Medium, Low severity violations separately  
    - Compose report title with audit date  
  - Inputs: Archive Evidence to Storage  
  - Outputs: Generate Audit Report HTML  
  - Edge Cases: Empty or malformed violation arrays.

---

#### 2.6 Audit Report Generation and Distribution

**Overview:**  
Generates a detailed HTML audit report, sends it via email to configured recipients, and creates corrective action tasks in a task management system.

**Nodes Involved:**  
- Generate Audit Report HTML  
- Send Audit Report Email  
- Create Corrective Action Tasks

**Node Details:**  

- **Generate Audit Report HTML**  
  - Type: Code  
  - Role: Creates a comprehensive HTML report summarizing violations, severity breakdowns, compliance rate, recommendations, and detailed tables with evidence and corrective actions.  
  - Configuration: JavaScript code use dynamic data from previous node outputs, constructs styled HTML with sections for summary, violation details categorized by severity, and recommendations based on violation patterns.  
  - Inputs: Prepare Audit Report Data  
  - Outputs: Send Audit Report Email  
  - Edge Cases: JavaScript runtime errors, malformed data inputs.

- **Send Audit Report Email**  
  - Type: Gmail  
  - Role: Sends the generated HTML report to configured email addresses.  
  - Configuration:  
    - Recipients from Workflow Configuration  
    - Subject set to report title  
    - Message body is the generated HTML  
    - Uses Gmail OAuth2 credentials  
  - Inputs: Generate Audit Report HTML  
  - Outputs: Create Corrective Action Tasks  
  - Failure Modes: Email sending errors, invalid recipient addresses, Gmail API quota limits.

- **Create Corrective Action Tasks**  
  - Type: HTTP Request  
  - Role: Posts tasks for each detected violation to a task management system to facilitate remediation tracking.  
  - Configuration:  
    - URL from Workflow Configuration  
    - Method: POST  
    - JSON body maps violations to tasks with title, description, priority, corrective action, and due date calculated based on severity (Critical = +1 day, High = +3 days, else +7 days).  
  - Inputs: Send Audit Report Email  
  - Outputs: None (workflow end)  
  - Failure Modes: API errors, data mapping issues, network failures.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                                  | Input Node(s)                             | Output Node(s)                        | Sticky Note                                                                                                                        |
|----------------------------|---------------------------------------|-------------------------------------------------|------------------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Compliance Audit   | Schedule Trigger                      | Initiates workflow daily at 2 AM                 | None                                     | Workflow Configuration              | ## Schedule Trigger **What:** Initiates the workflow on a defined schedule. **Why:** Ensures regular and consistent data aggregation |
| Workflow Configuration      | Set                                  | Sets API URLs, compliance standards, recipients | Schedule Compliance Audit                 | Pull Server Logs, Pull API Gateway Logs, Pull HR System Data, Pull Finance System Data | ## Setup Steps * Connect credentials: Slack, Teams, Gmail OAuth, GitHub, Anthropic * Configure monitoring parameters * Set schedule triggers |
| Pull Server Logs            | HTTP Request                        | Fetch server logs for last 24 hours              | Workflow Configuration                   | Combine All Logs                   | ## Merge AI Sources **What:** Consolidates data from five input channels into a single stream. **Why:** Creates a unified dataset     |
| Pull API Gateway Logs       | HTTP Request                        | Fetch API gateway logs for last 24 hours         | Workflow Configuration                   | Combine All Logs                   | See above                                                                                                                        |
| Pull HR System Data         | HTTP Request                        | Fetch HR system data for last 24 hours            | Workflow Configuration                   | Combine All Logs                   | See above                                                                                                                        |
| Pull Finance System Data    | HTTP Request                        | Fetch finance system data for last 24 hours       | Workflow Configuration                   | Combine All Logs                   | See above                                                                                                                        |
| Combine All Logs            | Aggregate                           | Aggregate all logs into one dataset               | Pull Server Logs, Pull API Gateway Logs, Pull HR System Data, Pull Finance System Data | Normalize Log Data                |                                                                                                                                |
| Normalize Log Data          | Set                                | Normalize and enrich aggregated logs              | Combine All Logs                         | AI Compliance Classifier           |                                                                                                                                |
| AI Compliance Classifier    | LangChain Agent                    | Analyze logs for compliance violations            | Normalize Log Data                       | Check for Violations, OpenAI GPT-4 | ## AI Content Analyzer **What:** Processes the merged data using the Claude API. **Why:** Extracts key insights and structures communication |
| OpenAI GPT-4               | LangChain LM Chat OpenAI            | Executes GPT-4 AI model inference                 | AI Compliance Classifier (languageModel) | Structured Violation Output        | See above                                                                                                                        |
| Structured Violation Output | LangChain Output Parser Structured  | Parses AI output to structured JSON                | OpenAI GPT-4                           | AI Compliance Classifier (outputParser) | See above                                                                                                                        |
| Check for Violations        | If                                 | Routes flow based on presence of violations       | AI Compliance Classifier                 | Archive Evidence to Storage (if true) | ## Format & Publish **What:** Reviews output quality and applies standardized formatting for distribution. **Why:** Ensures consistency, clarity |
| Archive Evidence to Storage | HTTP Request                        | Store audit evidence and violation data           | Check for Violations                    | Prepare Audit Report Data          | See above                                                                                                                        |
| Prepare Audit Report Data   | Set                                | Calculate violation counts and report metadata    | Archive Evidence to Storage              | Generate Audit Report HTML         |                                                                                                                                |
| Generate Audit Report HTML  | Code                               | Generate detailed HTML audit report                | Prepare Audit Report Data                | Send Audit Report Email            | See above                                                                                                                        |
| Send Audit Report Email     | Gmail                              | Email the audit report to configured recipients   | Generate Audit Report HTML               | Create Corrective Action Tasks     |                                                                                                                                |
| Create Corrective Action Tasks | HTTP Request                    | Create remediation tasks in task management system| Send Audit Report Email                 | None                            |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Compliance Audit" node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 2 AM.

2. **Create "Workflow Configuration" node:**  
   - Type: Set  
   - Define variables:  
     - `serverLogsUrl`: URL string placeholder for server logs API  
     - `apiGatewayLogsUrl`: URL string placeholder for API gateway logs API  
     - `hrSystemUrl`: URL string placeholder for HR system API  
     - `financeSystemUrl`: URL string placeholder for finance system API  
     - `storageApiUrl`: URL string placeholder for evidence storage API  
     - `taskManagementUrl`: URL string placeholder for task management API  
     - `auditReportRecipients`: Comma-separated emails for report recipients  
     - `complianceStandards`: Set as "GDPR,SOC 2,ISO 27001"

3. **Connect "Schedule Compliance Audit" output to "Workflow Configuration" input.**

4. **Create four HTTP Request nodes:**

   - **Pull Server Logs:**  
     - URL: `={{ $('Workflow Configuration').first().json.serverLogsUrl }}`  
     - Method: GET  
     - Query parameters:  
       - `startDate` = `{{$now.minus({ days: 1 }).toISO()}}`  
       - `endDate` = `{{$now.toISO()}}`  

   - **Pull API Gateway Logs:**  
     - URL: from `apiGatewayLogsUrl` in Workflow Configuration  
     - Same method and query parameters as above.

   - **Pull HR System Data:**  
     - URL: from `hrSystemUrl`  
     - Same query parameters.

   - **Pull Finance System Data:**  
     - URL: from `financeSystemUrl`  
     - Same query parameters.

5. **Connect "Workflow Configuration" output to all four HTTP Request nodes in parallel.**

6. **Create "Combine All Logs" node:**  
   - Type: Aggregate  
   - Aggregate all incoming data arrays into a single array.

7. **Connect all four Pull Data HTTP nodes outputs to "Combine All Logs" inputs.**

8. **Create "Normalize Log Data" node:**  
   - Type: Set  
   - Assignments:  
     - `timestamp`: Current ISO datetime `$now.toISO()`  
     - `source`: `$json.source` or "unknown"  
     - `logData`: entire JSON of combined logs  
     - `auditDate`: formatted as `$now.toFormat('yyyy-MM-dd')`  

9. **Connect "Combine All Logs" output to "Normalize Log Data" input.**

10. **Create "AI Compliance Classifier" node:**  
    - Type: LangChain Agent  
    - Parameters:  
      - Text input: `{{$json.logData}}`  
      - System message: Detailed compliance auditor instructions specifying GDPR, SOC 2, ISO 27001 violation types, severity classification, evidence extraction, and corrective actions.  
      - Prompt type: define  
      - Enable output parser.

11. **Connect "Normalize Log Data" output to "AI Compliance Classifier" input.**

12. **Create "OpenAI GPT-4" node:**  
    - Type: LangChain LM Chat OpenAI  
    - Model: `gpt-4o`  
    - Credentials: Configure with valid OpenAI API key.  

13. **Link "AI Compliance Classifier" language model output to "OpenAI GPT-4" input.**

14. **Create "Structured Violation Output" node:**  
    - Type: LangChain Output Parser Structured  
    - Input schema: JSON schema defining `hasViolations`, `violations` array with detailed fields, and `summary`.  

15. **Connect "OpenAI GPT-4" output to "Structured Violation Output" input.**

16. **Connect "Structured Violation Output" back to "AI Compliance Classifier" output parser input.**

17. **Create "Check for Violations" node:**  
    - Type: If  
    - Condition: `$json.hasViolations === true`  

18. **Connect "AI Compliance Classifier" output to "Check for Violations" input.**

19. **Create "Archive Evidence to Storage" node:**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: from Workflow Configuration `storageApiUrl`  
    - JSON Body includes:  
      - `auditDate`, `timestamp` from Normalize Log Data  
      - `violations`, `summary`, `rawLogs` from AI output  
    - Send body as JSON.  

20. **Connect "Check for Violations" true output to "Archive Evidence to Storage" input.**

21. **Create "Prepare Audit Report Data" node:**  
    - Type: Set  
    - Assignments:  
      - `reportTitle`: "Compliance Audit Report - " + auditDate  
      - `totalViolations`: length of violations array  
      - Counts of violations by severity: Critical, High, Medium, Low (using JS filter expressions)  

22. **Connect "Archive Evidence to Storage" output to "Prepare Audit Report Data" input.**

23. **Create "Generate Audit Report HTML" node:**  
    - Type: Code  
    - JavaScript code builds a styled HTML report with:  
      - Executive summary with violation counts and compliance rate  
      - Sectioned violation lists by severity with description, evidence, corrective action  
      - Detailed violation table  
      - Recommendations based on violation presence and standards  
      - Footer with generation info  

24. **Connect "Prepare Audit Report Data" output to "Generate Audit Report HTML" input.**

25. **Create "Send Audit Report Email" node:**  
    - Type: Gmail  
    - Recipients: from Workflow Configuration `auditReportRecipients`  
    - Subject: from report title  
    - Message: HTML content from previous node  
    - Credentials: Configure Gmail OAuth2 with appropriate account.  

26. **Connect "Generate Audit Report HTML" output to "Send Audit Report Email" input.**

27. **Create "Create Corrective Action Tasks" node:**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: from Workflow Configuration `taskManagementUrl`  
    - JSON Body maps violations to tasks with fields: title, description, priority, action, and dueDate (calculated based on severity)  
    - Send as JSON.  

28. **Connect "Send Audit Report Email" output to "Create Corrective Action Tasks" input.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Customizable integration sources allow adding/removing triggers and adjusting AI prompts as needed.      | Sticky Note1 near workflow start                                                                   |
| Prerequisites include access to Slack, Teams, Gmail, GitHub, Anthropic API, Notion, and n8n hosted instance. | Sticky Note3                                                                                        |
| Setup involves connecting required credentials, configuring monitored sources and setting schedule triggers. | Sticky Note4                                                                                        |
| Workflow centralizes communication data to improve knowledge management and reduce manual aggregation.   | Sticky Note5                                                                                        |
| AI content analysis uses Claude API concepts adapted here via LangChain and OpenAI GPT-4 integration.    | Sticky Note8                                                                                        |
| Generated reports ensure clarity through formatting and are distributed via email with task creation for remediation. | Sticky Note9                                                                                        |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.