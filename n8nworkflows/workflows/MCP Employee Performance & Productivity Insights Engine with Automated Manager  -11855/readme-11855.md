MCP Employee Performance & Productivity Insights Engine with Automated Manager  

https://n8nworkflows.xyz/workflows/mcp-employee-performance---productivity-insights-engine-with-automated-manager---11855


# MCP Employee Performance & Productivity Insights Engine with Automated Manager  

---

### 1. Workflow Overview

This workflow, titled **"MCP Employee Performance & Productivity Insights Engine with Automated Manager,"** automates the weekly analysis of employee performance and productivity by aggregating and processing data from multiple business systems. It targets engineering managers and team leads who need real-time insights into team velocity, code quality, meeting effectiveness, and customer engagement, with automated alerting and task creation to assist managerial follow-up.

The workflow’s logic is grouped into the following functional blocks:

- **1.1 Scheduled Trigger & Configuration:** Initiates the workflow on a weekly schedule and sets key configuration parameters such as API endpoints and analysis timeframe.

- **1.2 Data Acquisition:** Fetches performance-related data concurrently from four main sources:
  - Project Management (PM) tool
  - Code Repository (e.g., GitHub/GitLab)
  - Meeting Logs (e.g., Google Calendar, Zoom)
  - Customer Relationship Management (CRM) system

- **1.3 Data Aggregation & Normalization:** Combines and normalizes all fetched data into a unified structure suitable for AI processing.

- **1.4 AI-Powered Performance Analysis:** Uses a LangChain agent with OpenAI GPT-4 to analyze combined data and generate structured employee performance insights, including strengths, bottlenecks, workload status, and recommendations.

- **1.5 Structured Output Parsing & Processing:** Parses the AI output into structured JSON, processes it to detect bottlenecks and overload conditions, and prepares summary data.

- **1.6 Manager Notification & Task Automation:** For employees exceeding workload thresholds or with identified issues, creates follow-up tasks in the PM tool and sends detailed performance summary emails to the manager.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Configuration

**Overview:**  
This block triggers the workflow every Monday at 9 AM and sets essential configuration parameters such as API URLs, manager email, analysis timeframe, and workload thresholds.

**Nodes Involved:**  
- Weekly Performance Analysis Trigger  
- Workflow Configuration

**Node Details:**

- **Weekly Performance Analysis Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow execution weekly on Mondays at 9:00 AM.  
  - *Configuration:* Interval set to weeks, triggering specifically on day 1 (Monday) at hour 9.  
  - *Input:* None  
  - *Output:* Triggers the next node on schedule  
  - *Potential Failures:* Misconfigured timezone or schedule could cause missed triggers.

- **Workflow Configuration**  
  - *Type:* Set  
  - *Role:* Defines workflow parameters such as API URLs, manager email, analysis timeframe (7 days), and overload threshold (40 units).  
  - *Configuration:* Static string assignments with placeholders for actual URLs and email address.  
  - *Variables:*  
    - `pmToolApiUrl` (string)  
    - `codeRepoApiUrl` (string)  
    - `meetingLogsApiUrl` (string)  
    - `crmApiUrl` (string)  
    - `managerEmail` (string)  
    - `analysisTimeframe` (number, default 7)  
    - `overloadThreshold` (number, default 40)  
  - *Input:* Trigger node  
  - *Output:* Passes configuration data downstream  
  - *Edge cases:* Missing or invalid URLs or email would cause downstream API calls or email sending to fail.

---

#### 2.2 Data Acquisition

**Overview:**  
Fetches data in parallel from four external sources representing different aspects of employee activities, parameterized by the analysis timeframe.

**Nodes Involved:**  
- Fetch PM Tool Data  
- Fetch Code Repo Data  
- Fetch Meeting Logs  
- Fetch CRM Activity

**Node Details:**

- **Fetch PM Tool Data**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves project management data filtered by the analysis timeframe.  
  - *Configuration:* URL from config node, sends GET request with query parameter `timeframe` (days).  
  - *Input:* Workflow Configuration node  
  - *Output:* JSON data representing PM tool metrics  
  - *Failures:* API endpoint unreachable, authentication issues, malformed responses.

- **Fetch Code Repo Data**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves code repository data (commits, PRs) since the start of the timeframe.  
  - *Configuration:* URL from config node, query parameter `since` set to ISO date (now minus timeframe days).  
  - *Input:* Workflow Configuration node  
  - *Output:* JSON data with commit and review metrics  
  - *Failures:* Token expiration, rate limits, invalid date format.

- **Fetch Meeting Logs**  
  - *Type:* HTTP Request  
  - *Role:* Fetches meeting attendance and duration logs since the timeframe start.  
  - *Configuration:* URL from config node, query parameter `timeMin` set to ISO date (now minus timeframe days).  
  - *Input:* Workflow Configuration node  
  - *Output:* Meeting data JSON  
  - *Failures:* API quota exceeded, connectivity issues.

- **Fetch CRM Activity**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves CRM customer interaction data for the analysis timeframe.  
  - *Configuration:* URL from config node, query parameter `days` with timeframe value.  
  - *Input:* Workflow Configuration node  
  - *Output:* CRM activity data JSON  
  - *Failures:* Authentication errors, incomplete data.

---

#### 2.3 Data Aggregation & Normalization

**Overview:**  
Combines all data fetched from the above sources into a single aggregated dataset for unified analysis.

**Nodes Involved:**  
- Combine All Data Sources

**Node Details:**

- **Combine All Data Sources**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all incoming item data from multiple sources into one array under the field `allData`.  
  - *Configuration:* Aggregates all input items into a single JSON object field.  
  - *Input:* Outputs from all four Fetch nodes connected in parallel  
  - *Output:* Single item containing combined data array  
  - *Edge Cases:* Inconsistent data schemas or empty source results could affect subsequent AI analysis.

---

#### 2.4 AI-Powered Performance Analysis

**Overview:**  
Analyzes the combined data using an AI agent powered by OpenAI GPT-4, generating structured insights on employee performance, bottlenecks, workload, and recommendations.

**Nodes Involved:**  
- Performance Analysis Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **Performance Analysis Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Sends combined data as prompt to AI, specifying detailed instructions to analyze performance, strengths, bottlenecks, workload status, and recommendations per employee.  
  - *Configuration:*  
    - System message defines agent role and analysis scope.  
    - Input text is entire combined data (`allData`).  
    - Output expected as structured JSON.  
  - *Input:* Combined data from aggregation node  
  - *Output:* Raw AI-generated JSON text  
  - *Edge Cases:* AI rate limits, unexpected output format, incomplete data handling.

- **OpenAI Chat Model**  
  - *Type:* OpenAI GPT-4 Chat Model  
  - *Role:* Processes the prompt from the agent and returns the AI-generated response.  
  - *Configuration:* Model set to "gpt-4.1-mini", requires valid OpenAI API credentials.  
  - *Input:* Text prompt from agent  
  - *Output:* AI response text  
  - *Failures:* API quota exhaustion, token limits, credential misconfiguration.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI response into a strict JSON schema to ensure data validity and structure.  
  - *Configuration:* Manual JSON schema defining employeeName, performanceScore, workloadLevel, strengths, bottlenecks, recommendations, priorityActions.  
  - *Input:* AI response text  
  - *Output:* Parsed structured data ready for further processing  
  - *Edge Cases:* Parsing errors if AI response deviates from schema, schema misalignment.

---

#### 2.5 Structured Output Processing & Bottleneck Detection

**Overview:**  
Processes structured AI output to identify employees with bottlenecks or high workload and prepare data for notification and task creation.

**Nodes Involved:**  
- Process Performance Data  
- Check for Bottlenecks or Overload

**Node Details:**

- **Process Performance Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Iterates over AI output, checks for presence of bottlenecks or overload, adds a `hasIssues` boolean and timestamp to each employee's record.  
  - *Key Logic:* Flags employees if bottlenecks exist or workload is high/overloaded.  
  - *Input:* Parsed structured AI output  
  - *Output:* Array of processed employee performance data objects  
  - *Failures:* Code errors if input format changes or missing fields.

- **Check for Bottlenecks or Overload**  
  - *Type:* If (Conditional)  
  - *Role:* Routes only employees with workloadLevel "high" or "overloaded" to subsequent notification and task creation.  
  - *Conditions:* workloadLevel equals "high" OR "overloaded"  
  - *Input:* Processed performance data  
  - *Output:* Passes filtered data forward  
  - *Edge Cases:* If workloadLevel is missing or unexpected string, condition may fail.

---

#### 2.6 Manager Notification & Task Automation

**Overview:**  
For employees flagged with performance issues, creates follow-up tasks in the PM tool and sends a formatted performance summary email to the manager.

**Nodes Involved:**  
- Create Manager Follow-up Tasks  
- Send Performance Summary to Manager  
- Create Tasks in PM Tool

**Node Details:**

- **Create Manager Follow-up Tasks**  
  - *Type:* Set  
  - *Role:* Builds task data including title, description (with detailed metrics and recommendations), priority, due date (3 days from now), and embeds employee data for downstream use.  
  - *Input:* Employees flagged by conditional node  
  - *Output:* Task data object for email and API calls  
  - *Edge Cases:* Missing employeeName or arrays could cause template rendering issues.

- **Send Performance Summary to Manager**  
  - *Type:* Gmail Node (OAuth2)  
  - *Role:* Sends an HTML email to the configured manager email with a detailed breakdown of performance score, workload level, strengths, bottlenecks, recommendations, and priority actions.  
  - *Configuration:* Uses Gmail OAuth2 credentials; email subject includes employee name; content dynamically generated from task data.  
  - *Input:* Task data from Set node  
  - *Output:* Email sent status  
  - *Failures:* OAuth token expiry, email quota limits, invalid email address.

- **Create Tasks in PM Tool**  
  - *Type:* HTTP Request  
  - *Role:* Creates a follow-up task in the project management tool via API POST request, using task details from the Set node.  
  - *Configuration:* URL formed by appending `/tasks` to PM tool API URL; POST method with JSON body including title, description, priority, due date.  
  - *Input:* Task data from Set node  
  - *Output:* API response status  
  - *Failures:* API authentication, validation errors, network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                          | Input Node(s)                   | Output Node(s)                         | Sticky Note                                                                                                        |
|-------------------------------|----------------------------------|----------------------------------------|--------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Weekly Performance Analysis Trigger | Schedule Trigger                 | Starts workflow on weekly schedule     | None                           | Workflow Configuration                | ## Fetch Data from Multiple Sources\n### What\nFetch PM Tool, Code Repo, Meeting, and CRM data via four parallel API calls.\n### Why\nConsolidates performance metrics from all key business systems into a single workflow. |
| Workflow Configuration         | Set                              | Sets API URLs, manager email, params   | Weekly Performance Analysis Trigger | Fetch PM Tool Data, Fetch Code Repo Data, Fetch Meeting Logs, Fetch CRM Activity | ## Setup Steps\n1. Configure credentials: PM Tool API key\n2. Set the OpenAI API key.\n3. Connect your Gmail account \n4. In the Workflow Configuration node, adjust API endpoints and polling intervals.\n5. Map data field names \n6. Test data fetch nodes using sample queries. |
| Fetch PM Tool Data             | HTTP Request                     | Fetch PM tool data for timeframe       | Workflow Configuration         | Combine All Data Sources               | ## Fetch Data from Multiple Sources\n### What\nFetch PM Tool, Code Repo, Meeting, and CRM data via four parallel API calls.\n### Why\nConsolidates performance metrics from all key business systems into a single workflow. |
| Fetch Code Repo Data           | HTTP Request                     | Fetch code repo data since timeframe   | Workflow Configuration         | Combine All Data Sources               | Same as above                                                                                                      |
| Fetch Meeting Logs             | HTTP Request                     | Fetch meeting logs since timeframe     | Workflow Configuration         | Combine All Data Sources               | Same as above                                                                                                      |
| Fetch CRM Activity             | HTTP Request                     | Fetch CRM activity data for timeframe  | Workflow Configuration         | Combine All Data Sources               | Same as above                                                                                                      |
| Combine All Data Sources       | Aggregate                       | Combine and normalize all fetched data | Fetch PM Tool Data, Fetch Code Repo Data, Fetch Meeting Logs, Fetch CRM Activity | Performance Analysis Agent                | ## Data Combination and Normalization\n### What\nCombine and normalize all data sources into a unified format.\n### Why\nCreates a consistent data structure for reliable AI analysis. |
| Performance Analysis Agent     | LangChain Agent                 | AI analysis of combined data            | Combine All Data Sources       | Process Performance Data               | ## Parse AI Output\n### What\nParse structured output from OpenAI analysis using a custom JSON parser.\n### Why\nExtracts actionable insights, such as bottlenecks and trends, into a structured format. |
| OpenAI Chat Model             | OpenAI Chat Model               | AI language model processing            | Performance Analysis Agent     | Structured Output Parser               | Same as above                                                                                                      |
| Structured Output Parser       | LangChain Output Parser         | Parse AI response into structured JSON | OpenAI Chat Model             | Performance Analysis Agent             | Same as above                                                                                                      |
| Process Performance Data       | Code                           | Process structured AI output, flag issues | Performance Analysis Agent     | Check for Bottlenecks or Overload      | ## Evaluate Team Capacity and Give Notification\n### What\nEvaluate team capacity thresholds and bottleneck indicators. Provide notification\n### Why\nDetermines if manager intervention is required for workload management and provide next action. |
| Check for Bottlenecks or Overload | If                            | Filter employees with high workload or bottlenecks | Process Performance Data       | Create Manager Follow-up Tasks         | Same as above                                                                                                      |
| Create Manager Follow-up Tasks | Set                            | Create task details for manager follow-up | Check for Bottlenecks or Overload | Send Performance Summary to Manager, Create Tasks in PM Tool |                                                                                                                    |
| Send Performance Summary to Manager | Gmail                         | Send detailed performance email to manager | Create Manager Follow-up Tasks | None                                  |                                                                                                                    |
| Create Tasks in PM Tool        | HTTP Request                   | Create follow-up tasks in PM tool       | Create Manager Follow-up Tasks | None                                  |                                                                                                                    |
| Sticky Note                   | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Customization\nReplace PM tool with Jira/Linear; swap OpenAI for Claude/Gemini;  \n\n## Benefits\nReduces manual performance tracking by 6+ hours weekly; provides real-time visibility into team capacity;  |
| Sticky Note1                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Prerequisites\nPM tool API access, GitHub/GitLab token, CRM credentials, OpenAI API key, Gmail OAuth connection \n\n## Use Cases\nTrack engineering team productivity weekly; identify code review bottlenecks;  |
| Sticky Note2                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Setup Steps\n1. Configure credentials: PM Tool API key\n2. Set the OpenAI API key.\n3. Connect your Gmail account \n4. In the Workflow Configuration node, adjust API endpoints and polling intervals.\n5. Map data field names \n6. Test data fetch nodes using sample queries. |
| Sticky Note3                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## How It Works\nThis workflow automates performance monitoring by aggregating data from PM tools, code repositories, meeting logs, and CRM systems. It processes team metrics using AI-powered analysis via OpenAI, identifies bottlenecks and workload issues, then creates manager follow-ups and tasks. The system runs weekly, collecting 4 data sources, combining them, analyzing trends, evaluating team capacity, and routing alerts to managers via Gmail. Managers receive structured summaries highlighting performance gaps and required actions. Target audience: Engineering managers and team leads monitoring team velocity, code quality, and capacity planning. |
| Sticky Note4                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Evaluate Team Capacity and Give Notification\n### What\nEvaluate team capacity thresholds and bottleneck indicators. Provide notification\n### Why\nDetermines if manager intervention is required for workload management and provide next action. |
| Sticky Note5                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Parse AI Output\n### What\nParse structured output from OpenAI analysis using a custom JSON parser.\n### Why\nExtracts actionable insights, such as bottlenecks and trends, into a structured format. |
| Sticky Note6                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Data Combination and Normalization\n### What\nCombine and normalize all data sources into a unified format.\n### Why\nCreates a consistent data structure for reliable AI analysis. |
| Sticky Note7                  | Sticky Note                    | Documentation and guidance notes        | None                         | None                                  | ## Fetch Data from Multiple Sources\n### What\nFetch PM Tool, Code Repo, Meeting, and CRM data via four parallel API calls.\n### Why\nConsolidates performance metrics from all key business systems into a single workflow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Weekly Performance Analysis Trigger" Node**  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Mondays at 9:00 AM.

2. **Create "Workflow Configuration" Node**  
   - Type: Set  
   - Add string fields for:  
     - `pmToolApiUrl` (set placeholder URL)  
     - `codeRepoApiUrl` (set placeholder)  
     - `meetingLogsApiUrl` (set placeholder)  
     - `crmApiUrl` (set placeholder)  
     - `managerEmail` (set placeholder email)  
   - Add number fields for:  
     - `analysisTimeframe` (default 7)  
     - `overloadThreshold` (default 40)  
   - Connect trigger node to this node.

3. **Create Four Parallel HTTP Request Nodes to Fetch Data:**

   - **"Fetch PM Tool Data"**  
     - URL: `={{ $('Workflow Configuration').first().json.pmToolApiUrl }}`  
     - Query parameter: `timeframe` = `={{ $('Workflow Configuration').first().json.analysisTimeframe }}`  
     - Method: GET  
     - Connect from Workflow Configuration.

   - **"Fetch Code Repo Data"**  
     - URL: `={{ $('Workflow Configuration').first().json.codeRepoApiUrl }}`  
     - Query parameter: `since` = `={{ $now.minus({ days: $('Workflow Configuration').first().json.analysisTimeframe }).toISO() }}`  
     - Method: GET  
     - Connect from Workflow Configuration.

   - **"Fetch Meeting Logs"**  
     - URL: `={{ $('Workflow Configuration').first().json.meetingLogsApiUrl }}`  
     - Query parameter: `timeMin` = `={{ $now.minus({ days: $('Workflow Configuration').first().json.analysisTimeframe }).toISO() }}`  
     - Method: GET  
     - Connect from Workflow Configuration.

   - **"Fetch CRM Activity"**  
     - URL: `={{ $('Workflow Configuration').first().json.crmApiUrl }}`  
     - Query parameter: `days` = `={{ $('Workflow Configuration').first().json.analysisTimeframe }}`  
     - Method: GET  
     - Connect from Workflow Configuration.

4. **Create "Combine All Data Sources" Node**  
   - Type: Aggregate  
   - Set to aggregate all input items into a single field called `allData`.  
   - Connect all four fetch nodes to this node.

5. **Create "Performance Analysis Agent" Node**  
   - Type: LangChain Agent  
   - Input parameter `text`: `={{ $json.allData }}`  
   - Configure system message to instruct AI to analyze employee performance data with detailed points on inputs and expected outputs.  
   - Enable output parser.  
   - Connect output of Combine All Data Sources to this node.

6. **Create "OpenAI Chat Model" Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: "gpt-4.1-mini"  
   - Add valid OpenAI API credentials.  
   - Connect from Performance Analysis Agent’s AI language model output.

7. **Create "Structured Output Parser" Node**  
   - Type: LangChain Structured Output Parser  
   - Define JSON schema with fields: employeeName, performanceScore, workloadLevel (enum: normal, high, overloaded), strengths (array), bottlenecks (array), recommendations (array), priorityActions (array).  
   - Connect from OpenAI Chat Model.

8. **Connect "Structured Output Parser" back to "Performance Analysis Agent" node**  
   - Connect parsed output to the agent’s output parser input.

9. **Create "Process Performance Data" Node**  
   - Type: Code  
   - JavaScript code to:  
     - Iterate over AI parsed items  
     - Add boolean `hasIssues` if bottlenecks exist or workloadLevel is "high" or "overloaded"  
     - Add current timestamp  
   - Connect from Performance Analysis Agent.

10. **Create "Check for Bottlenecks or Overload" Node**  
    - Type: If  
    - Condition: `workloadLevel` equals "high" OR "overloaded"  
    - Connect from Process Performance Data.

11. **Create "Create Manager Follow-up Tasks" Node**  
    - Type: Set  
    - Assign fields:  
      - `taskTitle`: "Follow-up: {{ $json.employeeName }} - Performance Review"  
      - `taskDescription`: detailed multiline string including performanceScore, workloadLevel, bottlenecks, recommendations, priorityActions.  
      - `taskPriority`: "high"  
      - `taskDueDate`: 3 days from now (`={{ $now.plus({ days: 3 }).toISO() }}`)  
      - `employeeData`: entire employee JSON object  
    - Connect from Check for Bottlenecks node’s true branch.

12. **Create "Send Performance Summary to Manager" Node**  
    - Type: Gmail (OAuth2)  
    - Use Gmail OAuth2 credentials  
    - Send to `={{ $('Workflow Configuration').first().json.managerEmail }}`  
    - Subject: "Employee Performance Insights - {{ $json.employeeName }}"  
    - HTML body with formatted performance summary lists for strengths, bottlenecks, recommendations, priorityActions.  
    - Connect from Create Manager Follow-up Tasks.

13. **Create "Create Tasks in PM Tool" Node**  
    - Type: HTTP Request (POST)  
    - URL: `={{ $('Workflow Configuration').first().json.pmToolApiUrl }}/tasks`  
    - Body: JSON with title, description, priority, due date from Set node.  
    - Connect from Create Manager Follow-up Tasks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Customization options include replacing the PM tool API with Jira or Linear and swapping OpenAI for other LLMs like Claude or Gemini. Benefits include saving over 6 hours of manual performance tracking weekly and providing real-time team capacity visibility.                                                                                                        | Sticky Note near top-left corner of workflow canvas                                                       |
| Prerequisites: API access credentials for PM tool, code repository (GitHub/GitLab), CRM system; OpenAI API key; Gmail OAuth connection. Use cases include weekly tracking of engineering team productivity and identifying code review bottlenecks.                                                                                                                   | Sticky Note near top-left corner of workflow canvas                                                       |
| Setup steps summarize configuring credentials, setting API keys, connecting Gmail, adjusting API endpoints, mapping data fields, and testing fetch nodes with sample queries.                                                                                                                                                                                        | Sticky Note near top-left corner of workflow canvas                                                       |
| Workflow works by automating data aggregation from four business systems, AI-powered analysis, bottleneck detection, workload evaluation, and alerting managers via email and task creation. Target users are engineering managers and team leads focusing on velocity, code quality, and capacity planning.                                                                 | Sticky Note near top-left corner of workflow canvas                                                       |
| The workflow evaluates team capacity thresholds and bottlenecks to decide if managerial intervention is required and initiates notifications accordingly.                                                                                                                                                                                                          | Sticky Note near bottleneck evaluation nodes                                                             |
| AI output is parsed into structured JSON to extract actionable insights for subsequent processing and notifications.                                                                                                                                                                                                                                              | Sticky Note near AI parsing nodes                                                                         |
| Combining and normalizing data ensures consistent structure for reliable AI analysis.                                                                                                                                                                                                                                                                            | Sticky Note near data aggregation node                                                                    |
| Data fetch nodes run in parallel to consolidate performance metrics from multiple systems into a unified dataset.                                                                                                                                                                                                                                                | Sticky Note near data fetch nodes                                                                          |

---

**Disclaimer:**  
The provided content is derived solely from an n8n automated workflow. It complies strictly with valid content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.