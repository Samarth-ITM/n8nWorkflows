Purge n8n execution history located in Mysql

https://n8nworkflows.xyz/workflows/purge-n8n-execution-history-located-in-mysql-700


# Purge n8n execution history located in Mysql

### 1. Workflow Overview

This workflow is designed to maintain and optimize the size of the n8n execution history stored in a MySQL database (or any SQL-based database like PostgreSQL or MongoDB with adapted queries). It targets scenarios where many workflows run frequently, generating a large volume of execution logs that can degrade database performance over time.

The logical blocks included are:

- **1.1 Manual and Scheduled Triggering:** Two ways to initiate the purge — manually via a trigger node or automatically via a daily scheduled cron job.
- **1.2 Database Cleanup Execution:** A MySQL node that performs the deletion of execution history entries older than 30 days, based on the `stoppedAt` timestamp column.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual and Scheduled Triggering

- **Overview:**  
  This block allows the workflow purge to be started either manually by a user or automatically on a daily schedule. This flexibility supports ad-hoc cleanups and automated maintenance during low-traffic periods.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Cron (Scheduled Trigger)

- **Node Details:**

  - **On clicking 'execute'**  
    - *Type and Role:* Manual Trigger node; initiates workflow execution on user demand.  
    - *Configuration:* No parameters; simply waits for user to press "Execute".  
    - *Expressions/Variables:* None.  
    - *Input/Output:* No input; outputs a trigger event to the MySQL node.  
    - *Version Requirements:* Compatible with n8n version 0.120+ (typical for manual triggers).  
    - *Edge Cases:* Manual execution may be forgotten or run too frequently; no safeguards for concurrency.  
    - *Sub-workflow:* None.

  - **Cron**  
    - *Type and Role:* Cron Trigger node; schedules automatic workflow execution.  
    - *Configuration:* Set to trigger daily at 07:00 (7 AM).  
    - *Expressions/Variables:* Static time configuration.  
    - *Input/Output:* No input; outputs trigger event to MySQL node.  
    - *Version Requirements:* Standard across n8n versions supporting Cron node.  
    - *Edge Cases:* Server time zone discrepancies could shift execution time; if n8n is down at trigger time, the execution is missed unless retry configured.  
    - *Sub-workflow:* None.

#### 2.2 Database Cleanup Execution

- **Overview:**  
  This block executes an SQL query that deletes rows from the `execution_entity` table where the `stoppedAt` date is older than 30 days. This effectively purges stale workflow execution history, preventing uncontrolled growth of the execution log table.

- **Nodes Involved:**  
  - MySQL

- **Node Details:**

  - **MySQL**  
    - *Type and Role:* MySQL node; performs SQL query execution against the configured MySQL database.  
    - *Configuration:*  
      - Operation: Execute Query  
      - Query:  
        ```sql
        DELETE FROM execution_entity 
        WHERE DATE(stoppedAt) < DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ```  
      - Credentials: Uses stored MySQL credentials named "n8n".  
    - *Expressions/Variables:* Static SQL query; no dynamic expressions used.  
    - *Input/Output:* Receives trigger from either Manual Trigger or Cron node; outputs query execution result (number of affected rows).  
    - *Version Requirements:* Requires n8n version with MySQL node supporting executeQuery operation.  
    - *Edge Cases:*  
      - Authentication errors if credentials are invalid or expired.  
      - Query failures if table schema changes or if the `execution_entity` table is locked or missing.  
      - Potential impact on running workflows if the table is heavily locked during deletion.  
      - Timezone issues if `stoppedAt` stored in different timezone than `CURDATE()`.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role            | Input Node(s)          | Output Node(s) | Sticky Note                                                                                              |
|---------------------|----------------------|----------------------------|-----------------------|----------------|--------------------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger       | Initiates manual execution  | -                     | MySQL          |                                                                                                        |
| Cron                | Cron Trigger         | Initiates scheduled execution | -                   | MySQL          | Configure cron to run once per day at a low traffic hour (e.g., 7 AM)                                   |
| MySQL               | MySQL                | Executes purge SQL query    | On clicking 'execute', Cron | -          | SQL query deletes entries older than 30 days using `stoppedAt` column; connection must be properly set |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a node of type "Manual Trigger" called "On clicking 'execute'."  
   - No additional configuration needed.

2. **Create Cron Trigger Node:**  
   - Add a node of type "Cron."  
   - Set trigger time: Hour = 7 (7 AM daily).  
   - Leave other fields as default.

3. **Create MySQL Node:**  
   - Add a node of type "MySQL."  
   - Set operation to "Execute Query."  
   - Enter the SQL query:  
     ```sql
     DELETE FROM execution_entity 
     WHERE DATE(stoppedAt) < DATE_SUB(CURDATE(), INTERVAL 30 DAY)
     ```  
   - Configure MySQL credentials: Create or select credentials that allow access to the n8n database.  
   - Ensure credentials have DELETE permissions on `execution_entity` table.

4. **Connect Nodes:**  
   - Connect "On clicking 'execute'" output to the MySQL node input.  
   - Connect "Cron" output to the MySQL node input as well (parallel triggers).

5. **Activate Workflow:**  
   - Ensure the workflow is active so the cron trigger can execute automatically.  
   - Optionally, test manual execution by clicking execute on "On clicking 'execute'" node.

6. **Scheduling and Maintenance:**  
   - Optionally configure n8n instance time zone to align cron execution with expected low-traffic hours.  
   - Monitor execution logs for any errors related to database connectivity or query execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                               |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Frequent workflow executions can generate ~1024 rows per day per task, rapidly increasing table size.          | Workflow description                           |
| Recommended to run purge once daily during off-peak hours to minimize performance impact.                      | Workflow description and cron node sticky note|
| SQL query uses `stoppedAt` column as a reliable timestamp reference for deletion cutoff.                       | SQL query node configuration                   |
| Credentials must be correctly configured for MySQL node with appropriate permissions to delete records.        | MySQL node credentials                         |
| Adjust `INTERVAL 30 DAY` in SQL query to change retention period as required.                                  | SQL query customization                        |