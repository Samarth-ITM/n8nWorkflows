Generate Weekly Workflow Analytics Reports with n8n API & Email Delivery

https://n8nworkflows.xyz/workflows/generate-weekly-workflow-analytics-reports-with-n8n-api---email-delivery-10226


# Generate Weekly Workflow Analytics Reports with n8n API & Email Delivery

### 1. Workflow Overview

This workflow automates the generation and delivery of weekly analytics reports summarizing the executions of all workflows within an n8n instance. It targets users who want to monitor their automation performance, error rates, and general workflow activity on a recurring basis without manual intervention.

**Logical Blocks:**

- **1.1 Scheduled Trigger**  
  Automatically initiates the workflow every 7 days to generate the report.

- **1.2 Data Retrieval**  
  Fetches all previous workflow executions and all defined workflows from the n8n instance.

- **1.3 Data Preparation and Merging**  
  Normalizes and combines execution data with workflow metadata for comprehensive analysis.

- **1.4 Filtering Recent Executions**  
  Filters execution records to include only those from the last 7 days.

- **1.5 Report Generation**  
  Processes filtered data to compute statistics by workflow and generates a styled HTML report summarizing execution counts and runtimes by status.

- **1.6 Email Delivery**  
  Sends the generated HTML report via email using Microsoft Outlook and Gmail nodes.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow run at a fixed 7-day interval.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger` (n8n base node)  
    - Configuration: Set to trigger every 7 days (`daysInterval: 7`)  
    - Input: None (starts the workflow)  
    - Output: Event trigger to start the workflow execution  
    - Edge Cases: Misconfiguration could cause overlapping runs; ensure the interval fits system capacity.  
    - Version: 1.1

---

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves all historical execution data and all workflows currently defined in n8n to prepare for analytics.

- **Nodes Involved:**  
  - Get all previous executions  
  - Get all Workflows

- **Node Details:**  
  - **Get all previous executions**  
    - Type: `n8n` (internal n8n API node)  
    - Configuration:  
      - Resource: `execution`  
      - Return All: `true` (fetches all executions without pagination)  
      - Filters: none (fetches all executions)  
    - Input: Trigger from Schedule Trigger node  
    - Output: Array of execution records including workflow IDs, status, timestamps, etc.  
    - Edge Cases: Large datasets could cause performance issues; ensure adequate permissions to access executions.  
    - Version: 1

  - **Get all Workflows**  
    - Type: `n8n` (internal n8n API node)  
    - Configuration:  
      - No filters or options (fetches all workflows)  
    - Input: Trigger from Schedule Trigger node  
    - Output: Array of workflows with metadata including IDs and names  
    - Edge Cases: Large number of workflows may impact performance; permissions required.  
    - Version: 1

---

#### 1.3 Data Preparation and Merging

- **Overview:**  
  Transforms and merges workflow and execution data by mapping workflow IDs for accurate correlation.

- **Nodes Involved:**  
  - Edit Field ID to workflowId  
  - Merge

- **Node Details:**  
  - **Edit Field ID to workflowId**  
    - Type: `Set` node  
    - Role: Renames the field `id` to `workflowId` to harmonize keys for merging  
    - Configuration:  
      - Assign `workflowId` = `={{ $json.id }}`  
      - Includes all other fields unchanged  
    - Input: Output from Get all Workflows  
    - Output: Workflows with `workflowId` field replacing `id`  
    - Edge Cases: Fails if `id` is missing in input JSON  
    - Version: 3.4

  - **Merge**  
    - Type: `Merge` node  
    - Role: Combines execution data (input 1) with workflow data (input 2) by matching on `workflowId`  
    - Configuration:  
      - Mode: Combine  
      - Join Mode: Enrich Input 2 (workflow data enriched with execution data)  
      - Merge By Fields: `workflowId` matched with `workflowId`  
      - Clash Handling: Prefer last (prioritizes execution data values on conflicts)  
    - Inputs:  
      - Input 1: Executions (from Get all previous executions)  
      - Input 2: Workflows (from Edit Field ID to workflowId)  
    - Output: Combined dataset with enriched execution and workflow information  
    - Edge Cases: Missing or mismatched `workflowId` fields cause incomplete merges  
    - Version: 3.2

---

#### 1.4 Filtering Recent Executions

- **Overview:**  
  Filters the combined execution data to include only those executions started within the past 7 days.

- **Nodes Involved:**  
  - Filter executions last week

- **Node Details:**  
  - **Filter executions last week**  
    - Type: `Filter` node  
    - Role: Filters input data by date condition on `startedAt` field  
    - Configuration:  
      - Condition: `startedAt` after date exactly 7 days before current date  
      - Operator: DateTime after  
      - Left Value: `={{ $json.startedAt }}`  
      - Right Value: `={{ DateTime.now().minus({ days: 7 }) }}`  
      - Case sensitive: true  
    - Input: Output from Merge node (combined executions and workflow data)  
    - Output: Only executions started in the last 7 days  
    - Edge Cases:  
      - No matching executions in the last 7 days yields empty output; downstream nodes must handle empty datasets gracefully  
      - Date format must be ISO 8601 or compatible for correct filtering  
    - Version: 2

---

#### 1.5 Report Generation

- **Overview:**  
  Processes filtered execution data to compute statistics (counts and runtimes) by workflow and status (error, success, waiting), and generates a styled HTML report with this information.

- **Nodes Involved:**  
  - Prepare html report

- **Node Details:**  
  - **Prepare html report**  
    - Type: `Code` node (JavaScript)  
    - Role:  
      - Groups executions by workflow  
      - Calculates counts and total/average runtime per status category  
      - Formats runtime durations into human-readable strings  
      - Constructs an HTML document styled with CSS including:  
        - Header with date range  
        - Summary section with total error, success, waiting counts  
        - Detailed table listing per-workflow stats with badges and runtime info  
      - Generates a subject line for the email with the date range  
    - Input: Filtered executions (with workflow metadata) from Filter executions last week  
    - Output: JSON with two keys:  
      - `html`: the full HTML report string  
      - `subject`: email subject line containing the report date range  
    - Edge Cases:  
      - Empty input results in a report with no data rows but full structure  
      - Runtime calculation handles missing stop times (runtime zeroed)  
      - Date locale fixed to US English for consistent formatting  
    - Version: 2

---

#### 1.6 Email Delivery

- **Overview:**  
  Sends the generated HTML report via email through both Microsoft Outlook and Gmail services for redundancy or user preference.

- **Nodes Involved:**  
  - Send a message outlook  
  - Send a message gmail

- **Node Details:**  
  - **Send a message outlook**  
    - Type: `Microsoft Outlook` node  
    - Role: Sends email using Outlook API  
    - Configuration:  
      - Subject: `={{ $json.subject }}` (dynamic from report node)  
      - Body Content: `={{ $json.html }}` (HTML report)  
      - Body Content Type: `html`  
      - Recipient(s): Must be set in the node (e.g., `wessel@bulte.org` as example)  
    - Input: Output from Prepare html report  
    - Output: Confirmation or error response of email delivery  
    - Edge Cases:  
      - Authentication errors if OAuth2 credentials invalid or expired  
      - Delivery failures due to invalid recipient address or network issues  
    - Version: 2

  - **Send a message gmail**  
    - Type: `Gmail` node  
    - Role: Sends email using Gmail API  
    - Configuration:  
      - Subject: `={{ $json.subject }}`  
      - Message Body: `={{ $json.html }}` (HTML formatted)  
      - OAuth2 credentials must be configured  
    - Input: Output from Prepare html report  
    - Output: Confirmation or error response of email delivery  
    - Edge Cases:  
      - Disabled by default (must be enabled to send)  
      - API quota limits or authentication failures possible  
      - HTML formatting may render differently depending on client  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                          | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                                  |
|----------------------------|----------------------------|----------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger            | Initiate workflow every 7 days         | None                       | Get all previous executions, Get all Workflows | Purpose: This node initiates a scheduled trigger, allowing workflows to run at specified intervals.          |
| Get all previous executions| n8n API node               | Fetch all previous executions           | Schedule Trigger           | Merge                         | Purpose: Retrieve all previous executions; no filters applied.                                               |
| Get all Workflows          | n8n API node               | Fetch all workflows metadata            | Schedule Trigger           | Edit Field ID to workflowId   | Purpose: Fetch all workflows; no filters applied.                                                            |
| Edit Field ID to workflowId| Set                        | Rename `id` to `workflowId` field       | Get all Workflows          | Merge                         | Purpose: Updates field `id` to `workflowId` for merging datasets.                                            |
| Merge                     | Merge                      | Combine executions with workflow data   | Get all previous executions, Edit Field ID to workflowId | Filter executions last week     | Purpose: Combines two data sets based on `workflowId` to enrich data.                                        |
| Filter executions last week| Filter                     | Include only executions from last 7 days| Merge                      | Prepare html report           | Purpose: Filters executions started within the last week based on `startedAt` datetime.                      |
| Prepare html report       | Code                       | Generate HTML report summarizing stats | Filter executions last week| Send a message outlook, Send a message gmail | Purpose: Generate styled HTML report on workflow executions last 7 days with counts and runtimes.             |
| Send a message outlook    | Microsoft Outlook          | Send report email via Outlook           | Prepare html report        | None                         | Purpose: Send email through Outlook with HTML content; configure recipients and OAuth2 credentials.          |
| Send a message gmail      | Gmail                      | Send report email via Gmail              | Prepare html report        | None                         | Purpose: Send email through Gmail with HTML content; currently disabled; configure OAuth2 credentials.      |
| Sticky Note - Overview    | Sticky Note                | Documentation overview                   | None                       | None                         | Detailed overview of workflow purpose and structure.                                                        |
| Sticky Note - Send a message gmail | Sticky Note        | Documentation for Gmail node             | None                       | None                         | Explains Gmail node purpose, inputs, outputs, and tips; notes node disabled by default.                      |
| Sticky Note - Send a message outlook | Sticky Note       | Documentation for Outlook node           | None                       | None                         | Explains Outlook node purpose, inputs, outputs, and tips; notes OAuth2 and recipient validation.             |
| Sticky Note - Prepare html report | Sticky Note           | Documentation for HTML report node       | None                       | None                         | Describes report generation logic and input/output expectations.                                            |
| Sticky Note - Filter executions last week | Sticky Note     | Documentation for filtering last week executions | None                   | None                         | Details filter condition and tip for date formatting and empty outputs.                                     |
| Sticky Note - Merge       | Sticky Note                | Documentation for merge node             | None                       | None                         | Explains purpose of merging datasets by `workflowId` field.                                                 |
| Sticky Note - Edit Field ID to workflowId | Sticky Note      | Documentation for set node renaming field | None                      | None                         | Describes renaming `id` to `workflowId` and preserving other fields.                                        |
| Sticky Note - Get all previous executions | Sticky Note      | Documentation for fetching executions    | None                       | None                         | Details purpose and configuration to get all previous executions.                                           |
| Sticky Note - Get all Workflows | Sticky Note           | Documentation for fetching workflows     | None                       | None                         | Details purpose and configuration to get all workflows metadata.                                            |
| Sticky Note - Schedule Trigger | Sticky Note             | Documentation for schedule trigger        | None                       | None                         | Describes initiating workflow at scheduled intervals.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to every 7 days (`daysInterval: 7`)  
   - Position: Start of the workflow

2. **Create a Get all previous executions node:**  
   - Type: n8n (internal API)  
   - Resource: `execution`  
   - Return All: `true`  
   - No filters  
   - Connect input from Schedule Trigger (main output)

3. **Create a Get all Workflows node:**  
   - Type: n8n (internal API)  
   - No filters or options  
   - Connect input from Schedule Trigger (main output)

4. **Create an Edit Field ID to workflowId node:**  
   - Type: Set  
   - Add an assignment:  
     - Name: `workflowId`  
     - Value: Expression `={{ $json.id }}`  
   - Enable "Include Other Fields" to keep all other workflow data  
   - Connect input from Get all Workflows

5. **Create a Merge node:**  
   - Type: Merge  
   - Mode: Combine  
   - Join Mode: Enrich Input 2  
   - Merge By Fields: field1: `workflowId`, field2: `workflowId`  
   - Clash Handling: Prefer last  
   - Connect input 1 from Get all previous executions  
   - Connect input 2 from Edit Field ID to workflowId

6. **Create a Filter executions last week node:**  
   - Type: Filter  
   - Add condition:  
     - Operator: DateTime after  
     - Left Value: `={{ $json.startedAt }}`  
     - Right Value: `={{ DateTime.now().minus({ days: 7 }) }}`  
     - Case Sensitive: true  
   - Connect input from Merge node

7. **Create a Prepare html report node:**  
   - Type: Code  
   - Paste JavaScript code that:  
     - Calculates date range (last 7 days)  
     - Groups executions by workflow and status  
     - Calculates counts, total and average runtimes  
     - Generates an HTML report with styling and summary table  
     - Outputs JSON with keys: `html` (report) and `subject` (email subject)  
   - Connect input from Filter executions last week

8. **Create a Send a message outlook node:**  
   - Type: Microsoft Outlook  
   - Set subject: `={{ $json.subject }}`  
   - Set bodyContent: `={{ $json.html }}`  
   - Set bodyContentType: `html`  
   - Add recipient email(s) in Additional Fields or "To Recipients"  
   - Configure Microsoft Outlook OAuth2 credentials  
   - Connect input from Prepare html report

9. **Create a Send a message gmail node:** (optional)  
   - Type: Gmail  
   - Set subject: `={{ $json.subject }}`  
   - Set message: `={{ $json.html }}` (HTML format)  
   - Configure Gmail OAuth2 credentials  
   - This node is disabled by default; enable to use  
   - Connect input from Prepare html report

10. **Add Sticky Notes:** (optional but recommended for documentation)  
    - Add notes describing the purpose and configuration of each node as per the sticky notes in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                     |
|------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Use ISO 8601 date format for all timestamps to ensure correct filtering and sorting in the workflow. | Date handling is critical in Filter executions last week and report generation nodes.                               |
| Gmail node is disabled by default; enable only if proper OAuth2 credentials are configured.           | Prevents accidental email spamming or failure without credentials.                                                 |
| Microsoft Outlook node requires valid OAuth2 credentials and a correctly set recipient address.      | Authentication failures or invalid recipient emails will cause delivery errors.                                     |
| The HTML report is styled with embedded CSS for better email client compatibility and readability.   | Styling ensures clear visualization of workflow execution statistics.                                              |
| Large numbers of workflows or executions may affect performance; consider filtering or batching data. | Workflow designed for small to medium sized n8n instances; scaling may require optimization.                        |
| The report subject uses the date range in US English locale format.                                  | Locales may be changed if needed in the `formatDate` helper function inside the code node.                         |
| Keep schedule trigger frequency aligned with system load to avoid overlapping runs.                  | For example, avoid scheduling during peak hours or heavy processing times.                                         |
| This workflow relies on internal n8n API nodes to fetch executions and workflows, requiring appropriate permissions. | Permissions must be verified in n8n settings to avoid authorization errors.                                        |

---

This document fully describes the workflow "Generate Weekly Workflow Analytics Reports with n8n API & Email Delivery," enabling advanced users and AI agents to understand, reproduce, maintain, and extend the workflow with confidence.