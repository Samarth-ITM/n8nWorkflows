Automated Product Health Monitor with Anomaly Detection & AI Root Cause Analysis

https://n8nworkflows.xyz/workflows/automated-product-health-monitor-with-anomaly-detection---ai-root-cause-analysis-11117


# Automated Product Health Monitor with Anomaly Detection & AI Root Cause Analysis

---

## 1. Workflow Overview

This workflow, titled **Automated Product Health Monitor with Anomaly Detection & AI Root Cause Analysis**, is designed for continuous monitoring of product health metrics, detecting anomalies in revenue and usage, logging incidents, and notifying the product team through multiple communication channels. It integrates statistical anomaly detection with AI-powered root cause analysis and provides daily summary reporting to leadership.

### Target Use Cases

- Automated detection of significant deviations in product revenue (e.g., churn MRR) and feature usage metrics.
- Structured incident logging for anomalies with contextual details.
- Alerting the product team via Slack and email upon anomaly detection.
- Optional synchronization of incidents to Notion for documentation and postmortem analysis.
- AI-powered root cause hypothesis generation with a summary sent to the product team.
- Daily reporting of overall product health incidents for leadership review.

### Logical Blocks

The workflow is organized into the following logical blocks, each representing a major functional segment:

- **1.1 Daily Revenue Health Monitoring**  
  Queries recent revenue metrics, detects anomalies in churn MRR, logs incidents, and triggers alerts.

- **1.2 Daily Usage Health Monitoring**  
  Queries daily product usage metrics, detects usage drops, logs incidents, and triggers alerts.

- **1.3 Root Cause Analysis & AI Summary**  
  For open incidents, aggregates contextual data by country and plan, generates AI-driven hypotheses, updates incidents, and notifies teams.

- **1.4 Daily Incident Reporting**  
  Generates a daily summary report of incidents detected the previous day, sends emails, and logs the report in Notion.

- **1.5 System Logging**  
  Records key workflow statuses and messages in a system logs table for auditing and monitoring.

---

## 2. Block-by-Block Analysis

### 2.1 Daily Revenue Health Monitoring

**Overview:**  
This block runs daily to query revenue metrics (specifically churn MRR), analyzes for statistical anomalies indicating abnormal increases, logs incidents if anomalies are detected, and sends alerts through Slack and email.

**Nodes Involved:**  
- Trigger RH (Schedule Trigger)  
- Execute the SQL query (Postgres)  
- anomalies check (Code)  
- log incident (Postgres)  
- Update notions (Notion)  
- slack notification (Slack)  
- email alert (Gmail)  
- log system (Postgres)

**Node Details:**

- **Trigger RH**  
  - Type: Schedule Trigger  
  - Role: Initiates the daily revenue health check at 8:00 AM.  
  - Configuration: Cron-like hourly trigger at 8:00.  
  - Edge cases: Missed triggers if n8n is down; time zone considerations.

- **Execute the SQL query**  
  - Type: Postgres node  
  - Role: Queries last 30 days of revenue metrics (`date`, `new_mrr`, `churn_mrr`, `net_mrr_delta`) from the `daily_revenue_metrics` table.  
  - Configuration: Simple SELECT with date filter and order.  
  - Credentials: Connected to Postgres (Supabase).  
  - Edge cases: DB connectivity issues, empty data sets.

- **anomalies check**  
  - Type: Code node (JavaScript)  
  - Role: Applies a statistical anomaly detection model using a 7-day baseline window and z-score calculation to detect significant increases in churn MRR (`z > 2`).  
  - Key logic:  
    - Sorts data by date.  
    - Calculates mean and standard deviation over baseline.  
    - Computes z-score and delta percentage.  
    - Flags anomaly if z-score > 2 with severity levels based on delta %.  
  - Output: Anomaly data or empty if none.  
  - Edge cases: Less than 8 data points, zero std deviation, no anomalies.

- **log incident**  
  - Type: Postgres node  
  - Role: Inserts detected anomalies as incidents into the `incidents` table with detailed context including metric, date, baseline, actual values, delta, severity, and raw JSON context.  
  - Configuration: SQL INSERT with parameter binding from JSON data.  
  - Edge cases: DB write failures, duplicate incidents.

- **Update notions**  
  - Type: Notion node  
  - Role: Updates a Notion database page with incident details for documentation (optional).  
  - Configuration: Uses page URL and Notion API credentials.  
  - Edge cases: API rate limits, invalid page ID.

- **slack notification**  
  - Type: Slack node  
  - Role: Sends a formatted Slack alert with anomaly details (metric, date, severity, baseline, current, delta).  
  - Configuration: OAuth2 authentication, target channel configured by user.  
  - Edge cases: Slack API errors, authentication failures.

- **email alert**  
  - Type: Gmail node  
  - Role: Sends an email alert with the anomaly details to configured recipients.  
  - Configuration: OAuth2 Gmail credentials, HTML formatted message, subject includes metric name.  
  - Edge cases: Email delivery failures, invalid addresses.

- **log system**  
  - Type: Postgres node  
  - Role: Logs status message indicating successful incident logging and notification sending.  
  - Configuration: Inserts a record to `system_logs` with fixed message.  
  - Edge cases: DB connectivity issues.

---

### 2.2 Daily Usage Health Monitoring

**Overview:**  
This block monitors product feature usage metrics daily, detecting abnormal drops in key usage indicators, logging incidents, and notifying the team.

**Nodes Involved:**  
- Trigger UH (Schedule Trigger)  
- daily usage metrics (Postgres)  
- anomalies (Code)  
- insert incidents (Postgres)  
- Slack notification (Slack)  
- usage health email (Gmail)  
- log system UH (Postgres)

**Node Details:**

- **Trigger UH**  
  - Type: Schedule Trigger  
  - Role: Runs the usage health check daily at 8:15 AM.  
  - Configuration: Scheduled trigger with minute offset.  
  - Edge cases: Same as Trigger RH.

- **daily usage metrics**  
  - Type: Postgres node  
  - Role: Queries last 30 days of usage metrics (`active_accounts`, `accounts_viewing_dashboard`, `accounts_using_alerts`) from `daily_usage_metrics` table.  
  - Configuration: SELECT query ordered by date.  
  - Edge cases: Empty data, DB issues.

- **anomalies**  
  - Type: Code node (JavaScript)  
  - Role: Detects significant drops in `accounts_using_alerts` using similar z-score methodology but specifically for decreases (`z < -2`).  
  - Additional logic: Severity levels based on delta percentage thresholds (-25%, -50%).  
  - Output: Detected anomalies or empty.  
  - Edge cases: Insufficient data, zero std deviation.

- **insert incidents**  
  - Type: Postgres node  
  - Role: Inserts usage anomalies into `incidents` table with metric name, date, baseline, actual, delta %, severity, and full JSON context.  
  - Edge cases: Same as log incident.

- **Slack notification**  
  - Type: Slack node  
  - Role: Posts Slack alert for usage anomaly with detailed data.  
  - Edge cases: Same as above.

- **usage health email**  
  - Type: Gmail node  
  - Role: Sends an email alert about the usage anomaly.  
  - Edge cases: Same as email alert.

- **log system UH**  
  - Type: Postgres node  
  - Role: Logs success message for usage health check execution.  
  - Edge cases: DB write failures.

---

### 2.3 Root Cause Analysis & AI Summary

**Overview:**  
This block selects the earliest open incident, enriches it with aggregated churn revenue data by country and plan, uses OpenAI to generate a root cause summary with hypotheses and recommendations, updates the incident record, and sends the summary to Slack and email.

**Nodes Involved:**  
- Trigger CS (Schedule Trigger)  
- select open incident (Postgres)  
- Condition incident (If)  
- revenue by country (Postgres)  
- revenue by plan (Postgres)  
- Merge data (Merge)  
- sum up/ hypothesis (OpenAI node)  
- root cause summary (Slack)  
- root cause summary email (Gmail)  
- update incident status (Postgres)  
- log system final (Postgres)

**Node Details:**

- **Trigger CS**  
  - Type: Schedule Trigger  
  - Role: Runs every 15 minutes to check for open incidents to analyze.  
  - Edge cases: Frequent execution may hit rate limits with AI or DB.

- **select open incident**  
  - Type: Postgres node  
  - Role: Fetches the oldest open incident from `incidents` table ordered by detection time.  
  - Edge cases: No open incidents returns empty.

- **Condition incident**  
  - Type: If node  
  - Role: Checks if a valid incident was retrieved (incident_id exists).  
  - Edge cases: No incident, no further execution.

- **revenue by country**  
  - Type: Postgres node  
  - Role: Aggregates churn MRR by country over the 3 days preceding the incident date.  
  - Configuration: Joins `revenue_events` with `accounts`, filters churn events, groups by country.  
  - Edge cases: No data for given interval.

- **revenue by plan**  
  - Type: Postgres node  
  - Role: Aggregates churn MRR by plan over same period.  
  - Edge cases: Similar to above.

- **Merge data**  
  - Type: Merge node  
  - Role: Combines country and plan churn data into a single JSON object for AI input.  
  - Edge cases: Mismatched data sets.

- **sum up/ hypothesis**  
  - Type: OpenAI (LangChain) node  
  - Role: Uses GPT-4O-mini to generate a natural language summary describing the incident, key impacted segments, hypotheses for root causes, and recommended next steps.  
  - Key expressions: Multi-line prompt with incident and churn data injected.  
  - Edge cases: API errors, rate limits, prompt failures.

- **root cause summary**  
  - Type: Slack node  
  - Role: Posts the AI-generated summary to a Slack channel for product team visibility.  
  - Edge cases: Same Slack API considerations.

- **root cause summary email**  
  - Type: Gmail node  
  - Role: Sends the AI summary via email to product stakeholders.  
  - Edge cases: Email delivery issues.

- **update incident status**  
  - Type: Postgres node  
  - Role: Updates the incident status to 'analyzed', stores the AI summary, and timestamps the update.  
  - Edge cases: DB write failures.

- **log system final**  
  - Type: Postgres node  
  - Role: Logs a system message confirming successful root cause analysis and summary dispatch.  
  - Edge cases: DB connectivity.

---

### 2.4 Daily Incident Reporting

**Overview:**  
Runs daily to collect all incidents detected the previous day, summarizes counts by metric and severity, sends a report email to leadership, and creates a corresponding Notion page for documentation.

**Nodes Involved:**  
- daily report trigger (Schedule Trigger)  
- Execute SQL query incident check (Postgres)  
- sum up (Code)  
- daily report email (Gmail)  
- Notions database creation (Notion)  
- log system final1 (Postgres)

**Node Details:**

- **daily report trigger**  
  - Type: Schedule Trigger  
  - Role: Fires at 9:00 AM daily for report generation.  
  - Edge cases: Missed triggers.

- **Execute SQL query incident check**  
  - Type: Postgres node  
  - Role: Retrieves all incidents detected yesterday, ordered by detection time.  
  - Edge cases: Empty result sets.

- **sum up**  
  - Type: Code node (JavaScript)  
  - Role: Aggregates incidents by metric and severity, composes a formatted textual report with counts and summary text.  
  - Edge cases: No incidents yields minimal report.

- **daily report email**  
  - Type: Gmail node  
  - Role: Sends the composed daily report email to configured recipients.  
  - Edge cases: Email failures.

- **Notions database creation**  
  - Type: Notion node  
  - Role: Creates a new Notion page titled with current date containing the daily report summary and incident count.  
  - Edge cases: API rate limits, invalid database ID.

- **log system final1**  
  - Type: Postgres node  
  - Role: Logs success message for daily report execution.  
  - Edge cases: DB write failures.

---

### 2.5 System Logging

**Overview:**  
Throughout the workflow, system logs are written to a dedicated `system_logs` table to track execution statuses, successes, and messages for auditability.

**Nodes Involved:**  
- log system  
- log system UH  
- log system final  
- log system final1

**Node Details:**

- All are Postgres nodes performing simple insertions of fixed text messages describing the workflow stage and outcome.

- These nodes ensure visibility into workflow health and can aid troubleshooting.

- Edge cases include DB connectivity failures or transaction errors.

---

## 3. Summary Table

| Node Name                  | Node Type                | Functional Role                                  | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                               |
|----------------------------|--------------------------|-------------------------------------------------|-------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------|
| Trigger RH                 | Schedule Trigger         | Starts daily revenue health check                | —                             | Execute the SQL query           |                                                                                                           |
| Execute the SQL query      | Postgres                 | Queries last 30 days revenue metrics             | Trigger RH                    | anomalies check                |                                                                                                           |
| anomalies check            | Code                     | Detects anomalies in churn MRR                    | Execute the SQL query         | log incident                  |                                                                                                           |
| log incident               | Postgres                 | Inserts detected incidents into DB                | anomalies check              | Update notions                |                                                                                                           |
| Update notions             | Notion                   | Updates Notion page with incident details         | log incident                 | slack notification            |                                                                                                           |
| slack notification         | Slack                    | Sends Slack alert for revenue anomaly             | Update notions               | email alert                  |                                                                                                           |
| email alert               | Gmail                    | Sends email alert for revenue anomaly             | slack notification           | log system                   |                                                                                                           |
| log system                | Postgres                 | Logs success message for revenue anomaly workflow | email alert                  | —                            |                                                                                                           |
| Trigger UH                | Schedule Trigger         | Starts daily usage health check                    | —                             | daily usage metrics           |                                                                                                           |
| daily usage metrics       | Postgres                 | Queries last 30 days usage metrics                 | Trigger UH                   | anomalies                    |                                                                                                           |
| anomalies                 | Code                     | Detects drops in feature usage                      | daily usage metrics          | insert incidents             |                                                                                                           |
| insert incidents          | Postgres                 | Inserts usage incidents                             | anomalies                    | Slack notification           |                                                                                                           |
| Slack notification        | Slack                    | Sends Slack alert for usage anomaly                 | insert incidents             | usage health email           |                                                                                                           |
| usage health email        | Gmail                    | Sends email alert for usage anomaly                 | Slack notification           | log system UH                |                                                                                                           |
| log system UH             | Postgres                 | Logs success message for usage health workflow      | usage health email           | —                            |                                                                                                           |
| Trigger CS                | Schedule Trigger         | Starts root cause analysis every 15 minutes         | —                             | select open incident         |                                                                                                           |
| select open incident      | Postgres                 | Retrieves oldest open incident                       | Trigger CS                   | Condition incident           |                                                                                                           |
| Condition incident        | If                       | Checks if an open incident exists                    | select open incident         | revenue by country, revenue by plan |                                                                                                           |
| revenue by country        | Postgres                 | Aggregates churn MRR by country                      | Condition incident           | Merge data                  |                                                                                                           |
| revenue by plan           | Postgres                 | Aggregates churn MRR by plan                         | Condition incident           | Merge data                  |                                                                                                           |
| Merge data                | Merge                    | Combines country and plan churn data                 | revenue by country, revenue by plan | sum up/ hypothesis           |                                                                                                           |
| sum up/ hypothesis        | OpenAI (LangChain)        | Generates AI summary and hypotheses                  | Merge data                  | root cause summary           |                                                                                                           |
| root cause summary        | Slack                    | Posts AI-generated summary to Slack                  | sum up/ hypothesis          | root cause summary email    |                                                                                                           |
| root cause summary email  | Gmail                    | Emails AI summary to product team                     | root cause summary          | update incident status       |                                                                                                           |
| update incident status    | Postgres                 | Updates incident status with AI summary              | root cause summary email    | log system final            |                                                                                                           |
| log system final          | Postgres                 | Logs success message for root cause analysis         | update incident status      | —                            |                                                                                                           |
| daily report trigger      | Schedule Trigger         | Starts daily incident report at 9:00 AM              | —                             | Execute SQL query incident check |                                                                                                           |
| Execute SQL query incident check | Postgres           | Retrieves yesterday's incidents                       | daily report trigger        | sum up                     |                                                                                                           |
| sum up                    | Code                     | Aggregates and formats daily incidents report          | Execute SQL query incident check | daily report email          |                                                                                                           |
| daily report email        | Gmail                    | Sends daily product health report email               | sum up                      | Notions database creation   |                                                                                                           |
| Notions database creation | Notion                   | Creates Notion page for the daily report               | daily report email          | log system final1           |                                                                                                           |
| log system final1         | Postgres                 | Logs success message for daily report workflow         | Notions database creation   | —                            |                                                                                                           |
| Sticky Note               | Sticky Note              | Workflow overview and setup instructions               | —                             | —                            | ## How it works... (Detailed workflow explanation and setup instructions)                                  |
| Sticky Note1              | Sticky Note              | Block 1) Daily revenue health summary                  | —                             | —                            | ## 1) Daily revenue health\nChecks for MRR anomalies. Logs incident + notifies Product Team.               |
| Sticky Note2              | Sticky Note              | Block 2) Daily usage health summary                     | —                             | —                            | ## 2) Daily usage health\nMonitors feature adoption. Detects unusual drops in usage patterns.              |
| Sticky Note3              | Sticky Note              | Block 3) Root cause & summary overview                   | —                             | —                            | ## 3) Root Cause & Summary\nEnriches incidents with country/plan breakdown + AI-generated hypothesis.      |
| Sticky Note5              | Sticky Note              | Block 4) Daily report summary                             | —                             | —                            | ## 4) daily report\nGenerates daily global health report for leadership.                                   |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Set Up Credentials

- Configure your Postgres (Supabase) credentials in n8n with access to your product metrics and incidents tables.
- Add Slack OAuth2 credentials with bot token permissions for posting messages.
- Add Gmail OAuth2 credentials for sending emails.
- (Optional) Add Notion API credentials and supply the database ID for incident logging.
- Add OpenAI API credentials for AI summaries.

### Step 2: Create Schedule Triggers

- **Trigger RH**: Create a Schedule Trigger node configured to run daily at 08:00.
- **Trigger UH**: Create a Schedule Trigger node configured to run daily at 08:15.
- **Trigger CS**: Create a Schedule Trigger node configured to run every 15 minutes.
- **daily report trigger**: Create a Schedule Trigger node configured to run daily at 09:00.

### Step 3: Daily Revenue Health Monitoring Setup

- Create a Postgres node "Execute the SQL query" connected from Trigger RH:  
  - Query last 30 days from `daily_revenue_metrics` with columns `date`, `new_mrr`, `churn_mrr`, `net_mrr_delta`.
- Create a Code node "anomalies check" connected from the above:  
  - Implement 7-day baseline z-score anomaly detection for *increased* churn MRR (z > 2).
- Create a Postgres node "log incident" connected from Code node:  
  - Insert detected anomalies into `incidents` with detailed parameters.
- Create a Notion node "Update notions" connected from "log incident":  
  - Update a Notion page representing the incident.
- Create a Slack node "slack notification" connected from "Update notions":  
  - Send a Slack alert with incident details.
- Create a Gmail node "email alert" connected from "slack notification":  
  - Send a formatted email alert.
- Create a Postgres node "log system" connected from "email alert":  
  - Log success message for workflow progress.

### Step 4: Daily Usage Health Monitoring Setup

- Create a Postgres node "daily usage metrics" connected from Trigger UH:  
  - Query last 30 days from `daily_usage_metrics` with relevant usage columns.
- Create a Code node "anomalies" connected from above:  
  - Detect *decreases* in usage metric `accounts_using_alerts` (z < -2) and set severity.
- Create a Postgres node "insert incidents" connected from Code node:  
  - Insert usage anomalies into `incidents`.
- Create a Slack node "Slack notification" connected from "insert incidents":  
  - Send Slack alert for usage anomaly.
- Create a Gmail node "usage health email" connected from Slack node:  
  - Send email alert for usage anomaly.
- Create a Postgres node "log system UH" connected from "usage health email":  
  - Log success message.

### Step 5: Root Cause Analysis & AI Summary Setup

- Connect Trigger CS to Postgres node "select open incident":  
  - Query oldest open incident.
- Connect "select open incident" to an If node "Condition incident":  
  - Check if incident_id is not empty.
- If true, connect to Postgres nodes:  
  - "revenue by country": Query churn MRR by country for 3 days prior to incident date.  
  - "revenue by plan": Query churn MRR by plan for same period.
- Connect both revenue queries to a Merge node "Merge data".
- Connect "Merge data" to OpenAI node "sum up/ hypothesis":  
  - Use GPT-4O-mini with prompt injecting incident data and churn by country/plan to produce a root cause summary and hypotheses.
- Connect OpenAI node to Slack node "root cause summary" to post summary.
- Connect Slack node to Gmail node "root cause summary email" to email summary.
- Connect email to Postgres node "update incident status" to mark incident analyzed and store AI summary.
- Connect that to Postgres node "log system final" to log completion.

### Step 6: Daily Incident Reporting Setup

- Connect "daily report trigger" to Postgres node "Execute SQL query incident check":  
  - Query incidents detected yesterday.
- Connect to Code node "sum up":  
  - Aggregate incidents count by metric and severity and compose report text.
- Connect "sum up" to Gmail node "daily report email" to send report.
- Connect email node to Notion node "Notions database creation" to create daily report page.
- Connect Notion node to Postgres node "log system final1" to log success.

### Step 7: Add Sticky Notes for Documentation

- Add multiple Sticky Note nodes at appropriate canvas positions describing workflow purpose, block summaries, and setup instructions as in the original.

### Step 8: Connect all nodes according to the connections described in section 3 Summary Table.

### Additional Notes:

- Use parameter binding in Postgres nodes to prevent injection and ensure data integrity.
- Ensure all API credentials are valid and permissions are sufficient.
- Test anomaly detection code nodes with sample data covering edge cases (e.g., no data, zero variance).
- Adjust schedule timings as per operational requirements.
- Optional: Customize Slack channels, email recipients, and Notion database IDs before deployment.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow is based on a Supabase (Postgres) backend with core tables: accounts, revenue_events, product_usage_events, incidents, system_logs. Ensure these tables exist with the correct schema.                                                                                                                                                                    | Setup instructions in Sticky Note content                                                                           |
| The anomaly detection uses a simple z-score method with a 7-day baseline window and delta % thresholds to classify severity. This is a lightweight statistical model suitable for many use cases but can be enhanced with more advanced techniques if needed.                                                                                                        | See Code nodes "anomalies check" and "anomalies"                                                                     |
| The AI root cause summary uses OpenAI GPT-4O-mini model via n8n LangChain integration. Ensure your API quota and rate limits are adequate.                                                                                                                                                                                                                          | OpenAI API key required                                                                                              |
| Slack notifications use OAuth2 authentication and require the bot to have permissions to post messages in the target channel.                                                                                                                                                                                                                                       | Slack app setup required                                                                                             |
| Gmail nodes use OAuth2 and require appropriate scopes for sending emails.                                                                                                                                                                                                                                                                                            | Gmail OAuth2 credentials                                                                                            |
| Notion nodes require integration token and database ID configured for incident logging and report creation.                                                                                                                                                                                                                                                        | Notion API setup                                                                                                    |
| The workflow is designed to minimize alert noise by only triggering on significant statistical anomalies. No anomaly results in silent stop.                                                                                                                                                                                                                       | Explained in Sticky Note overview                                                                                   |
| The system logs table captures workflow execution info for audit and troubleshooting. Review these logs regularly.                                                                                                                                                                                                                                                  | `system_logs` table                                                                                                 |
| For extending the system, additional workflows can read from the incidents table to add features such as daily recaps, detailed root cause analyses, or integration with ticketing systems.                                                                                                                                                                         | Workflow modularity                                                                                                 |

---

# Disclaimer

The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---