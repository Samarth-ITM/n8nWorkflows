Monitor and Report OKR Variance from Monday.com and Jira via Slack and Email

https://n8nworkflows.xyz/workflows/monitor-and-report-okr-variance-from-monday-com-and-jira-via-slack-and-email-9831


# Monitor and Report OKR Variance from Monday.com and Jira via Slack and Email

### 1. Workflow Overview

This workflow automates the monitoring and reporting of OKR (Objectives and Key Results) progress variance by integrating data from Monday.com and Jira, then distributing summarized insights via Slack and Email. It supports product teams, engineering managers, and leadership in tracking strategic objectives with real-time updates.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Scheduling:** Periodic initiation of the sync process.
- **1.2 Monday.com Data Retrieval & Mapping:** Fetching OKRs and standardizing data.
- **1.3 Jira Epic Data Fetching & Normalization:** Retrieving and flattening Jira epic details.
- **1.4 Data Joining & Variance Calculation:** Merging Monday and Jira data, then computing progress variance.
- **1.5 Monday.com Update:** Writing back computed metrics to Monday.com.
- **1.6 Aggregation & Reporting:** Consolidating results and sending reports to Slack and Email.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Scheduling

- **Overview:** Initiates the workflow on a scheduled basis, typically daily or weekly, to maintain up-to-date OKR tracking.
- **Nodes Involved:**  
  - Daily OKR Sync Trigger  
  - Note: Schedule Setup  

- **Node Details:**

  - **Daily OKR Sync Trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow execution based on defined interval.  
    - Configuration: Runs at an interval set by the user (default is daily).  
    - Connections: Output triggers "Fetch OKRs from Monday.com."  
    - Edge Cases: Misconfigured schedule could cause missed runs or excessive API calls. Timezone misalignment might cause unexpected trigger times.

  - **Note: Schedule Setup**  
    - Type: Sticky Note  
    - Role: Provides user instructions on setting the schedule trigger interval and best practices.  
    - Edge Cases: None (documentation only).

#### 1.2 Monday.com Data Retrieval & Mapping

- **Overview:** Fetches OKR items from Monday.com, extracts and standardizes key fields into a consistent schema for further processing.
- **Nodes Involved:**  
  - Fetch OKRs from Monday.com  
  - Note: Monday Board Config  
  - Note: Field Mapping Logic  
  - Map Monday Fields → Standard KR Schema  
  - Note: Epic Splitting Logic  
  - Split KR → Epics Mapper  

- **Node Details:**

  - **Fetch OKRs from Monday.com**  
    - Type: Monday.com Node (boardItem getAll)  
    - Role: Retrieves all OKR items from specified Monday.com board and group.  
    - Configuration: Uses environment variables for boardId and groupId. Returns all items.  
    - Credential: Monday.com API with proper access.  
    - Connections: Output to "Map Monday Fields → Standard KR Schema."  
    - Edge Cases: API rate limits, invalid board/group IDs, missing credentials.

  - **Map Monday Fields → Standard KR Schema**  
    - Type: Set Node  
    - Role: Maps raw Monday.com item fields (using column indices) to readable, typed fields like "Objective Name," "Jira Epic Keys," "Target Progress," etc.  
    - Key Expressions: Accesses column_values by index with expressions like `$json.column_values[0].text`.  
    - Connections: Outputs standardized objects to "Split KR → Epics Mapper."  
    - Edge Cases: Column order changes in Monday.com boards could break field mapping; requires manual update.

  - **Split KR → Epics Mapper**  
    - Type: Code Node  
    - Role: Splits comma-separated Jira Epic keys into separate items for parallel processing. Each Key Result (KR) item is duplicated per epic.  
    - JS Logic: Splits `Jira Epic Keys` by comma, trims, filters empty strings, then creates one output item per epic with KR metadata.  
    - Connections: Outputs to both "Fetch Jira Epic Details" and "Join KR + Epic Data (SQL Merge)" (as inputs 0 and 1 respectively).  
    - Edge Cases: Empty or malformed epic key fields; missing epics lead to no downstream Jira fetch.

  - **Note: Monday Board Config** and **Note: Field Mapping Logic** and **Note: Epic Splitting Logic**  
    - Types: Sticky Notes  
    - Roles: Provide setup instructions for Monday.com board columns, explain field mapping importance and epic splitting rationale.  
    - Edge Cases: None (documentation only).

#### 1.3 Jira Epic Data Fetching & Normalization

- **Overview:** Retrieves detailed epic data from Jira for each epic key, then normalizes the complex API response into a flat schema suitable for merging and calculation.
- **Nodes Involved:**  
  - Fetch Jira Epic Details  
  - Note: Jira Epic Fetching  
  - Normalize Jira Response  
  - Note: Jira Response Cleanup  

- **Node Details:**

  - **Fetch Jira Epic Details**  
    - Type: Jira Node (issue get)  
    - Role: Fetches Jira issue data for each epic key provided by previous node.  
    - Configuration: Uses `$json.epic_key` expression to specify issueKey.  
    - Credential: Jira Software Cloud API with read access.  
    - Connections: Outputs to "Normalize Jira Response."  
    - Edge Cases: API rate limits (150 requests/min), missing or inaccessible epic keys returning errors, authentication failures.

  - **Normalize Jira Response**  
    - Type: Code Node  
    - Role: Flattens nested Jira issue JSON to extract fields: key, summary, status, assignee, due date, created, updated, description.  
    - JS Logic: Maps over issues array, picks and renames fields for easier processing.  
    - Connections: Output to second input of "Join KR + Epic Data (SQL Merge)."  
    - Edge Cases: Unexpected API schema changes, null fields (e.g., unassigned epics), empty responses.

  - **Note: Jira Epic Fetching** and **Note: Jira Response Cleanup**  
    - Types: Sticky Notes  
    - Roles: Describe setup and API rate limit considerations, normalization rationale.  
    - Edge Cases: None (documentation only).

#### 1.4 Data Joining & Variance Calculation

- **Overview:** Performs SQL-style join of KR metadata and Jira epic details, then computes progress variance and status per KR.
- **Nodes Involved:**  
  - Join KR + Epic Data (SQL Merge)  
  - Note: Data Merging  
  - Compute KR Progress & Variance  
  - Note: Variance Calculation  

- **Node Details:**

  - **Join KR + Epic Data (SQL Merge)**  
    - Type: Merge Node (combineBySql)  
    - Role: Performs LEFT JOIN on KR epic key to Jira issue key, merging Monday.com metadata with Jira details.  
    - SQL Query: Selects relevant fields from both inputs, joining on epic keys.  
    - Connections: Output to "Compute KR Progress & Variance."  
    - Edge Cases: Missing epic details cause null fields; join failures due to key mismatches.

  - **Compute KR Progress & Variance**  
    - Type: Code Node  
    - Role: Aggregates multiple epic records per KR; calculates weighted actual progress based on Jira epic statuses; computes variance from target progress and assigns status labels ("At Risk", "Ahead", "On Track").  
    - JS Logic:  
      - Groups input by KR ID.  
      - Maps epic statuses to weights: To Do=0%, In Progress=50%, Done=100%.  
      - Calculates mean actual progress, variance, absolute variance, and status.  
      - Outputs aggregated item per KR with all computed metrics.  
    - Connections: Outputs to "Update KR Status on Monday" and "Aggregate Final Results for Reporting."  
    - Edge Cases: No epics for a KR results in zero actual progress; unknown statuses default to 0%.

  - **Note: Data Merging** and **Note: Variance Calculation**  
    - Types: Sticky Notes  
    - Roles: Explain SQL merge strategy and progress calculation logic.  
    - Edge Cases: None (documentation only).

#### 1.5 Monday.com Update

- **Overview:** Writes back the computed actual progress, variance, and status fields into the Monday.com board for each KR.
- **Nodes Involved:**  
  - Update KR Status on Monday  
  - Note: Monday Board Update  

- **Node Details:**

  - **Update KR Status on Monday**  
    - Type: Monday.com Node (changeMultipleColumnValues)  
    - Role: Updates multiple columns for a given KR item ID with computed metrics.  
    - Configuration: Uses environment variables for board ID and column IDs. Column values are set with expressions referencing computed JSON fields.  
    - Credential: Monday.com API.  
    - Connections: Output flows to aggregation node.  
    - Edge Cases: Incorrect column IDs cause update failure; API rate limits; missing credentials.

  - **Note: Monday Board Update**  
    - Type: Sticky Note  
    - Role: Instructions for configuring correct Monday.com column IDs for actual progress, variance, and status.  
    - Edge Cases: None (documentation only).

#### 1.6 Aggregation & Reporting

- **Overview:** Aggregates all KR variance data into a single payload, then sends formatted reports via Slack and Outlook email.
- **Nodes Involved:**  
  - Aggregate Final Results for Reporting  
  - Note: Data Aggregation  
  - Post Slack Variance Report  
  - Note: Slack Configuration  
  - Email Variance Digest (Outlook)  
  - Note: Email Configuration  

- **Node Details:**

  - **Aggregate Final Results for Reporting**  
    - Type: Aggregate Node (aggregateAllItemData)  
    - Role: Combines multiple KR items into one aggregated data array for reporting nodes.  
    - Connections: Outputs to Slack and Email nodes.  
    - Edge Cases: Large datasets could impact performance.

  - **Post Slack Variance Report**  
    - Type: Slack Node (message send)  
    - Role: Sends a formatted message to a Slack channel, listing all KRs with their variance details.  
    - Configuration: Uses Slack channel ID from environment variable; message uses Markdown with dynamic content rendering including owner, target, actual, variance, epic count, last update, and status.  
    - Credential: Slack API credentials.  
    - Edge Cases: Invalid channel ID, API rate limits, auth failures.

  - **Email Variance Digest (Outlook)**  
    - Type: Microsoft Outlook Node (send email)  
    - Role: Sends a styled HTML email report to a configured recipient with OKR variance details.  
    - Configuration: Subject line includes timestamp; toRecipients is an environment variable; body uses HTML template with dynamic data.  
    - Credential: Outlook OAuth2 with send permissions.  
    - Edge Cases: Invalid recipient email, credential expiration, send permission errors.

  - **Note: Data Aggregation**, **Note: Slack Configuration**, **Note: Email Configuration**  
    - Types: Sticky Notes  
    - Roles: Provide setup guidance for aggregation, Slack channel configuration, and email setup.  
    - Edge Cases: None (documentation only).

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                              | Input Node(s)               | Output Node(s)                               | Sticky Note                                                                                                                       |
|-----------------------------------|----------------------------|----------------------------------------------|-----------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview                 | Sticky Note                | Describes workflow purpose and usage         | -                           | -                                            | ## 🎯 OKR Sync & Variance Tracking System ... See full overview content above                                                  |
| Note: Schedule Setup             | Sticky Note                | Scheduling instructions                       | -                           | -                                            | ## ⏰ Schedule Configuration ... See full content above                                                                         |
| Daily OKR Sync Trigger           | Schedule Trigger           | Triggers workflow on schedule                 | -                           | Fetch OKRs from Monday.com                    |                                                                                                                                  |
| Note: Monday Board Config        | Sticky Note                | Monday.com board setup notes                   | -                           | -                                            | ## 📋 Monday.com Board Setup ... See full content above                                                                         |
| Fetch OKRs from Monday.com       | Monday.com Node            | Fetches OKRs from Monday.com board            | Daily OKR Sync Trigger      | Map Monday Fields → Standard KR Schema        |                                                                                                                                  |
| Note: Field Mapping Logic         | Sticky Note                | Explains Monday.com field mapping             | -                           | -                                            | ## 🔄 Field Mapping ... See full content above                                                                                   |
| Map Monday Fields → Standard KR Schema | Set Node                 | Standardizes Monday.com item fields            | Fetch OKRs from Monday.com  | Split KR → Epics Mapper                       |                                                                                                                                  |
| Note: Epic Splitting             | Sticky Note                | Explains epic splitting logic                  | -                           | -                                            | ## 🔀 Epic Splitting Logic ... See full content above                                                                            |
| Split KR → Epics Mapper          | Code Node                  | Splits KR with multiple epics into multiple items | Map Monday Fields → Standard KR Schema | Join KR + Epic Data (SQL Merge), Fetch Jira Epic Details |                                                                                                                                  |
| Note: Jira Epic Fetching         | Sticky Note                | Jira integration and rate limit notes          | -                           | -                                            | ## 🔗 Jira Integration ... See full content above                                                                               |
| Fetch Jira Epic Details          | Jira Node                  | Fetches epic details from Jira                  | Split KR → Epics Mapper     | Normalize Jira Response                       |                                                                                                                                  |
| Note: Jira Response Cleanup      | Sticky Note                | Explains Jira response normalization           | -                           | -                                            | ## 🧹 Response Normalization ... See full content above                                                                         |
| Normalize Jira Response          | Code Node                  | Flattens and extracts Jira epic fields          | Fetch Jira Epic Details     | Join KR + Epic Data (SQL Merge)               |                                                                                                                                  |
| Note: Data Merging              | Sticky Note                | Explains SQL merging strategy                   | -                           | -                                            | ## 🔗 SQL Merge Strategy ... See full content above                                                                             |
| Join KR + Epic Data (SQL Merge) | Merge Node                 | Left joins KR and epic data for combined context | Split KR → Epics Mapper, Normalize Jira Response | Compute KR Progress & Variance                  |                                                                                                                                  |
| Note: Variance Calculation       | Sticky Note                | Explains progress calculation logic             | -                           | -                                            | ## 📊 Progress Calculation Logic ... See full content above                                                                     |
| Compute KR Progress & Variance   | Code Node                  | Calculates actual progress, variance, and status| Join KR + Epic Data (SQL Merge) | Update KR Status on Monday, Aggregate Final Results for Reporting |                                                                                                                                  |
| Note: Monday Board Update        | Sticky Note                | Monday.com column update instructions           | -                           | -                                            | ## ✍️ Monday.com Update ... See full content above                                                                             |
| Update KR Status on Monday       | Monday.com Node            | Updates actual progress, variance, and status   | Compute KR Progress & Variance | Aggregate Final Results for Reporting          |                                                                                                                                  |
| Note: Data Aggregation           | Sticky Note                | Explains aggregation for reporting               | -                           | -                                            | ## 📦 Aggregation for Reporting ... See full content above                                                                      |
| Aggregate Final Results for Reporting | Aggregate Node            | Aggregates all KR metrics into single payload   | Update KR Status on Monday  | Post Slack Variance Report, Email Variance Digest (Outlook) |                                                                                                                                  |
| Note: Slack Configuration        | Sticky Note                | Slack message setup instructions                 | -                           | -                                            | ## 💬 Slack Notification ... See full content above                                                                             |
| Post Slack Variance Report       | Slack Node                 | Sends variance report message to Slack channel  | Aggregate Final Results for Reporting | -                                            |                                                                                                                                  |
| Note: Email Configuration        | Sticky Note                | Email setup instructions                          | -                           | -                                            | ## 📧 Email Notification ... See full content above                                                                             |
| Email Variance Digest (Outlook)  | Microsoft Outlook Node     | Sends variance report email via Outlook          | Aggregate Final Results for Reporting | -                                            |                                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure interval (e.g., daily at 9 AM) and timezone as needed.

2. **Add a Monday.com node to fetch OKRs:**  
   - Type: Monday.com (boardItem getAll)  
   - Set `boardId` to environment variable `MONDAY_BOARD_ID`  
   - Set `groupId` to environment variable `MONDAY_GROUP_ID`  
   - Return all items.  
   - Connect output from Schedule Trigger.

3. **Add a Set node to map Monday.com fields:**  
   - Extract and assign readable fields from Monday.com columns using expressions like `$json.column_values[index].text` for each required field:  
     - id, name, created_at, Objective Name, Jira Epic Keys, Target Progress, Current Progress, Threshold, Owners Email, Status, Date.  
   - Connect output from Monday.com fetch node.

4. **Add a Code node to split KR items by Jira Epic keys:**  
   - Write JS code to split the comma-separated `Jira Epic Keys` into multiple output items, each with KR metadata and single epic key.  
   - Connect output from Set node.

5. **Add a Jira node to fetch epic details:**  
   - Type: Jira (issue get)  
   - Set `issueKey` to expression `$json.epic_key`  
   - Use Jira credentials with read access.  
   - Connect output from the split code node.

6. **Add a Code node to normalize Jira responses:**  
   - Flatten nested Jira JSON to extract key, summary, status, assignee, due dates, created and updated timestamps, description.  
   - Connect output from Jira fetch node.

7. **Add a Merge node configured as SQL combineBySql:**  
   - Inputs:  
     - Input 1: Output from the split KR → Epics mapper node (input 0)  
     - Input 2: Output from normalized Jira response node (input 1)  
   - SQL Query: LEFT JOIN on epic key to Jira issue key, selecting all required fields for variance calculation.  
   - Connect outputs accordingly.

8. **Add a Code node to compute KR progress and variance:**  
   - Group items by KR id.  
   - Map epic statuses to weights: "To Do"=0, "In Progress"=50, "Done"=100.  
   - Calculate average actual progress, variance, absolute variance.  
   - Determine status: "At Risk" if variance exceeds threshold negatively, "Ahead" if positive and above threshold, else "On Track."  
   - Output aggregated KR items.  
   - Connect output from Merge node.

9. **Add a Monday.com node to update KR status:**  
   - Type: Monday.com (changeMultipleColumnValues)  
   - Use environment variables for boardId and column IDs for Actual Progress, Variance, Status columns.  
   - Map update values from computed JSON fields.  
   - Connect output from variance calculation node.

10. **Add an Aggregate node to combine all KR metrics into one array:**  
    - Configure to aggregate all item data.  
    - Connect output from Monday.com update node.

11. **Add a Slack node to post variance report:**  
    - Use Slack API credentials.  
    - Target Slack channel via environment variable `SLACK_CHANNEL_ID`.  
    - Format message using Slack Markdown, including all KR variance details dynamically from aggregated data.  
    - Connect output from aggregate node.

12. **Add a Microsoft Outlook node to email variance digest:**  
    - Use Outlook OAuth2 credentials with send permissions.  
    - Set recipient to environment variable `REPORT_RECIPIENT_EMAIL`.  
    - Compose HTML email body dynamically with KR variance data, including styling and color coding variance values.  
    - Connect output from aggregate node.

13. **Add Sticky Notes at relevant points to document configuration and best practices:**  
    - Schedule configuration instructions.  
    - Monday.com board setup and field mapping notes.  
    - Jira integration setup and rate limit notes.  
    - Data merging and variance calculation logic explanations.  
    - Slack and Email configuration guides.

14. **Test the workflow end-to-end:**  
    - Validate environment variables are correctly set for board IDs, column IDs, Slack channel, email recipients.  
    - Confirm credentials for Monday.com, Jira, Slack, and Outlook are authorized and functional.  
    - Run the workflow manually and verify data flow, updates, and notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow automates OKR sync and variance reporting with integrations to Monday.com, Jira, Slack, and Microsoft Outlook.                            | Workflow Overview sticky note                                                                       |
| Schedule trigger should be configured to run during off-peak hours to avoid API rate limits and ensure timely updates.                            | Note: Schedule Setup sticky note                                                                   |
| Monday.com board requires specific columns including Objective Name, Jira Epic Keys, Target Progress, Current Progress, Threshold, Owner Email.   | Note: Monday Board Config sticky note                                                              |
| Jira API calls are rate-limited; the workflow respects limits by processing sequentially.                                                          | Note: Jira Epic Fetching sticky note                                                               |
| Slack message formatting leverages Slack’s Block Kit and Markdown for clear variance reporting.                                                     | Note: Slack Configuration sticky note                                                              |
| Outlook email uses dynamic HTML template with conditional styling for variance values (red for negative, green for positive).                      | Note: Email Configuration sticky note                                                              |

---

This document provides a detailed breakdown of the workflow structure, node functionality, configuration details, and stepwise reproduction instructions, enabling efficient understanding, maintenance, and customization by technical users or automation agents.