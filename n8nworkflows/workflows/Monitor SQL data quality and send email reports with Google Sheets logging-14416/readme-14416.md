Monitor SQL data quality and send email reports with Google Sheets logging

https://n8nworkflows.xyz/workflows/monitor-sql-data-quality-and-send-email-reports-with-google-sheets-logging-14416


# Monitor SQL data quality and send email reports with Google Sheets logging

# 1. Workflow Overview

This workflow performs recurring SQL-based data quality monitoring on a database table, evaluates the results against configurable thresholds, sends an HTML email report, and logs summary metrics to Google Sheets.

Its main use case is lightweight operational data observability for a table such as `restaurant_orders`, without requiring a full data quality platform. It is designed for teams that want a daily health check covering nulls, duplicates, row-count drift, and numeric outliers.

## 1.1 Trigger & Configuration

The workflow starts on a schedule and loads all operational parameters from a central configuration node. These parameters drive the SQL queries, threshold evaluation, recipient selection, and output formatting.

## 1.2 Parallel SQL Data Quality Checks

Four database queries run in parallel against PostgreSQL:
- null / missing value check
- duplicate check
- row count check
- outlier check

Each query returns a compact metrics row tagged with a `check_type`.

## 1.3 Result Consolidation & Quality Evaluation

The four SQL outputs are merged, then a Code node interprets the metrics, applies PASS/WARN/FAIL logic, computes an overall status, and builds a styled HTML report.

## 1.4 Reporting & Logging

The final formatted result is sent by Gmail and appended to a Google Sheet for audit/history tracking.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Configuration

### Overview
This block controls execution frequency and centralizes all user-editable settings. It is the only place where the table name, columns, thresholds, and recipient list are defined, making the rest of the workflow parameter-driven.

### Nodes Involved
- Schedule Trigger
- Config

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a recurring interval.
- **Configuration choices:**  
  Configured to run every 24 hours using an hourly interval rule.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to `Config`.
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Workflow must be activated for the trigger to run.
  - Schedule timing depends on the n8n instance timezone/settings.
  - The sticky note says “daily at 8am by default,” but the actual JSON shows an every-24-hours interval, not a specific 8am cron schedule.
- **Sub-workflow reference:**  
  None.

#### Config
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines reusable configuration values as fields in the item JSON.
- **Configuration choices:**  
  Creates the following fields:
  - `tableName` = `restaurant_orders`
  - `nullCheckCol` = `restaurant_orders`
  - `duplicateCheckCol` = `order_id`
  - `outlierCol` = `order_id`
  - `outlierMin` = `0`
  - `outlierMax` = `150`
  - `rowCountMin` = `500`
  - `rowCountMax` = `100000`
  - `nullThresholdPct` = `5`
  - `dupThresholdPct` = `1`
  - `Distribution list` = `user@example.com`
- **Key expressions or variables used:**  
  Downstream nodes reference this node via expressions such as:
  - `$('Config').item.json.tableName`
  - `$('Config').item.json.duplicateCheckCol`
  - `$('Config').item.json['Distribution list']`
- **Input and output connections:**  
  Input from `Schedule Trigger`. Outputs in parallel to:
  - `Null Check`
  - `Duplicate Check`
  - `Row Count`
  - `Outlier Check`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - The `nullCheckCol` value is currently set to `restaurant_orders`, which appears to be a table name, not a column name. This will likely break the null-check SQL unless the table really has a column with that name.
  - The Code node later expects `cfg.reportEmail`, but this Set node does not define `reportEmail`; it defines `Distribution list`. Email sending still works because Gmail uses `Distribution list` directly, but the `reportEmail` field returned by the Code node will be undefined.
  - Any typo in table or column names will cause SQL execution failures.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Parallel SQL Data Quality Checks

### Overview
This block executes four PostgreSQL queries in parallel. Each query is dynamically built from the configuration values and returns one summary row describing a specific quality dimension.

### Nodes Involved
- Null Check
- Duplicate Check
- Row Count
- Outlier Check

### Node Details

#### Null Check
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a SQL query against PostgreSQL to compute missing/null statistics.
- **Configuration choices:**  
  Runs a custom query that returns:
  - `check_type = 'null_check'`
  - `total_rows`
  - `null_count`
  - `null_pct`
  
  Logic:
  - Counts rows where the configured column is `NULL`
  - Also treats empty string values as missing by casting the column to text and comparing to `''`
- **Key expressions or variables used:**  
  - `$('Config').item.json.nullCheckCol`
  - `$('Config').item.json.tableName`
- **Input and output connections:**  
  Input from `Config`. Output to `Merge` input 0.
- **Version-specific requirements:**  
  Type version `2.5`. Requires valid PostgreSQL credentials.
- **Edge cases or potential failure types:**  
  - Invalid table/column name causes SQL syntax or relation errors.
  - If the selected column type cannot be cast safely in the database context, the query may fail.
  - Empty table is handled with `NULLIF(COUNT(*), 0)` to avoid division-by-zero, but the resulting percentage may be `NULL`.
  - SQL injection risk exists if config values are made user-controlled; these expressions are inserted directly into the query text.
- **Sub-workflow reference:**  
  None.

#### Duplicate Check
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a SQL query to estimate duplicate rate based on a chosen key column.
- **Configuration choices:**  
  Returns:
  - `check_type = 'dup_check'`
  - `total_rows`
  - `dup_count`
  - `dup_pct`
  
  Logic:
  - `COUNT(*) - COUNT(DISTINCT key_column)`
- **Key expressions or variables used:**  
  - `$('Config').item.json.duplicateCheckCol`
  - `$('Config').item.json.tableName`
- **Input and output connections:**  
  Input from `Config`. Output to `Merge` input 1.
- **Version-specific requirements:**  
  Type version `2.5`.
- **Edge cases or potential failure types:**  
  - If the duplicate-check column contains `NULL`, PostgreSQL `COUNT(DISTINCT ...)` semantics may affect interpretation.
  - This check identifies duplicate values of one column, not duplicate full rows.
  - Very large tables may make `COUNT(DISTINCT ...)` expensive.
- **Sub-workflow reference:**  
  None.

#### Row Count
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a SQL query to return total row count for the target table.
- **Configuration choices:**  
  Returns:
  - `check_type = 'row_count'`
  - `row_count`
- **Key expressions or variables used:**  
  - `$('Config').item.json.tableName`
- **Input and output connections:**  
  Input from `Config`. Output to `Merge` input 3.
- **Version-specific requirements:**  
  Type version `2.5`.
- **Edge cases or potential failure types:**  
  - Missing table or permissions issue will fail the query.
  - For very large tables, `COUNT(*)` may be slow depending on storage and indexes.
- **Sub-workflow reference:**  
  None.

#### Outlier Check
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a SQL query to count values outside a configured min/max range.
- **Configuration choices:**  
  Returns:
  - `check_type = 'outlier_check'`
  - `total_rows`
  - `outlier_count`
  - `outlier_pct`
  
  Logic:
  - counts rows where `outlierCol < outlierMin OR outlierCol > outlierMax`
- **Key expressions or variables used:**  
  - `$('Config').item.json.outlierCol`
  - `$('Config').item.json.outlierMin`
  - `$('Config').item.json.outlierMax`
  - `$('Config').item.json.tableName`
- **Input and output connections:**  
  Input from `Config`. Output to `Merge` input 4.
- **Version-specific requirements:**  
  Type version `2.5`.
- **Edge cases or potential failure types:**  
  - The outlier column should be numeric or comparable to numeric bounds.
  - If the column is text or mixed-type, the query may fail.
  - Empty table again leads to `NULL` percentage unless handled downstream.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Result Consolidation & Quality Evaluation

### Overview
This block combines the outputs from the SQL checks and transforms them into business-readable results. It calculates status labels, determines the overall health score, and renders the HTML email body.

### Nodes Involved
- Merge
- Evaluate & Format

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines multiple upstream result streams into one downstream input set.
- **Configuration choices:**  
  Configured with `numberInputs: 5`.
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  Inputs from:
  - `Null Check` → input 0
  - `Duplicate Check` → input 1
  - `Row Count` → input 3
  - `Outlier Check` → input 4
  
  Output to `Evaluate & Format`.
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - There is no node connected to input 2, even though the merge is configured for 5 inputs. Depending on node behavior/version, this may still work, but it is structurally inconsistent and should ideally be reduced to 4 inputs or fully wired.
  - If any SQL node fails, the merge will not receive complete data.
- **Sub-workflow reference:**  
  None.

#### Evaluate & Format
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript node that interprets quality metrics, applies thresholds, generates status badges, builds HTML, and prepares output fields for email and Sheets.
- **Configuration choices:**  
  The code:
  1. Reads config from `$('Config').item.json`
  2. Collects all merged items via `$input.all()`
  3. Finds each metric set by `check_type`
  4. Parses values into numbers
  5. Applies threshold logic
  6. Builds a `checks` array for report rows
  7. Determines `overall` status
  8. Generates a styled HTML report
  9. Returns one consolidated JSON object
- **Key expressions or variables used:**  
  Internally references:
  - `cfg.tableName`
  - `cfg.nullCheckCol`
  - `cfg.duplicateCheckCol`
  - `cfg.rowCountMin`
  - `cfg.rowCountMax`
  - `cfg.outlierCol`
  - `cfg.outlierMin`
  - `cfg.outlierMax`
  - `cfg.nullThresholdPct`
  - `cfg.dupThresholdPct`
  - `cfg.reportEmail`
  
  Returned fields include:
  - `overall`
  - `htmlReport`
  - `emailSubject`
  - `reportEmail`
  - `sheet_timestamp`
  - `sheet_table`
  - `sheet_null_pct`
  - `sheet_dup_pct`
  - `sheet_row_count`
  - `sheet_outlier_pct`
  - `sheet_status`
- **Input and output connections:**  
  Input from `Merge`. Outputs to:
  - `Log to Google Sheets`
  - `Send a message`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If one or more checks are missing, defaults are used (`0`), which may hide failures rather than surfacing them.
  - `evalStatus` for outliers uses a hard-coded threshold of `5` instead of a config field; this is functional but inconsistent with the other checks.
  - `cfg.reportEmail` is undefined because the Config node does not create that field.
  - `toLocaleString()` output for row count may vary slightly by locale/runtime.
  - HTML emails may render differently across mail clients.
  - Timestamp is generated in UTC string format by code, not by workflow timezone settings.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Reporting & Logging

### Overview
This block delivers the final result to two destinations: email for immediate visibility and Google Sheets for history and auditability.

### Nodes Involved
- Send a message
- Log to Google Sheets

### Node Details

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated HTML report through Gmail.
- **Configuration choices:**  
  - Recipient: `={{ $('Config').item.json['Distribution list'] }}`
  - Subject: `={{ $json.emailSubject }}`
  - Message body: `={{ $json.htmlReport }}`
- **Key expressions or variables used:**  
  - `$('Config').item.json['Distribution list']`
  - `$json.emailSubject`
  - `$json.htmlReport`
- **Input and output connections:**  
  Input from `Evaluate & Format`. No downstream output.
- **Version-specific requirements:**  
  Type version `2.2`. Requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Gmail OAuth token expiry or permission issues.
  - Invalid recipient list format.
  - Large HTML body or account sending limits.
  - Depending on Gmail node options, HTML rendering behavior should be verified in the email client.
- **Sub-workflow reference:**  
  None.

#### Log to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one row of summary metrics to a Google Sheet.
- **Configuration choices:**  
  Operation: `append`  
  Spreadsheet:
  - Document ID: `1zIzM4gtIvtEj6t9YmBCQtieYLZLRmXj_37BoBtVTWw8`
  - Sheet: `Sheet1` / `gid=0`
  
  Mapped columns:
  - `Timestamp` ← `sheet_timestamp`
  - `Table` ← `sheet_table`
  - `Null %` ← `sheet_null_pct`
  - `Dup %` ← `sheet_dup_pct`
  - `Row Count` ← `sheet_row_count`
  - `Outlier %` ← `sheet_outlier_pct`
  - `Status` ← `sheet_status`
  - `overall` ← `overall`
- **Key expressions or variables used:**  
  Standard field mappings such as:
  - `={{ $json.sheet_timestamp }}`
  - `={{ $json.sheet_status }}`
- **Input and output connections:**  
  Input from `Evaluate & Format`. No downstream output.
- **Version-specific requirements:**  
  Type version `4.4`. Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Spreadsheet or tab access denied.
  - Header mismatch between configured schema and actual sheet headers.
  - Data type conversion is not forced; if column expectations differ, append behavior may vary.
  - If the sheet is moved, deleted, or renamed, the node may fail.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Launches the workflow every 24 hours |  | Config | ⏰ Trigger & Configuration<br>Controls workflow execution and defines all dynamic inputs like table name, columns, thresholds, and email recipients. |
| Config | Set | Stores table, column, threshold, and recipient configuration | Schedule Trigger | Null Check; Duplicate Check; Row Count; Outlier Check | ⏰ Trigger & Configuration<br>Controls workflow execution and defines all dynamic inputs like table name, columns, thresholds, and email recipients.<br>⚠️ Critical Setup<br>Ensure all column names in the Config node match your database schema exactly. Incorrect names will break SQL queries. |
| Null Check | Postgres | Calculates null/missing value rate | Config | Merge | 🧪 Data Quality Checks (SQL)<br>Runs four parallel checks:<br>Null values<br>Duplicate records<br>Row count validation<br>Outlier detection<br>⚠️ Critical Setup<br>Ensure all column names in the Config node match your database schema exactly. Incorrect names will break SQL queries. |
| Duplicate Check | Postgres | Calculates duplicate percentage based on a key column | Config | Merge | 🧪 Data Quality Checks (SQL)<br>Runs four parallel checks:<br>Null values<br>Duplicate records<br>Row count validation<br>Outlier detection<br>⚠️ Critical Setup<br>Ensure all column names in the Config node match your database schema exactly. Incorrect names will break SQL queries. |
| Row Count | Postgres | Counts table rows | Config | Merge | 🧪 Data Quality Checks (SQL)<br>Runs four parallel checks:<br>Null values<br>Duplicate records<br>Row count validation<br>Outlier detection<br>⚠️ Critical Setup<br>Ensure all column names in the Config node match your database schema exactly. Incorrect names will break SQL queries. |
| Outlier Check | Postgres | Calculates percentage of values outside configured bounds | Config | Merge | 🧪 Data Quality Checks (SQL)<br>Runs four parallel checks:<br>Null values<br>Duplicate records<br>Row count validation<br>Outlier detection<br>⚠️ Critical Setup<br>Ensure all column names in the Config node match your database schema exactly. Incorrect names will break SQL queries. |
| Merge | Merge | Combines SQL result items before evaluation | Null Check; Duplicate Check; Row Count; Outlier Check | Evaluate & Format | 🔗 Merge Results<br>Combines outputs from all SQL checks into a single dataset for evaluation. |
| Evaluate & Format | Code | Applies thresholds, computes overall status, and builds HTML report | Merge | Log to Google Sheets; Send a message | 🧠 Evaluate & Format Report<br>Applies threshold logic (PASS/WARN/FAIL) and generates a clean HTML report with status indicators. |
| Log to Google Sheets | Google Sheets | Appends summary metrics to Google Sheets | Evaluate & Format |  | 📤 Output & Notifications<br>Sends report via email and logs results into Google Sheets for tracking and auditing. |
| Send a message | Gmail | Sends the HTML report by email | Evaluate & Format |  | 📤 Output & Notifications<br>Sends report via email and logs results into Google Sheets for tracking and auditing. |
| Sticky Note | Sticky Note | Workspace documentation / overview note |  |  | 📊 Automated Data Quality Report Bot<br><br>This workflow automatically monitors data quality for a selected database table and generates a structured report with actionable insights. It performs four key checks—null values, duplicates, row count anomalies, and outliers—then evaluates results against configurable thresholds to determine overall data health (PASS / WARN / FAIL). The output is sent via email and logged into Google Sheets for historical tracking and auditing.<br><br>How it works<br><br>The workflow runs on a schedule trigger (daily by default). It reads configuration values (table name, columns, thresholds) from the Config node and executes four SQL checks in parallel. Results are merged and processed in a code node, which evaluates each metric and generates a formatted HTML report. Finally, results are sent via email and appended to a Google Sheet.<br><br>Setup<br>## 🛠️ Setup Guide<br><br>**Step 1 — Config node**<br>Edit the `Config` node to set your table name, column names, thresholds and Email Distribution list<br><br>**Step 2 — DB credentials**<br>Open each of the 4 teal query nodes and attach your Postgres credential. Swap the node type if using MySQL or another DB.<br><br>**Step 3 — Email**<br>Connect your SMTP or Gmail credential to the `Send Email` node.<br><br>**Step 4 — Google Sheets**<br>Connect your Google account to `Log to Google Sheets`. Create a sheet with columns: Timestamp, Table, Null_Pct, Dup_Pct, Row_Count, Outlier_Pct, Status.<br><br>**Step 5 — Activate**<br>Toggle the workflow on. It runs daily at 8am by default.<br>Customization<br>Add more checks by duplicating query nodes.<br>Adjust thresholds for stricter/looser validation.<br>Extend reporting (Slack, Teams, dashboards).<br><br>**Customization**<br>Add more checks by duplicating query nodes.<br>Adjust thresholds for stricter/looser validation.<br>Extend reporting (Slack, Teams, dashboards). |
| Sticky Note1 | Sticky Note | Workspace documentation for trigger/config section |  |  | ⏰ Trigger & Configuration<br><br>Controls workflow execution and defines all dynamic inputs like table name, columns, thresholds, and email recipients. |
| Sticky Note2 | Sticky Note | Workspace documentation for SQL checks section |  |  | 🧪 Data Quality Checks (SQL)<br><br>Runs four parallel checks:<br><br>Null values<br>Duplicate records<br>Row count validation<br>Outlier detection |
| Sticky Note3 | Sticky Note | Workspace documentation for merge section |  |  | 🔗 Merge Results<br><br>Combines outputs from all SQL checks into a single dataset for evaluation. |
| Sticky Note4 | Sticky Note | Workspace documentation for evaluation section |  |  | 🧠 Evaluate & Format Report<br><br>Applies threshold logic (PASS/WARN/FAIL) and generates a clean HTML report with status indicators. |
| Sticky Note5 | Sticky Note | Workspace documentation for outputs section |  |  | 📤 Output & Notifications<br><br>Sends report via email and logs results into Google Sheets for tracking and auditing. |
| Sticky Note6 | Sticky Note | Workspace warning about schema consistency |  |  | ⚠️ Critical Setup<br><br>Ensure all column names in the Config node match your database schema exactly. Incorrect names will break SQL queries. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automated Data Quality Monitoring & Reporting (SQL + Email + Google Sheets)`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run every 24 hours.
   - If you prefer a fixed daily hour, replace the interval rule with a cron-style schedule.

3. **Add a Set node named `Config`**
   - Node type: `Set`
   - Add the following fields:
     - `tableName` → string → your target table, e.g. `restaurant_orders`
     - `nullCheckCol` → string → column to inspect for null/blank values
     - `duplicateCheckCol` → string → column to test for duplicate keys, e.g. `order_id`
     - `outlierCol` → string → numeric column for range checks
     - `outlierMin` → number → e.g. `0`
     - `outlierMax` → number → e.g. `150`
     - `rowCountMin` → number → e.g. `500`
     - `rowCountMax` → number → e.g. `100000`
     - `nullThresholdPct` → number → e.g. `5`
     - `dupThresholdPct` → number → e.g. `1`
     - `Distribution list` → string → recipient email(s)
   - Recommended improvement:
     - Also add `reportEmail` if you want the Code node output to remain internally consistent.

4. **Connect `Schedule Trigger` to `Config`**

5. **Add a Postgres node named `Null Check`**
   - Node type: `Postgres`
   - Operation: `Execute Query`
   - Credential: connect your PostgreSQL credential
   - Query:
     - Build a query that counts:
       - total rows
       - null/blank rows for the configured column
       - null percentage
     - Use expressions for table/column names from `Config`
   - Equivalent logic:
     - select `check_type = 'null_check'`
     - count all rows
     - count rows where target column is null or empty string
     - compute percentage safely with `NULLIF(COUNT(*), 0)`

6. **Add a Postgres node named `Duplicate Check`**
   - Operation: `Execute Query`
   - Credential: same PostgreSQL account
   - Query:
     - select `check_type = 'dup_check'`
     - compute `COUNT(*) - COUNT(DISTINCT configured_column)`
     - compute duplicate percentage

7. **Add a Postgres node named `Row Count`**
   - Operation: `Execute Query`
   - Query:
     - select `check_type = 'row_count'`
     - count all rows from configured table

8. **Add a Postgres node named `Outlier Check`**
   - Operation: `Execute Query`
   - Query:
     - select `check_type = 'outlier_check'`
     - count rows where configured numeric column is below `outlierMin` or above `outlierMax`
     - compute outlier percentage

9. **Connect `Config` to all four SQL nodes**
   - `Config` → `Null Check`
   - `Config` → `Duplicate Check`
   - `Config` → `Row Count`
   - `Config` → `Outlier Check`

10. **Add a Merge node named `Merge`**
    - Node type: `Merge`
    - Set number of inputs to match your SQL branches.
    - Best practice: use `4` inputs here, since there are four checks.
    - Connect:
      - `Null Check` → input 0
      - `Duplicate Check` → input 1
      - `Row Count` → input 2
      - `Outlier Check` → input 3

11. **Add a Code node named `Evaluate & Format`**
    - Node type: `Code`
    - Language: JavaScript
    - Paste logic that:
      - reads config from the `Config` node
      - reads all incoming items from Merge
      - finds each result by `check_type`
      - parses numeric values
      - evaluates status:
        - nulls: PASS/WARN/FAIL based on `nullThresholdPct`
        - duplicates: PASS/WARN/FAIL based on `dupThresholdPct`
        - row count: PASS/WARN/FAIL based on configured min/max range
        - outliers: PASS/WARN/FAIL based on a chosen threshold
      - computes overall status
      - generates an HTML report
      - returns a single JSON payload with email fields and sheet fields

12. **Implement status logic carefully**
    - In the provided workflow:
      - for “above” mode:
        - `0` → PASS
        - `<= threshold` → WARN
        - `> threshold` → FAIL
      - for “range” mode:
        - within min/max → PASS
        - outside 80%–120% tolerance band → FAIL
        - otherwise → WARN
    - Recommended improvement:
      - Add a configurable outlier threshold field to `Config` instead of hard-coding `5`.

13. **Generate output fields in the Code node**
    - Return at least:
      - `overall`
      - `htmlReport`
      - `emailSubject`
      - `sheet_timestamp`
      - `sheet_table`
      - `sheet_null_pct`
      - `sheet_dup_pct`
      - `sheet_row_count`
      - `sheet_outlier_pct`
      - `sheet_status`

14. **Connect `Merge` to `Evaluate & Format`**

15. **Add a Gmail node named `Send a message`**
    - Node type: `Gmail`
    - Credential: connect Gmail OAuth2
    - `To`: expression from Config, e.g. `$('Config').item.json['Distribution list']`
    - `Subject`: `{{$json.emailSubject}}`
    - `Message`: `{{$json.htmlReport}}`
    - Verify the node sends HTML as intended in your n8n version/account setup.

16. **Add a Google Sheets node named `Log to Google Sheets`**
    - Node type: `Google Sheets`
    - Credential: Google Sheets OAuth2
    - Operation: `Append`
    - Select spreadsheet and target sheet/tab
    - Create or use headers such as:
      - `Timestamp`
      - `Table`
      - `Null %`
      - `Dup %`
      - `Row Count`
      - `Outlier %`
      - `Status`
      - `overall`

17. **Map Google Sheets columns**
    - `Timestamp` ← `{{$json.sheet_timestamp}}`
    - `Table` ← `{{$json.sheet_table}}`
    - `Null %` ← `{{$json.sheet_null_pct}}`
    - `Dup %` ← `{{$json.sheet_dup_pct}}`
    - `Row Count` ← `{{$json.sheet_row_count}}`
    - `Outlier %` ← `{{$json.sheet_outlier_pct}}`
    - `Status` ← `{{$json.sheet_status}}`
    - `overall` ← `{{$json.overall}}`

18. **Connect `Evaluate & Format` to both output nodes**
    - `Evaluate & Format` → `Send a message`
    - `Evaluate & Format` → `Log to Google Sheets`

19. **Configure credentials**
    - **Postgres**
      - Host, port, database, user, password, SSL if required
      - Ensure the DB user can `SELECT` from the target table
    - **Gmail OAuth2**
      - Authorize the Google account
      - Confirm the account is allowed to send to the configured recipients
    - **Google Sheets OAuth2**
      - Authorize spreadsheet access
      - Ensure the spreadsheet is shared with the connected Google account if needed

20. **Create the Google Sheet structure**
    - In the referenced design, the sheet includes headers that match the node mapping.
    - Prefer exact header names to avoid mapping confusion.
    - Note that the large overview sticky note mentions underscored headers like `Null_Pct`, while the actual node uses headers with spaces like `Null %`. Use the actual node mapping you configure.

21. **Test each SQL node independently**
    - Run `Null Check`, `Duplicate Check`, `Row Count`, and `Outlier Check` one by one.
    - Validate:
      - table exists
      - columns exist
      - types are compatible
      - percentages return expected values

22. **Test the merged flow**
    - Run the workflow manually.
    - Verify that:
      - Merge receives all check rows
      - Code node outputs one item
      - Gmail receives a rendered HTML body
      - Google Sheets appends a row

23. **Correct two important issues if rebuilding faithfully**
    - Set `nullCheckCol` to a real column name, not the table name.
    - Either:
      - add `reportEmail` to Config, or
      - remove that unused field from the Code node output.

24. **Activate the workflow**
    - Once validated, switch the workflow to active so the Schedule Trigger starts running automatically.

25. **Optional enhancements**
    - Add an Error Trigger workflow for failed DB/email/sheets actions.
    - Add Slack/Teams notifications for FAIL only.
    - Add more SQL checks such as freshness, referential integrity, schema drift, or invalid enumerations.
    - Replace dynamic SQL text interpolation with safer validated input logic if config values can be edited by non-technical users.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 📊 Automated Data Quality Report Bot — monitors a database table for nulls, duplicates, row count anomalies, and outliers; evaluates PASS/WARN/FAIL; sends email and logs to Google Sheets. | Workspace overview note |
| The workflow description in notes says it runs “daily by default” and “daily at 8am by default,” but the actual trigger is configured as every 24 hours. | Important implementation discrepancy |
| Setup note: edit the `Config` node to set table name, column names, thresholds, and email distribution list. | Workspace setup guidance |
| Setup note: attach Postgres credentials to all four SQL nodes; replace node type if using MySQL or another database. | Workspace setup guidance |
| Setup note: connect Gmail/SMTP credentials to the email node. | Workspace setup guidance |
| Setup note: connect Google Sheets credentials and create a logging sheet. | Workspace setup guidance |
| The note suggests sheet columns `Timestamp, Table, Null_Pct, Dup_Pct, Row_Count, Outlier_Pct, Status`, but the actual configured sheet mapping uses headers like `Null %`, `Dup %`, `Row Count`, `Outlier %`, plus `overall`. | Important schema discrepancy |
| Customization note: duplicate SQL nodes to add more checks, adjust thresholds, or extend outputs to Slack, Teams, or dashboards. | Workspace customization guidance |

**Key implementation discrepancies to be aware of**
- `nullCheckCol` is misconfigured as `restaurant_orders`, likely causing SQL failure.
- `cfg.reportEmail` is referenced in code but not defined in the Config node.
- `Merge` is configured for 5 inputs while only 4 SQL branches are connected.
- Outlier evaluation uses a hard-coded threshold of `5` in code rather than a config value.