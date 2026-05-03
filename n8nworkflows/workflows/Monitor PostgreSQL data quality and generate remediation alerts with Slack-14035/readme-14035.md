Monitor PostgreSQL data quality and generate remediation alerts with Slack

https://n8nworkflows.xyz/workflows/monitor-postgresql-data-quality-and-generate-remediation-alerts-with-slack-14035


# Monitor PostgreSQL data quality and generate remediation alerts with Slack

# 1. Workflow Overview

This workflow is designed to monitor PostgreSQL data quality on a recurring schedule, detect several categories of anomalies, score their importance, and notify a team through Slack when issues are significant enough to justify action. It also records issues in PostgreSQL and attempts to refresh baseline statistics for future comparisons.

Its intended use cases include:

- Ongoing production database health monitoring
- Early detection of schema changes that may break downstream systems
- Detection of data degradation such as excessive null values
- Detection of unusual numeric distribution shifts
- Automated remediation support through suggested SQL statements
- Operational alerting and historical issue logging

## 1.1 Trigger & Configuration

The workflow starts every 6 hours and initializes a small set of runtime parameters such as the confidence threshold, null-rate threshold, outlier threshold, and the names of the audit/baseline tables.

## 1.2 Database Metadata Collection

The workflow retrieves three categories of PostgreSQL data:

- current schema metadata from `information_schema.columns`
- current table statistics from `pg_stat_user_tables`
- latest historical baselines from a configurable baseline table

These form the inputs for downstream anomaly checks.

## 1.3 Data Quality Detection Engine

Three parallel detection branches are intended to run:

- schema drift detection
- null explosion detection
- outlier distribution detection

Each branch emits issue records describing the detected problem.

## 1.4 Issue Aggregation, Scoring & Filtering

Detected issues are aggregated, assigned confidence scores, and then filtered so only issues that meet or exceed the configured confidence threshold proceed.

## 1.5 Remediation, Logging, Alerting & Baseline Maintenance

For qualifying issues, the workflow generates SQL fix suggestions, stores the issue in an audit table, sends a Slack alert, and finally updates baseline records.

## 1.6 Important Design Caveat

Although the overall architecture is clear, several nodes are not fully aligned with one another. In particular:

- some code nodes expect fields that upstream PostgreSQL nodes do not return
- some nodes reference property names that differ from actual emitted fields
- one branch expects merged inputs that are not actually connected
- the confidence-scoring stage does not match the aggregate node’s output shape
- the Slack message uses field names that do not exist in prior nodes

So the workflow is structurally useful, but as provided it will likely require corrections before it can run successfully end to end.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Configuration

### Overview

This block starts the workflow on a recurring schedule and injects reusable configuration values. These values are later referenced by expressions in SQL queries, conditional checks, and database writes.

### Nodes Involved

- Schedule DB Quality Scan
- Workflow Configuration

### Node Details

#### Schedule DB Quality Scan

- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger that runs the workflow automatically.
- **Configuration choices:**
  - Runs every 6 hours.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input; trigger node.
  - Outputs to `Workflow Configuration`.
- **Version-specific requirements:**
  - Uses node type version `1.3`.
- **Edge cases or potential failure types:**
  - Workflow will not run if inactive.
  - Timezone interpretation depends on n8n instance settings.
- **Sub-workflow reference:** None.

#### Workflow Configuration

- **Type and role:** `n8n-nodes-base.set`  
  Creates configuration fields used across the workflow.
- **Configuration choices:**
  - Adds:
    - `confidenceThreshold = 0.85`
    - `maxNullPercentage = 0.15`
    - `outlierStdDevThreshold = 3`
    - `auditTableName = data_quality_audit`
    - `baselineTableName = data_quality_baselines`
  - Keeps other incoming fields with `includeOtherFields: true`.
- **Key expressions or variables used:**
  - Referenced later via `$('Workflow Configuration').first().json...`
- **Input and output connections:**
  - Input from `Schedule DB Quality Scan`
  - Outputs in parallel to:
    - `Get Schema Metadata`
    - `Get Table Statistics`
    - `Get Historical Baselines`
- **Version-specific requirements:**
  - Uses Set node version `3.4`.
- **Edge cases or potential failure types:**
  - No `tablesToMonitor` is defined here, but `Detect Null Explosions` expects it.
  - `maxNullPercentage` is set to `0.15`, while the null-detection code treats it like a percentage value; this creates a semantic mismatch.
- **Sub-workflow reference:** None.

---

## 2.2 Database Metadata Collection

### Overview

This block queries PostgreSQL for current schema details, current table stats, and the latest historical baseline row. These outputs are intended to feed anomaly detection logic.

### Nodes Involved

- Get Schema Metadata
- Get Table Statistics
- Get Historical Baselines

### Node Details

#### Get Schema Metadata

- **Type and role:** `n8n-nodes-base.postgres`  
  Executes a SQL query against PostgreSQL to read schema metadata.
- **Configuration choices:**
  - Operation: Execute Query
  - Query reads:
    - `table_name`
    - `column_name`
    - `data_type`
    - `is_nullable`
  - Source: `information_schema.columns`
  - Filters by a placeholder schema name.
- **Key expressions or variables used:**
  - Query contains placeholder:
    - `<__PLACEHOLDER_VALUE__target schema name__>`
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Output to `Detect Schema Drift`
- **Version-specific requirements:**
  - Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - Will fail until the schema placeholder is replaced with a valid SQL string.
  - Database credential/auth issues.
  - Permissions may prevent access to `information_schema.columns`.
  - The output is row-based, but downstream schema-drift code expects a richer table/columns structure.
- **Sub-workflow reference:** None.

#### Get Table Statistics

- **Type and role:** `n8n-nodes-base.postgres`  
  Executes a SQL query to retrieve table-level statistics from PostgreSQL system views.
- **Configuration choices:**
  - Operation: Execute Query
  - Query reads:
    - `schemaname`
    - `tablename`
    - `n_live_tup as row_count`
    - `n_dead_tup as dead_rows`
  - Source: `pg_stat_user_tables`
  - Filters by a placeholder schema name.
- **Key expressions or variables used:**
  - Query contains placeholder:
    - `<__PLACEHOLDER_VALUE__target schema name__>`
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Outputs to:
    - `Detect Null Explosions`
    - `Detect Outlier Distributions`
- **Version-specific requirements:**
  - Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - Will fail until the schema placeholder is replaced.
  - Auth/permission issues.
  - Returned fields use `tablename`, not `table_name`; downstream code expects `table_name`.
  - Returned rows do not include per-column null counts, means, stddevs, or column types required by downstream code.
- **Sub-workflow reference:** None.

#### Get Historical Baselines

- **Type and role:** `n8n-nodes-base.postgres`  
  Reads the latest baseline record from a configurable baseline table.
- **Configuration choices:**
  - Operation: Execute Query
  - SQL is dynamically built from the configuration:
    - `SELECT * FROM <baselineTableName> ORDER BY recorded_at DESC LIMIT 1`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').first().json.baselineTableName`
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - No downstream connection in the provided graph.
- **Version-specific requirements:**
  - Postgres node version `2.6`.
- **Edge cases or potential failure types:**
  - The query sorts by `recorded_at`, but the later baseline-update node writes `baseline_timestamp`; column naming may not match.
  - If the baseline table does not exist, the query will fail.
  - Since this node is not connected to the detector code nodes, its data is not actually available to them in the current workflow graph.
- **Sub-workflow reference:** None.

---

## 2.3 Data Quality Detection Engine

### Overview

This block is intended to identify data quality issues using three independent detectors. Each detector emits issue objects with severity and metadata, but the current implementation has several input-shape mismatches.

### Nodes Involved

- Detect Schema Drift
- Detect Null Explosions
- Detect Outlier Distributions

### Node Details

#### Detect Schema Drift

- **Type and role:** `n8n-nodes-base.code` using Python  
  Compares current schema against historical baselines to detect new/removed tables, new/removed columns, type changes, and nullability changes.
- **Configuration choices:**
  - Python native runtime.
  - Uses UTC timestamp via `datetime.utcnow().isoformat()`.
  - Produces issue records with:
    - `type`
    - `severity`
    - `table_name`
    - `column_name`
    - `change_description`
    - `detected_at`
- **Key expressions or variables used:**
  - `_input.all()[0]` assumed to be current schema
  - `_input.all()[1]` assumed to be historical baselines
- **Input and output connections:**
  - Connected input only from `Get Schema Metadata`
  - Output to `Combine All Issues`
- **Version-specific requirements:**
  - Code node version `2`.
  - Requires Python native execution support in the n8n environment.
- **Edge cases or potential failure types:**
  - The node expects two inputs, but only one connection is present.
  - The schema query returns one row per column, but the code expects table-level objects with nested `columns`.
  - Baseline data is not connected into this node.
  - If `_input.all()[1]` does not exist, the code can error or behave unexpectedly.
- **Sub-workflow reference:** None.

#### Detect Null Explosions

- **Type and role:** `n8n-nodes-base.code` using Python  
  Intended to detect columns whose null percentage exceeds a configured threshold.
- **Configuration choices:**
  - Reads configuration from `_input.first().json`
  - Uses:
    - `tablesToMonitor`
    - `maxNullPercentage`
- **Key expressions or variables used:**
  - `config.get('tablesToMonitor', [])`
  - `config.get('maxNullPercentage', 5.0)`
- **Input and output connections:**
  - Connected input only from `Get Table Statistics`
  - Output to `Combine All Issues`
- **Version-specific requirements:**
  - Code node version `2`
  - Python native support required.
- **Edge cases or potential failure types:**
  - `Workflow Configuration` is not connected to this node, so config is not actually supplied.
  - `tablesToMonitor` is never defined in the configuration node.
  - `Get Table Statistics` returns table-level statistics, not `columns`, `null_count`, or `total_count`.
  - `stat.json.get('table_name')` will fail logically because upstream uses `tablename`.
  - Threshold semantics are inconsistent: config uses `0.15`, but the code compares against percentages like `5.0`.
- **Sub-workflow reference:** None.

#### Detect Outlier Distributions

- **Type and role:** `n8n-nodes-base.code` using Python  
  Intended to compare current numeric distribution statistics against historical baselines using z-scores.
- **Configuration choices:**
  - Reads outlier threshold from config
  - Scans input items for current stats and baseline rows
  - Produces issues with:
    - `type`
    - `severity`
    - `table_name`
    - `column_name`
    - `outlier_count`
    - `z_score`
    - `current_mean`
    - `baseline_mean`
    - `current_stddev`
    - `detected_at`
- **Key expressions or variables used:**
  - `config.get('outlierStdDevThreshold', 3)`
  - baseline key format: `table.column`
- **Input and output connections:**
  - Connected input only from `Get Table Statistics`
  - Output to `Combine All Issues`
- **Version-specific requirements:**
  - Code node version `2`
  - Python native support required.
- **Edge cases or potential failure types:**
  - Expects config and baseline items, but neither is actually merged into this node.
  - Upstream table-stat query does not provide:
    - `column_name`
    - `column_type`
    - `mean`
    - `stddev`
  - Field mismatch again: upstream has `tablename`, not `table_name`.
  - If no baseline exists or baseline stddev is zero, outlier detection is skipped.
- **Sub-workflow reference:** None.

---

## 2.4 Issue Aggregation, Scoring & Filtering

### Overview

This block combines issues from all detection branches, assigns confidence scores, and then checks whether any issue meets the configured threshold. It is conceptually central, but the aggregate output format and downstream logic are not aligned.

### Nodes Involved

- Combine All Issues
- Calculate Confidence Scores
- Check Confidence Threshold

### Node Details

#### Combine All Issues

- **Type and role:** `n8n-nodes-base.aggregate`  
  Aggregates all incoming items into a single object field named `issues`.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData`
  - Destination field: `issues`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Inputs from:
    - `Detect Schema Drift`
    - `Detect Null Explosions`
    - `Detect Outlier Distributions`
  - Output to `Calculate Confidence Scores`
- **Version-specific requirements:**
  - Aggregate node version `1`
- **Edge cases or potential failure types:**
  - If no issues are emitted by any branch, output shape may still exist but with an empty array.
  - This node produces a single item with `issues: [...]`, but the next code node appears to expect individual issue items.
- **Sub-workflow reference:** None.

#### Calculate Confidence Scores

- **Type and role:** `n8n-nodes-base.code` using Python  
  Intended to calculate a weighted confidence score for each issue.
- **Configuration choices:**
  - Uses weighted components:
    - severity 40%
    - volume 30%
    - frequency 20%
    - consistency 10%
  - Adds:
    - `confidence_score`
    - `confidence_breakdown`
- **Key expressions or variables used:**
  - `_items('all')`
  - issue fields expected:
    - `severity`
    - `historical_count`
    - `affected_rows`
    - `total_rows`
    - `detection_count`
- **Input and output connections:**
  - Input from `Combine All Issues`
  - Output to `Check Confidence Threshold`
- **Version-specific requirements:**
  - Code node version `2`
  - Python native support required.
- **Edge cases or potential failure types:**
  - Aggregate node sends one item with `issues` array, but this code iterates items as if each item itself were one issue.
  - Many expected fields are never produced upstream, so default values dominate the score.
  - Even if the code runs, confidence may be misleading because volume/frequency/consistency inputs are mostly absent.
- **Sub-workflow reference:** None.

#### Check Confidence Threshold

- **Type and role:** `n8n-nodes-base.if`  
  Filters results so only cases with at least one issue above the threshold continue.
- **Configuration choices:**
  - Condition 1: `$json.issues` is not empty
  - Condition 2: at least one issue in `issues` has `confidence_score >= confidenceThreshold`
- **Key expressions or variables used:**
  - `{{ $json.issues }}`
  - `{{ $json.issues.some(issue => issue.confidence_score >= $('Workflow Configuration').first().json.confidenceThreshold) }}`
- **Input and output connections:**
  - Input from `Calculate Confidence Scores`
  - True output connected to `Generate SQL Fixes`
  - False branch unused
- **Version-specific requirements:**
  - IF node version `2.3`
- **Edge cases or potential failure types:**
  - Depends on `issues` being an array after confidence scoring; that may not hold if prior node rewrites structure.
  - If confidence scores were attached to the wrong level, this check may always fail.
- **Sub-workflow reference:** None.

---

## 2.5 Remediation, Logging, Alerting & Baseline Maintenance

### Overview

This block generates suggested SQL fixes for confirmed issues, records them in PostgreSQL, sends Slack notifications, and writes baseline data back to PostgreSQL. The intended flow is operationally valuable, but field-name mismatches are significant.

### Nodes Involved

- Generate SQL Fixes
- Store Issue in Audit Log
- Send Alert to Team
- Update Baselines

### Node Details

#### Generate SQL Fixes

- **Type and role:** `n8n-nodes-base.code` using Python  
  Generates SQL remediation suggestions and rollback statements based on the issue type.
- **Configuration choices:**
  - Handles intended issue categories:
    - `schema_drift`
    - `null_explosion`
    - `outlier_distribution`
  - Adds:
    - `sql_fix`
    - `rollback_sql`
- **Key expressions or variables used:**
  - `_items('all')`
  - expects fields like:
    - `issue_type`
    - `details`
    - `null_percentage`
- **Input and output connections:**
  - Input from `Check Confidence Threshold`
  - Output to `Store Issue in Audit Log`
- **Version-specific requirements:**
  - Code node version `2`
  - Python native support required.
- **Edge cases or potential failure types:**
  - Upstream issues use `type`, not `issue_type`.
  - Schema-drift detector produces `change_description`, not `details.change_type`.
  - Outlier detector does not produce nested `details`.
  - If input is still one item containing `issues`, this node will not generate one SQL suggestion per issue as intended.
- **Sub-workflow reference:** None.

#### Store Issue in Audit Log

- **Type and role:** `n8n-nodes-base.postgres`  
  Inserts or maps issue information into the configured audit table.
- **Configuration choices:**
  - Table name is dynamic from config:
    - `$('Workflow Configuration').first().json.auditTableName`
  - Schema fixed to `public`
  - Column mapping:
    - `sql_fix`
    - `severity`
    - `issue_type` from `$json.type`
    - `table_name`
    - `column_name`
    - `description` from `$json.change_description`
    - `detected_at`
    - `confidence_score`
- **Key expressions or variables used:**
  - dynamic table reference via configuration node
- **Input and output connections:**
  - Input from `Generate SQL Fixes`
  - Output to `Send Alert to Team`
- **Version-specific requirements:**
  - Postgres node version `2.6`
- **Edge cases or potential failure types:**
  - Audit table must exist with compatible columns.
  - Description mapping only uses `change_description`; null/outlier issues may not populate a meaningful description.
  - Matching columns are defined, but operation behavior depends on table config and may not perform upsert unless explicitly supported/configured as such.
- **Sub-workflow reference:** None.

#### Send Alert to Team

- **Type and role:** `n8n-nodes-base.slack`  
  Sends a Slack message about the detected issue.
- **Configuration choices:**
  - Sends to a selected channel ID placeholder
  - Includes link to workflow
  - Message text contains:
    - severity
    - confidence score
    - table/column
    - issue details
    - SQL fix
    - detection time
    - issue ID
- **Key expressions or variables used:**
  - Placeholder channel ID/name:
    - `<__PLACEHOLDER_VALUE__Slack channel ID or name__>`
  - Message references:
    - `confidenceScore`
    - `table`
    - `column`
    - `issueDescription`
    - `sqlFix`
    - `detectedAt`
    - `issueId`
- **Input and output connections:**
  - Input from `Store Issue in Audit Log`
  - Output to `Update Baselines`
- **Version-specific requirements:**
  - Slack node version `2.4`
  - Requires Slack credentials and channel access.
- **Edge cases or potential failure types:**
  - The referenced JSON fields do not match upstream fields:
    - should likely be `confidence_score`, `table_name`, `column_name`, `change_description`, `sql_fix`, `detected_at`
  - Placeholder channel must be replaced.
  - Slack auth or channel permission failures.
- **Sub-workflow reference:** None.

#### Update Baselines

- **Type and role:** `n8n-nodes-base.postgres`  
  Writes baseline statistics back into the baseline table.
- **Configuration choices:**
  - Dynamic table name from config
  - Schema fixed to `public`
  - Writes fields including:
    - `avg_value`
    - `data_type`
    - `max_value`
    - `min_value`
    - `null_count`
    - `table_name`
    - `total_rows`
    - `column_name`
    - `schema_name`
    - `distinct_count`
    - `baseline_timestamp = $now`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').first().json.baselineTableName`
  - `{{ $now }}`
- **Input and output connections:**
  - Input from `Send Alert to Team`
  - No downstream output.
- **Version-specific requirements:**
  - Postgres node version `2.6`
- **Edge cases or potential failure types:**
  - Upstream path does not provide most of these baseline-stat fields.
  - Baseline read query expects `recorded_at`, while this node writes `baseline_timestamp`.
  - This placement means baselines update only after alerts, not after every scan.
  - The node appears to treat alert payloads as baseline rows, which is likely incorrect.
- **Sub-workflow reference:** None.

---

## 2.6 Documentation Sticky Notes

### Overview

These sticky notes describe the intended architecture and should be preserved because they explain the workflow’s conceptual blocks.

### Nodes Involved

- Sticky Note4
- Sticky Note6
- Sticky Note8
- Sticky Note10
- Sticky Note11
- Sticky Note12
- Sticky Note13

### Node Details

#### Sticky Note4

- **Type and role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for SQL suggestion generation.
- **Configuration choices:**
  - Content explains that high-confidence issues generate SQL suggestions.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6

- **Type and role:** `n8n-nodes-base.stickyNote`  
  High-level workflow description.
- **Configuration choices:**
  - Describes recurring monitoring, anomaly checks, confidence scoring, SQL remediation, logging, and Slack alerting.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note8

- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the final logging/alerting/baseline section.
- **Configuration choices:**
  - Notes storage in PostgreSQL, Slack alerts, and baseline updates.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note10

- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents scoring and filtering.
- **Configuration choices:**
  - Notes confidence scoring and threshold-based routing.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note11

- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the three detection branches.
- **Configuration choices:**
  - Lists schema drift, null explosions, outlier distributions.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note12

- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the metadata collection section.
- **Configuration choices:**
  - Notes schema metadata, table statistics, and historical baselines as reference inputs.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note13

- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the trigger/configuration block.
- **Configuration choices:**
  - Notes the 6-hour schedule.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule DB Quality Scan | Schedule Trigger | Starts the workflow every 6 hours |  | Workflow Configuration | ## Workflow Trigger & Configuration<br><br>Runs every 6 hours to monitor database quality. |
| Workflow Configuration | Set | Defines thresholds and table names used throughout the workflow | Schedule DB Quality Scan | Get Schema Metadata; Get Table Statistics; Get Historical Baselines | ## Workflow Trigger & Configuration<br><br>Runs every 6 hours to monitor database quality. |
| Get Schema Metadata | Postgres | Reads PostgreSQL schema metadata from `information_schema.columns` | Workflow Configuration | Detect Schema Drift | ## Database Metadata Collection<br><br>Fetches schema metadata, table statistics, and historical baseline records from PostgreSQL.<br>These datasets provide the reference points required to analyze schema changes, column behavior, and statistical deviations in the database. |
| Get Table Statistics | Postgres | Reads table-level statistics from `pg_stat_user_tables` | Workflow Configuration | Detect Null Explosions; Detect Outlier Distributions | ## Database Metadata Collection<br><br>Fetches schema metadata, table statistics, and historical baseline records from PostgreSQL.<br>These datasets provide the reference points required to analyze schema changes, column behavior, and statistical deviations in the database. |
| Get Historical Baselines | Postgres | Reads latest baseline record from the configured baseline table | Workflow Configuration |  | ## Database Metadata Collection<br><br>Fetches schema metadata, table statistics, and historical baseline records from PostgreSQL.<br>These datasets provide the reference points required to analyze schema changes, column behavior, and statistical deviations in the database. |
| Detect Schema Drift | Code | Detects schema changes against historical baseline data | Get Schema Metadata | Combine All Issues | ## Data Quality Detection Engine<br><br>Three parallel checks detect potential problems:<br>1>Schema drift (structure changes)<br>2>Null explosions (unexpected missing values)<br>3>Outlier distributions (statistical anomalies) |
| Detect Null Explosions | Code | Intended to detect excessive null rates in monitored tables/columns | Get Table Statistics | Combine All Issues | ## Data Quality Detection Engine<br><br>Three parallel checks detect potential problems:<br>1>Schema drift (structure changes)<br>2>Null explosions (unexpected missing values)<br>3>Outlier distributions (statistical anomalies) |
| Detect Outlier Distributions | Code | Intended to detect abnormal numeric distribution shifts | Get Table Statistics | Combine All Issues | ## Data Quality Detection Engine<br><br>Three parallel checks detect potential problems:<br>1>Schema drift (structure changes)<br>2>Null explosions (unexpected missing values)<br>3>Outlier distributions (statistical anomalies) |
| Combine All Issues | Aggregate | Aggregates all detected issues into a single `issues` array | Detect Schema Drift; Detect Null Explosions; Detect Outlier Distributions | Calculate Confidence Scores | ## Issue Scoring & Filtering<br><br>Each issue receives a confidence score based on severity, frequency, data volume affected, and consistency.<br>Only issues exceeding the configured confidence threshold proceed to remediation and alerting. |
| Calculate Confidence Scores | Code | Computes weighted confidence scores for issues | Combine All Issues | Check Confidence Threshold | ## Issue Scoring & Filtering<br><br>Each issue receives a confidence score based on severity, frequency, data volume affected, and consistency.<br>Only issues exceeding the configured confidence threshold proceed to remediation and alerting. |
| Check Confidence Threshold | IF | Allows only issue sets containing at least one high-confidence issue | Calculate Confidence Scores | Generate SQL Fixes | ## Issue Scoring & Filtering<br><br>Each issue receives a confidence score based on severity, frequency, data volume affected, and consistency.<br>Only issues exceeding the configured confidence threshold proceed to remediation and alerting. |
| Generate SQL Fixes | Code | Generates proposed SQL remediation and rollback statements | Check Confidence Threshold | Store Issue in Audit Log | ## Suggestions<br><br>For high-confidence issues, the workflow generates SQL suggestions for investigation or correction. |
| Store Issue in Audit Log | Postgres | Writes issue details into the audit table | Generate SQL Fixes | Send Alert to Team | ## Logging, Alerts & Baseline Updates<br><br>Confirmed issues are stored in a PostgreSQL audit table and alerts are sent to Slack.<br>Finally, the workflow updates baseline statistics to improve future anomaly detection accuracy. |
| Send Alert to Team | Slack | Sends a data-quality alert to a Slack channel | Store Issue in Audit Log | Update Baselines | ## Logging, Alerts & Baseline Updates<br><br>Confirmed issues are stored in a PostgreSQL audit table and alerts are sent to Slack.<br>Finally, the workflow updates baseline statistics to improve future anomaly detection accuracy. |
| Update Baselines | Postgres | Writes baseline statistics back to PostgreSQL | Send Alert to Team |  | ## Logging, Alerts & Baseline Updates<br><br>Confirmed issues are stored in a PostgreSQL audit table and alerts are sent to Slack.<br>Finally, the workflow updates baseline statistics to improve future anomaly detection accuracy. |
| Sticky Note4 | Sticky Note | Visual documentation for SQL suggestion generation |  |  |  |
| Sticky Note6 | Sticky Note | Visual overview of the whole workflow |  |  |  |
| Sticky Note8 | Sticky Note | Visual documentation for logging, alerting, and baseline updates |  |  |  |
| Sticky Note10 | Sticky Note | Visual documentation for scoring and filtering |  |  |  |
| Sticky Note11 | Sticky Note | Visual documentation for the detection engine |  |  |  |
| Sticky Note12 | Sticky Note | Visual documentation for metadata collection |  |  |  |
| Sticky Note13 | Sticky Note | Visual documentation for trigger and configuration |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a faithful reconstruction of the workflow as provided, followed by implementation notes where the current design will need correction to function properly.

## 4.1 Create the trigger and configuration

1. **Create a Schedule Trigger node**
   - Name it **Schedule DB Quality Scan**.
   - Set interval to **every 6 hours**.

2. **Create a Set node**
   - Name it **Workflow Configuration**.
   - Add these fields:
     - `confidenceThreshold` as Number = `0.85`
     - `maxNullPercentage` as Number = `0.15`
     - `outlierStdDevThreshold` as Number = `3`
     - `auditTableName` as String = `data_quality_audit`
     - `baselineTableName` as String = `data_quality_baselines`
   - Enable keeping other incoming fields.
   - Connect:
     - `Schedule DB Quality Scan -> Workflow Configuration`

## 4.2 Create PostgreSQL collection nodes

3. **Create a PostgreSQL node**
   - Name it **Get Schema Metadata**.
   - Operation: **Execute Query**
   - Configure PostgreSQL credentials.
   - Use this logical query:
     - select `table_name`, `column_name`, `data_type`, `is_nullable`
     - from `information_schema.columns`
     - where `table_schema` equals your target schema
     - order by `table_name`, ordinal position
   - Replace the placeholder schema name with a real schema string.
   - Connect:
     - `Workflow Configuration -> Get Schema Metadata`

4. **Create another PostgreSQL node**
   - Name it **Get Table Statistics**.
   - Operation: **Execute Query**
   - Configure the same PostgreSQL credentials.
   - Use this logical query:
     - select `schemaname`, `tablename`, `n_live_tup as row_count`, `n_dead_tup as dead_rows`
     - from `pg_stat_user_tables`
     - where `schemaname` equals your target schema
   - Replace the schema placeholder.
   - Connect:
     - `Workflow Configuration -> Get Table Statistics`

5. **Create a third PostgreSQL node**
   - Name it **Get Historical Baselines**.
   - Operation: **Execute Query**
   - Configure PostgreSQL credentials.
   - Use an expression-based query referencing the configuration node:
     - select all from the configured baseline table
     - order by `recorded_at` descending
     - limit 1
   - Connect:
     - `Workflow Configuration -> Get Historical Baselines`

## 4.3 Create anomaly detection nodes

6. **Create a Code node**
   - Name it **Detect Schema Drift**
   - Language: **Python**
   - Paste the provided schema-drift logic.
   - Connect:
     - `Get Schema Metadata -> Detect Schema Drift`

7. **Create a Code node**
   - Name it **Detect Null Explosions**
   - Language: **Python**
   - Paste the provided null-explosion logic.
   - Connect:
     - `Get Table Statistics -> Detect Null Explosions`

8. **Create a Code node**
   - Name it **Detect Outlier Distributions**
   - Language: **Python**
   - Paste the provided outlier-detection logic.
   - Connect:
     - `Get Table Statistics -> Detect Outlier Distributions`

## 4.4 Create aggregation and scoring nodes

9. **Create an Aggregate node**
   - Name it **Combine All Issues**
   - Aggregate mode: **Aggregate all item data**
   - Destination field: `issues`
   - Connect:
     - `Detect Schema Drift -> Combine All Issues`
     - `Detect Null Explosions -> Combine All Issues`
     - `Detect Outlier Distributions -> Combine All Issues`

10. **Create a Code node**
    - Name it **Calculate Confidence Scores**
    - Language: **Python**
    - Paste the provided confidence-scoring code.
    - Connect:
      - `Combine All Issues -> Calculate Confidence Scores`

11. **Create an IF node**
    - Name it **Check Confidence Threshold**
    - Add two AND conditions:
      1. `$json.issues` is not empty
      2. `$json.issues.some(issue => issue.confidence_score >= $('Workflow Configuration').first().json.confidenceThreshold)` is true
    - Connect:
      - `Calculate Confidence Scores -> Check Confidence Threshold`

## 4.5 Create remediation and output nodes

12. **Create a Code node**
    - Name it **Generate SQL Fixes**
    - Language: **Python**
    - Paste the provided SQL-fix generation code.
    - Connect the **true** branch:
      - `Check Confidence Threshold -> Generate SQL Fixes`

13. **Create a PostgreSQL node**
    - Name it **Store Issue in Audit Log**
    - Configure PostgreSQL credentials.
    - Set schema to `public`.
    - Set table name using expression:
      - `$('Workflow Configuration').first().json.auditTableName`
    - Map columns:
      - `sql_fix <- $json.sql_fix`
      - `severity <- $json.severity`
      - `issue_type <- $json.type`
      - `table_name <- $json.table_name`
      - `column_name <- $json.column_name`
      - `description <- $json.change_description`
      - `detected_at <- $json.detected_at`
      - `confidence_score <- $json.confidence_score`
    - Connect:
      - `Generate SQL Fixes -> Store Issue in Audit Log`

14. **Create a Slack node**
    - Name it **Send Alert to Team**
    - Configure Slack credentials.
    - Choose operation to send a message to a channel.
    - Set channel using your real Slack channel ID or name.
    - Enable “include link to workflow”.
    - Use the provided message template, or a corrected version if you want it to work.
    - Connect:
      - `Store Issue in Audit Log -> Send Alert to Team`

15. **Create a PostgreSQL node**
    - Name it **Update Baselines**
    - Configure PostgreSQL credentials.
    - Set schema to `public`.
    - Set table using expression:
      - `$('Workflow Configuration').first().json.baselineTableName`
    - Map fields:
      - `avg_value`
      - `data_type`
      - `max_value`
      - `min_value`
      - `null_count`
      - `table_name`
      - `total_rows`
      - `column_name`
      - `schema_name`
      - `distinct_count`
      - `baseline_timestamp <- $now`
    - Connect:
      - `Send Alert to Team -> Update Baselines`

## 4.6 Create sticky notes

16. Add these sticky notes with the corresponding content:
   - **Workflow Trigger & Configuration**
   - **Database Metadata Collection**
   - **Data Quality Detection Engine**
   - **Issue Scoring & Filtering**
   - **Suggestions**
   - **Logging, Alerts & Baseline Updates**
   - **Overall workflow description**

## 4.7 Credentials required

17. **PostgreSQL credential**
   - Required by:
     - Get Schema Metadata
     - Get Table Statistics
     - Get Historical Baselines
     - Store Issue in Audit Log
     - Update Baselines
   - Must have permission to:
     - query `information_schema.columns`
     - query `pg_stat_user_tables`
     - read baseline table
     - write audit and baseline tables

18. **Slack credential**
   - Required by:
     - Send Alert to Team
   - Must have permission to post to the target channel.

## 4.8 Required database objects

19. Create or verify an **audit table** matching the write fields:
   - `issue_type`
   - `table_name`
   - `column_name`
   - `severity`
   - `confidence_score`
   - `description`
   - `sql_fix`
   - `detected_at`

20. Create or verify a **baseline table** matching the read/write expectations.
   - Important: the current workflow is inconsistent about timestamp naming:
     - read query expects `recorded_at`
     - write node uses `baseline_timestamp`
   - Standardize this before production use.

## 4.9 Corrections needed for a working implementation

To make the workflow operational, a developer should also apply these fixes:

21. **Replace placeholder values**
   - target PostgreSQL schema name
   - Slack channel ID or name

22. **Normalize field names**
   - choose either `tablename` or `table_name` and use it consistently
   - choose either `type` or `issue_type`
   - choose either `detected_at` or `detectedAt`
   - choose either `sql_fix` or `sqlFix`

23. **Pass baseline data into detection nodes**
   - Add Merge nodes or redesign the data flow so:
     - `Detect Schema Drift` receives both current schema and historical baseline
     - `Detect Outlier Distributions` receives current stats, config, and baseline data

24. **Provide the data actually needed by null and outlier checks**
   - Replace `pg_stat_user_tables` query with per-column profiling queries if you want:
     - null counts
     - total row counts
     - means
     - stddev
     - min/max
     - distinct counts
     - column type

25. **Fix confidence scoring input shape**
   - Either:
     - score each issue before aggregating, or
     - unpack the `issues` array inside the code node, score each element, then return the array

26. **Fix SQL fix generation**
   - Use the real fields emitted by detectors.
   - For example, use `$json.type` instead of `$json.issue_type` unless you standardize earlier.

27. **Fix Slack message field references**
   - Current template refers to missing fields.
   - Use actual upstream fields such as:
     - severity
     - confidence_score
     - table_name
     - column_name
     - change_description
     - sql_fix
     - detected_at

28. **Separate baseline updates from alert payloads**
   - Baseline refresh should likely come from fresh computed profiling stats, not from Slack/audit output rows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Autonomous Data Quality Monitoring and Remediation for Production Databases. This workflow continuously monitors PostgreSQL database health by running automated data quality checks every six hours. It retrieves schema metadata, table statistics, and historical baselines to detect structural and statistical anomalies. The workflow analyzes three major issue types: schema drift, null value explosions, and abnormal data distributions. All detected issues are aggregated and evaluated using a confidence scoring system based on severity, frequency, and data impact. If an issue exceeds the defined confidence threshold, the workflow automatically generates SQL remediation suggestions, logs the issue in a database audit table, and sends alerts to Slack for team awareness. | Overall workflow note |
| Suggestions: For high-confidence issues, the workflow generates SQL suggestions for investigation or correction. | SQL remediation section |
| Logging, Alerts & Baseline Updates: Confirmed issues are stored in a PostgreSQL audit table and alerts are sent to Slack. Finally, the workflow updates baseline statistics to improve future anomaly detection accuracy. | Output section |
| Issue Scoring & Filtering: Each issue receives a confidence score based on severity, frequency, data volume affected, and consistency. Only issues exceeding the configured confidence threshold proceed to remediation and alerting. | Scoring section |
| Data Quality Detection Engine: Three parallel checks detect potential problems: schema drift, null explosions, outlier distributions. | Detection section |
| Database Metadata Collection: Fetches schema metadata, table statistics, and historical baseline records from PostgreSQL. These datasets provide the reference points required to analyze schema changes, column behavior, and statistical deviations in the database. | Collection section |
| Workflow Trigger & Configuration: Runs every 6 hours to monitor database quality. | Trigger section |

If you want, I can also produce a **corrected implementation specification** for this workflow that preserves the same intent but fixes the broken data mappings and missing merge logic.