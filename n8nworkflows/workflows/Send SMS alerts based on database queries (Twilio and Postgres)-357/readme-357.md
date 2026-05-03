Send SMS alerts based on database queries (Twilio and Postgres)

https://n8nworkflows.xyz/workflows/send-sms-alerts-based-on-database-queries--twilio-and-postgres--357


# Send SMS alerts based on database queries (Twilio and Postgres)

### 1. Workflow Overview

This workflow automates the monitoring of sensor readings stored in a Postgres database to detect outlier values and send SMS alerts via Twilio. It is designed for continuous database activity monitoring and alerting, running every minute to query the database for sensor readings exceeding a threshold and not yet notified. The workflow then sends SMS notifications and updates the database to mark these readings as alerted.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Periodically initiates the workflow every minute using a Cron node.
- **1.2 Database Query:** Retrieves outlier sensor readings without prior notifications from Postgres.
- **1.3 Alert Sending:** Sends SMS alerts for each outlier reading via Twilio.
- **1.4 Notification Update:** Marks the alerted readings in the database as notified to avoid duplicate alerts.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow execution every minute to ensure continuous monitoring.

- **Nodes Involved:**  
  - Cron

- **Node Details:**  
  - **Cron**  
    - Type: Core Trigger Node  
    - Configuration: Uses default settings to trigger every minute (default Cron parameters imply every minute if not explicitly set).  
    - Input: None (trigger node)  
    - Output: Triggers downstream Postgres node  
    - Expressions/Variables: None  
    - Version: 1  
    - Edge Cases: If the node is disabled or n8n is down, no triggers occur; no error handling needed here.  
    - Sub-workflow: None

#### 1.2 Database Query

- **Overview:**  
  Executes a SQL query to select sensor readings with a value greater than 70 and where no notification has been sent (notification = false).

- **Nodes Involved:**  
  - Postgres

- **Node Details:**  
  - **Postgres**  
    - Type: Database Node  
    - Configuration:  
      - Operation: Execute Query  
      - Query: `SELECT * FROM n8n WHERE value > 70 AND notification = false;`  
      - Credentials: Uses stored Postgres credentials.  
    - Input: Triggered by Cron node  
    - Output: Passes retrieved records to Twilio node, one per item (if multiple results)  
    - Expressions/Variables: None (static query)  
    - Version: 1  
    - Edge Cases:  
      - Database connectivity issues (network, credentials)  
      - Empty query result (no outliers) — workflow proceeds but no SMS sent  
      - Query syntax errors (unlikely unless DB schema changes)  
    - Sub-workflow: None

#### 1.3 Alert Sending

- **Overview:**  
  Sends an SMS alert for each outlier sensor reading, composing a message with sensor ID and reading value.

- **Nodes Involved:**  
  - Twilio

- **Node Details:**  
  - **Twilio**  
    - Type: Communication Node (SMS)  
    - Configuration:  
      - To: Phone number to receive SMS (currently empty, to be filled by user)  
      - From: Twilio phone number (empty here, to be configured)  
      - Message: Dynamically composed string:  
        `🚨 The Sensor ({sensor_id}) showed a reading of {value}.`  
        Uses expressions to insert `sensor_id` and `value` from Postgres node output, e.g., `{{$node["Postgres"].json["sensor_id"]}}`  
      - Credentials: Uses Twilio API credentials  
    - Input: Receives each record from Postgres node (one per outlier)  
    - Output: Passes data to Set node for notification update  
    - Version: 1  
    - Edge Cases:  
      - Missing or invalid phone number (`to`) or Twilio number (`from`) configuration leads to send failure  
      - Twilio API rate limits or downtime  
      - Message composition errors if Postgres node output is missing fields  
    - Sub-workflow: None

#### 1.4 Notification Update

- **Overview:**  
  Updates the database to mark the sensor reading's `notification` field as `true` to prevent repeated alerts.

- **Nodes Involved:**  
  - Set  
  - Postgres1

- **Node Details:**  
  - **Set**  
    - Type: Data Preparation Node  
    - Configuration:  
      - Sets two fields:  
        - `id` (number): set from Postgres node’s `id` field via expression `={{$node["Postgres"].json["id"]}}`  
        - `notification` (boolean): set to `true`  
      - Option: `Keep Only Set` enabled to clear other data  
    - Input: Receives output of Twilio node (one record per alert sent)  
    - Output: Passes prepared data to Postgres1 node for update  
    - Version: 1  
    - Edge Cases:  
      - Missing `id` value causes update failure  
      - Data type mismatches (should not occur due to explicit typing)  
    - Sub-workflow: None

  - **Postgres1**  
    - Type: Database Node  
    - Configuration:  
      - Operation: Update  
      - Table: `n8n`  
      - Columns to update: `notification`  
      - Uses input data from Set node for `id` and new `notification` value  
      - Credentials: Postgres credentials again  
    - Input: Receives data from Set node  
    - Output: None (end of workflow branch)  
    - Version: 1  
    - Edge Cases:  
      - Database connection or permission errors  
      - Update operation fails if `id` does not exist or is null  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name | Node Type           | Functional Role                | Input Node(s) | Output Node(s) | Sticky Note                                                                                       |
|-----------|---------------------|-------------------------------|---------------|----------------|-------------------------------------------------------------------------------------------------|
| Cron      | Cron Trigger        | Scheduled execution every minute | None          | Postgres       |                                                                                                 |
| Postgres  | Postgres DB         | Query outlier sensor readings  | Cron          | Twilio         |                                                                                                 |
| Twilio    | SMS (Twilio)        | Send alert SMS for outlier     | Postgres      | Set            |                                                                                                 |
| Set       | Set Data            | Prepare update data (id, notification) | Twilio        | Postgres1      |                                                                                                 |
| Postgres1 | Postgres DB         | Update notification status     | Set           | None           |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node:**  
   - Add a Cron node named `Cron`.  
   - Configure it to trigger every 1 minute (default settings suffice).  
   - No credentials needed.

2. **Create Postgres Query Node:**  
   - Add a Postgres node named `Postgres`.  
   - Set operation to `Execute Query`.  
   - Enter SQL query:  
     ```sql
     SELECT * FROM n8n WHERE value > 70 AND notification = false;
     ```  
   - Set credentials to your configured Postgres connection.  
   - Connect output of `Cron` node to input of `Postgres`.

3. **Create Twilio Node:**  
   - Add a Twilio node named `Twilio`.  
   - Configure `To` with the phone number to receive alerts (e.g., `+1234567890`).  
   - Configure `From` with your Twilio phone number.  
   - Set message to:  
     ```
     🚨 The Sensor ({{$node["Postgres"].json["sensor_id"]}}) showed a reading of {{$node["Postgres"].json["value"]}}.
     ```  
   - Set credentials with your Twilio API credentials.  
   - Connect output of `Postgres` node to input of `Twilio`.

4. **Create Set Node:**  
   - Add a Set node named `Set`.  
   - Enable `Keep Only Set`.  
   - Add two fields:  
     - `id` (Number): Set value to `={{$node["Postgres"].json["id"]}}`  
     - `notification` (Boolean): Set value to `true`  
   - Connect output of `Twilio` node to input of `Set`.

5. **Create Postgres Update Node:**  
   - Add a Postgres node named `Postgres1`.  
   - Set operation to `Update`.  
   - Set table to `n8n`.  
   - Set columns to update to `notification`.  
   - Use credentials for Postgres.  
   - Connect output of `Set` node to input of `Postgres1`.

6. **Activate the Workflow:**  
   - Ensure all nodes have valid credentials.  
   - Save and activate the workflow to run every minute automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow is Workflow 2 in the blog tutorial "Database activity monitoring and alerting".                  | https://blog.n8n.io/database-monitoring-and-alerting-with-n8n/                                              |
| Requires configured Postgres and Twilio credentials in n8n for database access and SMS sending respectively.  | https://docs.n8n.io/integrations/credentials/postgres/, https://docs.n8n.io/integrations/credentials/twilio/ |
| Make sure to set the phone numbers (`to` and `from`) in the Twilio node to valid E.164 format numbers.        | Twilio SMS guidelines                                                                                        |
| For production, consider adding error handling or notifications if Twilio or Postgres nodes fail.              | Best practices for resilient workflows                                                                       |