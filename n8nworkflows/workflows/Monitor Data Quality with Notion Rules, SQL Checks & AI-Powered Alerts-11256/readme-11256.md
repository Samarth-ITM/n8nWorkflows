Monitor Data Quality with Notion Rules, SQL Checks & AI-Powered Alerts

https://n8nworkflows.xyz/workflows/monitor-data-quality-with-notion-rules--sql-checks---ai-powered-alerts-11256


# Monitor Data Quality with Notion Rules, SQL Checks & AI-Powered Alerts

---

### 1. Workflow Overview

**Purpose:**  
This workflow automates data quality monitoring by dynamically applying user-defined validation rules stored in Notion to SQL datasets. It detects anomalies, generates quality scores and AI-powered diagnostics, and escalates incidents through alerts and issue tracking only when real data quality problems occur, minimizing noise and manual overhead.

**Target Use Cases:**  
- Continuous data quality monitoring for business-critical databases  
- Automated anomaly detection based on configurable rules without manual SQL editing  
- AI-powered root cause analysis and remediation recommendations  
- Incident management integration with Notion, Slack, email, and Jira  
- Quality trend tracking and reporting  

**Logical Blocks:**  
- **1.1 Input Reception and Rule Loading:** Scheduled trigger starts the process; rules are fetched from Notion database.  
- **1.2 Rule Processing & SQL Validation:** Rules are split in batches, dynamically routed by dataset, and executed as SQL queries against the database.  
- **1.3 Anomaly Aggregation & Scoring:** Query results are aggregated into summaries with quality scores and sample anomalies.  
- **1.4 AI-Powered Analysis:** Summary is passed to an OpenAI model for diagnostic interpretation and fix recommendations.  
- **1.5 Metadata Enrichment & Quality Trend:** Run metadata including timestamp, environment, and scores are set and a quality trend is calculated (stubbed as zero here).  
- **1.6 Incident Decision & Reporting:** Conditions determine if anomalies require escalation. If yes, a Notion incident page is created, alerts sent via Slack and email, and a Jira ticket is optionally created.  
- **1.7 Optional Autofix Simulation:** Marks anomalies with auto-fix recommendations for tracking.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Rule Loading

- **Overview:**  
Starts the workflow daily at 7 AM, retrieves data quality rules from a Notion database for processing.

- **Nodes Involved:**  
  - Daily trigger  
  - Get database data  

- **Node Details:**  

1. **Daily trigger**  
   - Type: Schedule Trigger  
   - Configured to trigger once daily at 07:00 local time.  
   - No inputs; outputs trigger event.  
   - Potential failures: scheduler misconfiguration, timezone issues.  
   - No sub-workflows.  

2. **Get database data**  
   - Type: Notion Database Query  
   - Retrieves all entries from a configured Notion database representing data quality rules.  
   - Credentials required: Notion API with access to the rules database.  
   - Output: JSON array of rules with fields such as dataset name ("Source") and validation SQL conditions ("Rule").  
   - Potential failures: API rate limits, credential expiration, database permissions.  

---

#### 1.2 Rule Processing & SQL Validation

- **Overview:**  
Splits the retrieved rules into batches, routes them by dataset type using a switch, then executes dynamic SQL queries using the rule conditions on the respective tables.

- **Nodes Involved:**  
  - split batch  
  - Switch module  
  - Product anomalies (Postgres node)  
  - check orders (Postgres node)  

- **Node Details:**  

1. **split batch**  
   - Type: SplitInBatches  
   - Splits input (rules) into manageable batches for sequential processing.  
   - No specific batch size configured (defaults apply).  
   - Outputs batches to the switch node.  
   - Edge cases: large rule sets may require batch size tuning to avoid timeouts.

2. **Switch module**  
   - Type: Switch  
   - Routes each rule batch based on the "Source" field to determine which SQL node to execute.  
   - Conditions:  
     - If Source = "products" → Product anomalies node  
     - If Source = "orders" → check orders node  
   - Prevents running irrelevant queries on wrong datasets.  
   - Edge cases: missing or unexpected "Source" values cause dropped batches (no default branch).  

3. **Product anomalies**  
   - Type: Postgres  
   - Executes a dynamic SQL query constructed from rule fields:  
     ```sql
     SELECT * FROM {{ $json["Source"] }} WHERE {{ $json["Rule"] }};
     ```  
   - Uses Postgres credentials.  
   - Returns rows violating the rule (anomalies).  
   - Potential failures: SQL syntax errors from malformed rules, DB connection issues, query timeouts.

4. **check orders**  
   - Type: Postgres  
   - Runs a hardcoded query on the `orders` table checking for NULL or zero values in key fields.  
   - Credentials same as above.  
   - Static validation complementary to dynamic rules.  
   - Potential failures: DB connectivity, query performance.

---

#### 1.3 Anomaly Aggregation & Scoring

- **Overview:**  
Aggregates query results from all rule checks, calculates overall data quality score, and generates a textual summary with samples.

- **Nodes Involved:**  
  - build summary  

- **Node Details:**  

1. **build summary**  
   - Type: Code (JavaScript)  
   - Aggregates all anomaly rows from previous nodes.  
   - Calculates:  
     - totalIssues = number of anomalies found  
     - totalRows = anomaly count (used for score denominator)  
     - qualityScore = 100 if no issues, otherwise decreases proportionally  
   - Constructs a multi-line text summary with top 5 anomalies listed by id, sku, price fields (if present).  
   - Outputs JSON object containing summary strings, anomaly arrays, and scores.  
   - Edge cases: empty input arrays handled gracefully, missing anomaly fields defaulted to "N/A".  

---

#### 1.4 AI-Powered Analysis

- **Overview:**  
Sends the summary to an OpenAI GPT-4o-mini model to generate a structured JSON diagnostic including urgency, root causes, impact, and recommended fixes.

- **Nodes Involved:**  
  - AI analysis and recommendation  

- **Node Details:**  

1. **AI analysis and recommendation**  
   - Type: OpenAI via LangChain node  
   - Model: GPT-4o-mini (OpenAI GPT-4 variant)  
   - Prompt includes data quality summary and requests JSON-only output with fields: Summary, Urgency, Root_Causes, Recommended_Fix, Impact.  
   - Inputs: `summary` field from previous node.  
   - Outputs parsed JSON diagnostic.  
   - Potential failures: API rate limits, invalid or incomplete prompt data, response parsing errors.  
   - Requires valid OpenAI API credentials.  

---

#### 1.5 Metadata Enrichment & Quality Trend

- **Overview:**  
Sets metadata fields for the run (timestamp, environment, anomaly counts, quality score) and calculates a quality trend (currently returning 0).

- **Nodes Involved:**  
  - run metadata  
  - calculate trend  

- **Node Details:**  

1. **run metadata**  
   - Type: Set node  
   - Sets variables:  
     - run_date = current datetime  
     - run_id = composite of timestamp plus random number  
     - env = "prod" as static environment tag  
     - total_anomalies and quality_score from summary  
   - Used for reporting and incident tracking.  

2. **calculate trend**  
   - Type: Code (JavaScript)  
   - Currently a stub setting `trend` field to 0, placeholder for future trend calculation logic.  
   - Inputs JSON with qualityScore, outputs augmented JSON.  

---

#### 1.6 Incident Decision & Reporting

- **Overview:**  
Determines if anomalies exceed thresholds, then creates a Notion incident run page, sends Slack and email alerts, and creates Jira tickets if required.

- **Nodes Involved:**  
  - Issue condition  
  - Data condition  
  - Data quality run page (Notion)  
  - Slack message alert  
  - Email reporting  
  - Jira issue  

- **Node Details:**  

1. **Issue condition**  
   - Type: If node  
   - Checks if anomalies_found > 0  
   - If no anomalies, workflow ends quietly.  

2. **Data condition**  
   - Type: If node  
   - Checks if quality_score < 80 OR total_anomalies > 50  
   - If true, proceeds to create incident page and alerts; otherwise, only logs the run.  

3. **Data quality run page**  
   - Type: Notion page creation  
   - Creates a detailed page in a Notion database for incident tracking with metadata:  
     - Run Date, Env, Total anomalies, Quality score  
     - Includes AI analysis and diagnostic text  
   - Credentials: Notion API access.  
   - Outputs used for Jira issue creation and alerts.  
   - Potential failures: API limits, permission issues.  

4. **Slack message alert**  
   - Type: Slack node with OAuth2  
   - Sends formatted alert message with run info and summary to a configured Slack channel.  
   - Credentials: Slack OAuth2 with posting rights.  
   - Potential failure: webhook or OAuth token expiration.  

5. **Email reporting**  
   - Type: Gmail OAuth2 node  
   - Sends email with data quality summary and link to Notion incident page.  
   - Credentials: Gmail OAuth2  
   - Potential failure: SMTP errors, auth expiration.  

6. **Jira issue**  
   - Type: Jira Software Cloud node  
   - Creates a Jira bug issue with high priority for the data quality incident.  
   - Uses run date and summary for issue title and description.  
   - Credentials: Jira Software Cloud API.  
   - Potential failures: API limits, incorrect project or issue type IDs, permission issues.  

---

#### 1.7 Optional Autofix Simulation

- **Overview:**  
Annotates anomalies with an "auto_fix" recommendation tag to simulate or track auto-fix suggestions.

- **Nodes Involved:**  
  - Autofix simulation  

- **Node Details:**  

1. **Autofix simulation**  
   - Type: Code (JavaScript)  
   - Iterates over anomalies array, adds a field `auto_fix` with value "recommended" for each anomaly.  
   - Outputs modified JSON for downstream nodes (Jira creation).  
   - No external dependencies or credentials.  

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                               | Input Node(s)          | Output Node(s)                       | Sticky Note                                                                                      |
|-------------------------|-------------------------------|----------------------------------------------|-----------------------|------------------------------------|-------------------------------------------------------------------------------------------------|
| Daily trigger           | Schedule Trigger               | Starts workflow daily at 07:00                |                       | Get database data                  | ##  How it works\n\nThis workflow monitors data quality by running rule-based checks...         |
| Get database data       | Notion Database Query          | Retrieves data quality rules from Notion      | Daily trigger          | split batch                       |                                                                                                 |
| split batch             | SplitInBatches                 | Splits rules into batches for processing      | Get database data      | Switch module                    |                                                                                                 |
| Switch module           | Switch                        | Routes rules by dataset source                 | split batch            | Product anomalies, check orders   | ## 1) Data Quality Rules\n\nRuns configurable validation rules from Notion and executes SQL checks automatically. |
| Product anomalies       | Postgres                      | Executes dynamic SQL for product anomalies    | Switch module          | build summary                   |                                                                                                 |
| check orders            | Postgres                      | Executes static SQL check on orders table     | Switch module          | build summary                   |                                                                                                 |
| build summary           | Code                          | Aggregates anomalies and calculates score     | Product anomalies, check orders | AI analysis and recommendation, run metadata | ## AI Summary & Scoring\n\nAggregates anomalies, generates a quality score, and provides root cause insights + recommended fixes. |
| AI analysis and recommendation | OpenAI (LangChain)       | Generates AI diagnostic JSON from summary     | build summary          | calculate trend                 |                                                                                                 |
| run metadata            | Set                           | Sets metadata fields for run                    | AI analysis and recommendation | calculate trend                 |                                                                                                 |
| calculate trend         | Code                          | Calculates quality trend (stub)                 | run metadata, AI analysis and recommendation | Issue condition               |                                                                                                 |
| Issue condition         | If                            | Checks if anomalies found > 0                    | calculate trend        | Data condition                  |                                                                                                 |
| Data condition          | If                            | Checks if quality score < 80 or anomalies > 50 | Issue condition        | Autofix simulation, Data quality run page |                                                                                                 |
| Autofix simulation      | Code                          | Marks anomalies with auto-fix recommendations   | Data condition (true)  | Jira issue                     |                                                                                                 |
| Data quality run page   | Notion Page                   | Creates Notion incident page                      | Data condition (false) | Email reporting, Slack message alert, Jira issue | ## 3) Incident Reporting & Alerts\n\nCreates Notion incident page, triggers Slack/Email/Jira alerts based on severity thresholds. |
| Slack message alert     | Slack (OAuth2)                | Sends Slack alert message                         | Data quality run page  |                                |                                                                                                 |
| Email reporting         | Gmail (OAuth2)                | Sends email report with data quality summary     | Data quality run page  |                                |                                                                                                 |
| Jira issue              | Jira Software Cloud           | Creates Jira ticket for data quality incident    | Autofix simulation     | Data quality run page            |                                                                                                 |
| Sticky Note             | Sticky Note                   | Documentation and explanations                   |                       |                                | ##  How it works\n\nThis workflow monitors data quality by running rule-based checks...         |
| Sticky Note1            | Sticky Note                   | Documentation for rules block                     |                       |                                | ## 1) Data Quality Rules\n\nRuns configurable validation rules from Notion and executes SQL checks automatically. |
| Sticky Note2            | Sticky Note                   | Documentation for AI scoring block                 |                       |                                | ## AI Summary & Scoring\n\nAggregates anomalies, generates a quality score, and provides root cause insights + recommended fixes. |
| Sticky Note3            | Sticky Note                   | Documentation for incident reporting block          |                       |                                | ## 3) Incident Reporting & Alerts\n\nCreates Notion incident page, triggers Slack/Email/Jira alerts based on severity thresholds. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**  
   - Postgres database credentials with read access.  
   - Notion API credentials with access to two databases: Data Quality Rules and Data Quality Incident Runs pages.  
   - OpenAI API credentials with access to GPT-4o-mini model.  
   - Slack OAuth2 credentials with permissions to post messages to a channel.  
   - Gmail OAuth2 credentials for sending email.  
   - Jira Software Cloud API credentials with rights to create issues.

2. **Add a Schedule Trigger Node:**  
   - Set to run daily at 07:00 local time.  
   - No inputs.

3. **Add Notion Database Node:**  
   - Configure to query the "Data quality rules" database by ID.  
   - Set output to fetch all rule entries (Source dataset and Rule condition fields).

4. **Add SplitInBatches Node:**  
   - Connect input from Notion database node.  
   - No batch size explicitly set (default).  
   - Purpose: process rules batch-wise.

5. **Add Switch Node:**  
   - Connect input from SplitInBatches node.  
   - Create two rules:  
     - If `Source` equals `"products"` route to Product anomalies node.  
     - If `Source` equals `"orders"` route to check orders node.  

6. **Add Postgres Node "Product anomalies":**  
   - Connect input from Switch node branch for "products".  
   - Set operation to Execute Query.  
   - Query:  
     ```sql
     SELECT * FROM {{ $json["Source"] }} WHERE {{ $json["Rule"] }};
     ```  
   - Use Postgres credentials.

7. **Add Postgres Node "check orders":**  
   - Connect input from Switch node branch for "orders".  
   - Set operation to Execute Query.  
   - Use static SQL query checking for NULL or zero in key fields on orders table.  
   - Use Postgres credentials.

8. **Add Code Node "build summary":**  
   - Connect inputs from both SQL query nodes.  
   - Aggregate anomaly rows.  
   - Calculate total anomalies, quality score (100 if zero issues, else decreasing).  
   - Build text summary with top 5 anomalies.  
   - Output JSON with anomalies and metrics.

9. **Add OpenAI Node "AI analysis and recommendation":**  
   - Connect from build summary.  
   - Select GPT-4o-mini model.  
   - Prompt includes summary text; request structured JSON with Summary, Urgency, Root Causes, Recommended Fix, Impact.  
   - Use OpenAI API credentials.

10. **Add Set Node "run metadata":**  
    - Connect from AI node.  
    - Set variables: run_date (now), run_id (timestamp + random), env ("prod"), total_anomalies, quality_score from summary.

11. **Add Code Node "calculate trend":**  
    - Connect from run metadata and AI node.  
    - Set `trend` field to 0 (placeholder).

12. **Add If Node "Issue condition":**  
    - Connect from calculate trend.  
    - Condition: anomalies_found > 0.  
    - True branch proceeds; false ends workflow quietly.

13. **Add If Node "Data condition":**  
    - Connect from Issue condition true branch.  
    - Condition: quality_score < 80 OR total_anomalies > 50.  
    - True branch for incident escalation, false branch logs run quietly.

14. **Add Code Node "Autofix simulation":**  
    - Connect from Data condition true branch.  
    - Iterate anomalies, add "auto_fix" field = "recommended".

15. **Add Notion Node "Data quality run page":**  
    - Connect from Data condition false branch and from Jira node (see below).  
    - Create page with run metadata, summary, AI diagnostic.  
    - Use Notion API credentials.

16. **Add Jira Node "Jira issue":**  
    - Connect from Autofix simulation node.  
    - Create bug issue with high priority, summary referencing run date, description from summary.  
    - Use Jira Software Cloud credentials.

17. **Connect Jira issue node output to Data quality run page node input.**

18. **Add Slack Node "Slack message alert":**  
    - Connect from Data quality run page node.  
    - Post formatted alert message to designated channel.  
    - Use Slack OAuth2 credentials.

19. **Add Gmail Node "Email reporting":**  
    - Connect from Data quality run page node.  
    - Send email with summary and link to Notion incident page.  
    - Use Gmail OAuth2 credentials.

20. **Add Sticky Notes:**  
    - Add descriptive sticky notes corresponding to workflow blocks for user clarity.

21. **Test Workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Verify query execution, AI analysis, incident creation, and alert delivery.  
    - Adjust batch sizes, threshold parameters, or rules as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow monitors data quality by applying dynamic validation rules defined in Notion, using SQL queries to detect anomalies, and escalating only when issues are detected to minimize noise and manual review.                  | Workflow Purpose overview in Sticky Note       |
| All runs and incidents are recorded in Notion for audit and trend analysis, enabling continuous quality improvements.                                                                                                            | Incident Reporting & Alerts block explanation  |
| The AI node uses a specialized prompt to generate structured JSON diagnostics from raw data quality summaries, improving root cause identification and remediation guidance.                                                     | AI Summary & Scoring sticky note                |
| Setup requires creating two Notion databases (rules and runs), configuring SQL credentials, and setting up OpenAI, Slack, Gmail, and Jira credentials for full functionality.                                                     | Setup steps in main Sticky Note                 |
| Jira uses project and issue type IDs specific to the organization and must be configured accordingly. Slack channel IDs and Gmail addresses should be updated to target appropriate recipients.                                 | Credentials and configuration notes             |
| The workflow currently includes a stub for quality trend calculation for future enhancement.                                                                                                                                   | Metadata enrichment block                        |
| Links to Notion pages and Slack channels are embedded as parameters and should be updated to reflect actual workspace URLs and IDs.                                                                                             | Notion and Slack node parameters                 |

---

Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---