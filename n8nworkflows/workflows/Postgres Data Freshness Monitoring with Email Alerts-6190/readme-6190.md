Postgres Data Freshness Monitoring with Email Alerts

https://n8nworkflows.xyz/workflows/postgres-data-freshness-monitoring-with-email-alerts-6190


# Postgres Data Freshness Monitoring with Email Alerts

### 1. Workflow Overview

This workflow, titled **Postgres Data Freshness Monitoring with Email Alerts**, automates the monitoring of data freshness across multiple tables in a PostgreSQL database. Its main goal is to ensure that monitored tables are regularly updated, and to send email alerts when data in any table becomes stale beyond a configurable threshold (default 3 days).

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger:** Periodically triggers the workflow run on a defined schedule (default: every day).
- **1.2 Produce Tables and Timestamp Columns:** Defines which tables and their associated timestamp columns are monitored.
- **1.3 Data Freshness Calculation Loop:** Iterates over each table, queries the most recent row based on the timestamp column, and calculates the lag in days since the last update.
- **1.4 Filter Stale Tables:** Filters out tables that are still fresh (updated within the last 3 days by default).
- **1.5 Aggregate and Alert:** Aggregates all stale tables and triggers an alert workflow that sends an email notification listing the stale tables.

Supporting this logic are several sticky notes providing documentation, usage instructions, and customization hints.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Initiates the workflow at regular intervals automatically.
- **Nodes Involved:** `Schedule Trigger`
- **Node Details:**
  - **Type:** Schedule Trigger
  - **Role:** Starts workflow runs on a schedule.
  - **Configuration:** Configured with a default interval of once per day.
  - **Expressions:** None.
  - **Input:** None (trigger node).
  - **Output:** Connected to `Produce tables + date columns`.
  - **Version:** 1.2
  - **Potential Failures:** Scheduler misconfiguration, n8n server downtime.
  - **Notes:** Fundamental for automation; no manual input required.

#### 1.2 Produce Tables and Timestamp Columns

- **Overview:** Produces a list of tables paired with their respective timestamp columns to monitor.
- **Nodes Involved:** `Produce tables + date columns`
- **Node Details:**
  - **Type:** Code Node (JavaScript)
  - **Role:** Outputs an array of objects each containing `tableName` and `timestampColumn`.
  - **Configuration:** Hardcoded array:
    - `sleep_summary` with timestamp column `sleep_date`
    - `mac_screentime_sessions` with timestamp column `ts`
  - **Expressions:** JavaScript code returning structured data.
  - **Input:** Triggered by `Schedule Trigger`.
  - **Output:** Connected to `Loop Over Items`.
  - **Version:** 2
  - **Potential Failures:** Code errors, empty array leading to no monitoring.
  - **Customization:** Users must edit this node to add/remove tables or change timestamp columns as per their database schema.

#### 1.3 Data Freshness Calculation Loop

- **Overview:** Iterates over each table to check data freshness.
- **Nodes Involved:** `Loop Over Items`, `Get most recent row from table`, `Calculate lag`, `Add back table name`
- **Node Details:**

  - `Loop Over Items`:
    - **Type:** SplitInBatches Node
    - **Role:** Processes each table individually.
    - **Configuration:** Default batching options.
    - **Input:** From `Produce tables + date columns`.
    - **Output:** Two branches:
      - To `Get most recent row from table` (main)
      - To `Remove fresh tables` (secondary output unused here)
    - **Version:** 3
    - **Potential Failures:** Large dataset performance issues.

  - `Get most recent row from table`:
    - **Type:** Postgres Node
    - **Role:** Queries the most recent row from the current table based on the timestamp column.
    - **Configuration:**
      - Operation: Select
      - Table name and timestamp column dynamically set via expressions from current item (`$json.tableName` and `$json.timestampColumn`).
      - Sort descending by timestamp column.
      - Limit to 1 row.
      - Schema: fixed to `public`.
      - Credentials: Uses stored Postgres credentials named "Quantified Life Postgres Account".
    - **Expressions:** Dynamic table and column names.
    - **Input:** From `Loop Over Items`.
    - **Output:** To `Calculate lag`.
    - **Version:** 2.6
    - **Potential Failures:** Database connection errors, table or column not existing, empty table returns no rows.

  - `Calculate lag`:
    - **Type:** DateTime Node
    - **Role:** Calculates difference in days between now and the timestamp of the latest row.
    - **Configuration:**
      - Operation: Get time between dates.
      - Start date: Timestamp from the queried row (`$json[$('Produce tables + date columns').item.json.timestampColumn]`).
      - End date: Current time (`$now`).
    - **Expressions:** Dynamic referencing of the timestamp column.
    - **Input:** From `Get most recent row from table`.
    - **Output:** To `Add back table name`.
    - **Version:** 2
    - **Potential Failures:** Null or invalid timestamp values, expression evaluation errors.

  - `Add back table name`:
    - **Type:** Set Node
    - **Role:** Adds `daysOld` (number) and `table` (string) properties back into the data stream.
    - **Configuration:** 
      - `daysOld` assigned from `$json.timeDifference.days`.
      - `table` assigned from the original table name in the `Produce tables + date columns` node.
    - **Input:** From `Calculate lag`.
    - **Output:** To `Loop Over Items` (creating a loop) for further processing.
    - **Version:** 3.4
    - **Potential Failures:** Data mismatch, missing properties.

#### 1.4 Filter Stale Tables

- **Overview:** Filters out tables updated within the last 3 days; retains only stale tables.
- **Nodes Involved:** `Remove fresh tables`
- **Node Details:**
  - **Type:** Filter Node
  - **Role:** Removes tables with `daysOld` less than 3.
  - **Configuration:** Condition: `daysOld >= 3`.
  - **Input:** From `Loop Over Items`.
  - **Output:** To `Aggregate Stale Tables`.
  - **Version:** 2.2
  - **Potential Failures:** Incorrect threshold configuration, filtering all tables by mistake.

#### 1.5 Aggregate and Alert

- **Overview:** Aggregates all stale tables into a list and triggers an alert workflow.
- **Nodes Involved:** `Aggregate Stale Tables`, `Send alerts`
- **Node Details:**

  - `Aggregate Stale Tables`:
    - **Type:** Aggregate Node
    - **Role:** Aggregates the `table` field from all stale tables into a list named `tables`.
    - **Configuration:** Aggregation by collecting the `table` field values.
    - **Input:** From `Remove fresh tables`.
    - **Output:** To `Send alerts`.
    - **Version:** 1
    - **Potential Failures:** Empty input leads to empty alert.

  - `Send alerts`:
    - **Type:** Execute Workflow Node
    - **Role:** Calls an external workflow responsible for sending the email alert.
    - **Configuration:**
      - Workflow ID: `17yjHH5i669dVRGk` (expected to be an alerting workflow).
      - Inputs: 
        - Subject: "Stale Quantified Life Data"
        - Body: Message listing stale tables joined by newline.
      - Mapping mode: Explicitly defined input fields.
    - **Input:** From `Aggregate Stale Tables`.
    - **Output:** None (end node).
    - **Version:** 1.2
    - **Potential Failures:** Called workflow missing or misconfigured, email sending failure, empty table list.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                              | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                               |
|-------------------------------|-------------------------|----------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger        | Triggers workflow on schedule                 | None                        | Produce tables + date columns|                                                                                                          |
| Produce tables + date columns  | Code                    | Produces list of tables and timestamp columns | Schedule Trigger            | Loop Over Items              |                                                                                                          |
| Loop Over Items               | SplitInBatches          | Iterates over tables for freshness check     | Produce tables + date columns, Add back table name | Get most recent row from table, Remove fresh tables |                                                                                                          |
| Get most recent row from table| Postgres                | Queries most recent row per table             | Loop Over Items             | Calculate lag               |                                                                                                          |
| Calculate lag                 | DateTime                | Calculates age in days of last update         | Get most recent row from table | Add back table name        |                                                                                                          |
| Add back table name           | Set                     | Adds daysOld and table name to data           | Calculate lag               | Loop Over Items             |                                                                                                          |
| Remove fresh tables           | Filter                  | Filters out tables updated within 3 days     | Loop Over Items             | Aggregate Stale Tables      |                                                                                                          |
| Aggregate Stale Tables        | Aggregate               | Aggregates stale table names into list        | Remove fresh tables         | Send alerts                |                                                                                                          |
| Send alerts                  | Execute Workflow        | Calls alert workflow to send email notification| Aggregate Stale Tables       | None                       | Recommended alerting workflow: https://creators.n8n.io/workflows/6189                                   |
| Sticky Note                  | Sticky Note             | Documentation: "Find tables with stale data"  | None                        | None                       |                                                                                                          |
| Sticky Note1                 | Sticky Note             | Extended documentation and usage instructions | None                        | None                       |                                                                                                          |
| Sticky Note2                 | Sticky Note             | Optional customization note for alert workflow| None                        | None                       | Recommended alerting workflow: https://creators.n8n.io/workflows/6189                                   |
| Sticky Note3                 | Sticky Note             | Optional customization note for stale threshold| None                        | None                       |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**
   - Type: Schedule Trigger
   - Configure interval to desired frequency (e.g., every day).
   - No credentials required.
   - Position it as the start node.

2. **Create `Produce tables + date columns` node:**
   - Type: Code
   - Add JavaScript code to output an array of objects with keys `tableName` and `timestampColumn`.
   - Example code:
     ```js
     const VALUE_TUPLES = [
       ["sleep_summary","sleep_date"],
       ["mac_screentime_sessions", "ts"],
     ];
     let result = [];
     for (const [tableName, timestampCol] of VALUE_TUPLES) {
       result.push({ "tableName": tableName, "timestampColumn": timestampCol });
     }
     return result;
     ```
   - Connect output of `Schedule Trigger` to this node.

3. **Create `Loop Over Items` node:**
   - Type: SplitInBatches
   - Use default options.
   - Connect output of `Produce tables + date columns` to this node.

4. **Create `Get most recent row from table` node:**
   - Type: Postgres
   - Credentials: Set up with your Postgres credentials (e.g., "Quantified Life Postgres Account").
   - Operation: Select
   - Table: Use expression `={{ $json.tableName }}`
   - Schema: `public` (or your schema)
   - Limit: 1
   - Sort: Column `={{ $json.timestampColumn }}` descending
   - Connect output of `Loop Over Items` (main) to this node.

5. **Create `Calculate lag` node:**
   - Type: DateTime
   - Operation: Get time between dates
   - Start Date: Expression `={{ $json[$('Produce tables + date columns').item.json.timestampColumn] }}`
   - End Date: Expression `={{ $now }}`
   - Connect output of `Get most recent row from table` to this node.

6. **Create `Add back table name` node:**
   - Type: Set
   - Add fields:
     - `daysOld` (number) = `={{ $json.timeDifference.days }}`
     - `table` (string) = `={{ $('Produce tables + date columns').item.json.tableName }}`
   - Connect output of `Calculate lag` to this node.

7. **Connect output of `Add back table name` back to `Loop Over Items`:**
   - This creates a loop allowing further processing of each table’s freshness data.

8. **Create `Remove fresh tables` node:**
   - Type: Filter
   - Condition: `daysOld >= 3`
   - Connect output of `Loop Over Items` (secondary output) to this node.

9. **Create `Aggregate Stale Tables` node:**
   - Type: Aggregate
   - Aggregate field: `table`
   - Output field name: `tables`
   - Connect output of `Remove fresh tables` to this node.

10. **Create `Send alerts` node:**
    - Type: Execute Workflow
    - Configure to call an alerting workflow (e.g., workflow ID `17yjHH5i669dVRGk`).
    - Pass inputs:
      - `subject`: "Stale Quantified Life Data"
      - `body`: `"The following quantified life tables are stale (>=3 days old):\n{{ $json.tables.join('\n') }}"`
    - Connect output of `Aggregate Stale Tables` to this node.
    - Ensure the called workflow is configured to send emails or alerts.

11. **Add Sticky Notes for documentation and customization hints as desired.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Workflow monitors PostgreSQL tables for freshness and sends alerts if data is stale for >=3 days.                                | Workflow purpose description                                                              |
| Each table must have a timestamp column tracking updates (e.g., `timestamp`, `last_updated`).                                    | Database schema requirement                                                               |
| Recommended alerting workflow for easy plug-and-play email alerts: https://creators.n8n.io/workflows/6189                        | Alerting workflow used in `Send alerts` node                                              |
| To customize tables or timestamp columns, modify the `Produce tables + date columns` node’s JavaScript code.                    | Customization instructions                                                                 |
| To adjust staleness threshold, modify the condition in the `Remove fresh tables` node (default is 3 days).                      | Customization instructions                                                                 |
| The workflow includes extensive documentation in sticky notes for ease of use and understanding.                                | Documentation nodes inside the workflow                                                   |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.