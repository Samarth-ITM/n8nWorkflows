Real-Time Flight Tracking with Aviation API, Slack & SendGrid/Twilio Alerts

https://n8nworkflows.xyz/workflows/real-time-flight-tracking-with-aviation-api--slack---sendgrid-twilio-alerts-6227


# Real-Time Flight Tracking with Aviation API, Slack & SendGrid/Twilio Alerts

### 1. Workflow Overview

This workflow automates real-time monitoring and notification of flight schedule changes using aviation data APIs. It is designed for airline operations teams to track updates continuously and inform affected passengers promptly via email and SMS. The workflow integrates data fetching, database comparison, notification routing, and logging, structured into the following logical blocks:

- **1.1 Scheduled Data Retrieval:** Periodically triggers the workflow and fetches flight data from an external aviation API, then retrieves current internal schedule records from a Postgres database.

- **1.2 Change Detection and Processing:** Compares the fetched flight data against stored schedules to detect differences in departure/arrival times, status, or gate, and assesses severity levels.

- **1.3 Database and Team Notifications:** If changes exist, updates the database and sends detailed update messages to the flight operations Slack channel, while syncing information with internal systems and logging the sync event.

- **1.4 Passenger Notification Management:** For urgent changes (high or critical severity), identifies affected passengers from the database and sends personalized email notifications via SendGrid, followed by critical SMS alerts via Twilio.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

- **Overview:**  
  This block triggers the workflow every 30 minutes, fetches the latest flight data from the Aviation Stack API, and retrieves recent flight schedule records from the internal Postgres database for comparison.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch Airline Data  
  - Get Current Schedules

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution every 30 minutes.  
    - Configuration: Interval set to 30 minutes.  
    - Inputs: None (trigger node)  
    - Outputs: Triggers "Fetch Airline Data"  
    - Edge cases: Workflow downtime could cause missed triggers; ensure n8n instance reliability.

  - **Fetch Airline Data**  
    - Type: HTTP Request  
    - Role: Fetches real-time flight data from Aviation Stack API.  
    - Configuration:  
      - Method: GET (default)  
      - URL: `https://api.aviationstack.com/v1/flights`  
      - Authentication: HTTP Query Auth using API key credential.  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Passes API response to "Get Current Schedules"  
    - Edge cases: API rate limits, network failures, invalid API keys, malformed responses.

  - **Get Current Schedules**  
    - Type: Postgres  
    - Role: Queries internal flight schedule records updated within the last hour.  
    - Configuration:  
      - SQL Query: Select flight details (number, times, status, gate, terminal) updated in last hour.  
      - Credentials: Postgres connection.  
    - Inputs: API data from Fetch Airline Data  
    - Outputs: Provides DB data and API data for comparison to "Process Changes"  
    - Edge cases: Database connectivity issues, query errors, empty results.

---

#### 2.2 Change Detection and Processing

- **Overview:**  
  This block compares the latest API flight data with internal database records to identify schedule changes such as time shifts, status updates, or gate changes. It classifies changes by severity and prepares notifications accordingly.

- **Nodes Involved:**  
  - Process Changes  
  - Check for Changes

- **Node Details:**

  - **Process Changes**  
    - Type: Code (JavaScript)  
    - Role: Compares API flight data and DB records, detects changes, and generates change summaries and notification lists.  
    - Configuration:  
      - Custom JS code reads API and DB data, checks for differences in departure/arrival times, status, gate.  
      - Change severity: low, medium, high, critical based on time difference or status (e.g., cancellations critical).  
      - Outputs JSON with total changes, detailed changes, and notifications for urgent cases.  
    - Inputs: Two inputs — API data and DB data (from "Get Current Schedules")  
    - Outputs: Passes processed results to "Check for Changes"  
    - Edge cases: Parsing errors, data format inconsistencies, missing fields.

  - **Check for Changes**  
    - Type: If  
    - Role: Determines if there are any detected changes requiring further processing.  
    - Configuration: Checks if total_changes > 0  
    - Inputs: Output from "Process Changes"  
    - Outputs:  
      - True branch: proceed to "Update Database" and "Check Urgent Notifications"  
      - False branch: loops back to "Get Current Schedules" (no changes)  
    - Edge cases: Expression failures, zero changes.

---

#### 2.3 Database and Team Notifications

- **Overview:**  
  When changes are detected, this block updates the flight schedules in the database, notifies the flight operations Slack channel with details, syncs updates to internal systems, and logs the synchronization event.

- **Nodes Involved:**  
  - Update Database  
  - Notify Slack Channel  
  - Update Internal Systems  
  - Log Sync Activity

- **Node Details:**

  - **Update Database**  
    - Type: Postgres  
    - Role: Updates the flight_schedules table with new times, status, gate, terminal, and timestamp.  
    - Configuration:  
      - Parameterized SQL update statement, executed independently per record.  
      - Inputs: Change data from "Check for Changes" true branch  
      - Outputs: Triggers "Notify Slack Channel"  
    - Edge cases: SQL errors, concurrent update conflicts.

  - **Notify Slack Channel**  
    - Type: HTTP Request  
    - Role: Posts a formatted message to a Slack channel summarizing the changes.  
    - Configuration:  
      - URL: Slack API chat.postMessage endpoint  
      - Authentication: Bearer token via HTTP Header Auth  
      - Body: JSON with channel `#flight-operations` and message containing total changes, flight numbers, airlines, routes, severity, and change types.  
    - Inputs: From "Update Database"  
    - Outputs: Triggers "Update Internal Systems"  
    - Edge cases: Slack API rate limits, invalid tokens, message formatting errors.

  - **Update Internal Systems**  
    - Type: HTTP Request  
    - Role: Sends webhook data to a Zapier endpoint for syncing changes with other airline systems.  
    - Configuration:  
      - URL: Zapier webhook URL  
      - Payload: event_type, total_changes, changes, sync timestamp, workflow ID  
    - Inputs: From "Notify Slack Channel"  
    - Outputs: Triggers "Log Sync Activity"  
    - Edge cases: Webhook failures, network issues.

  - **Log Sync Activity**  
    - Type: Postgres  
    - Role: Inserts a log entry recording the synchronization activity with timestamp and details.  
    - Configuration:  
      - SQL insert statement with workflow name, total changes, completion status, timestamp, and details.  
    - Inputs: From "Update Internal Systems"  
    - Outputs: End of this block  
    - Edge cases: DB insert failures.

---

#### 2.4 Passenger Notification Management

- **Overview:**  
  For urgent changes (high or critical), this block identifies passengers booked on affected flights, sends personalized email notifications via SendGrid, and sends SMS alerts for critical cases via Twilio.

- **Nodes Involved:**  
  - Check Urgent Notifications  
  - Get Affected Passengers  
  - Send Email Notifications  
  - Send SMS (Critical Only)

- **Node Details:**

  - **Check Urgent Notifications**  
    - Type: If  
    - Role: Checks if there are any notifications flagged as urgent (notifications array length > 0).  
    - Inputs: From "Check for Changes" true branch  
    - Outputs:  
      - True branch: proceeds to "Get Affected Passengers"  
      - False branch: loops back to "Get Current Schedules" (no urgent notifications)  
    - Edge cases: Expression errors, empty array.

  - **Get Affected Passengers**  
    - Type: Postgres  
    - Role: Queries passenger contact info (email, phone, name) for confirmed tickets on affected flights scheduled today or later.  
    - Configuration:  
      - SQL query uses flight numbers array as parameter to filter tickets and passengers.  
    - Inputs: From "Check Urgent Notifications"  
    - Outputs: Passes passenger data to "Send Email Notifications"  
    - Edge cases: Empty results, DB errors.

  - **Send Email Notifications**  
    - Type: HTTP Request  
    - Role: Sends personalized email notifications to passengers via SendGrid using a dynamic template.  
    - Configuration:  
      - URL: SendGrid mail send API  
      - Authentication: Bearer token via HTTP Header Auth  
      - Body: Personalizations with recipient email/name, dynamic template data including passenger name, flight number, and changes extracted from earlier processing.  
      - Batch sending enabled with batch size 10.  
      - From address: noreply@airline.com  
      - Template ID specified (dynamic template must exist in SendGrid).  
    - Inputs: Passenger data from "Get Affected Passengers"  
    - Outputs: Triggers "Send SMS (Critical Only)"  
    - Edge cases: API limits, invalid email addresses, template errors.

  - **Send SMS (Critical Only)**  
    - Type: HTTP Request  
    - Role: Sends SMS alerts via Twilio only to passengers affected by critical flight changes.  
    - Configuration:  
      - URL: Twilio API with Account SID dynamic from credentials  
      - Authentication: HTTP Basic Auth with Twilio credentials  
      - Body: Parameters include recipient phone, sender phone number, message body summarizing urgent flight change and instructions.  
      - Batch sending with batch size 5 and 2-second interval.  
    - Inputs: From "Send Email Notifications"  
    - Outputs: End of passenger notification flow  
    - Edge cases: SMS delivery failures, invalid phone numbers, API limits.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                   | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                                            |
|-------------------------|---------------------|-------------------------------------------------|--------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Triggers workflow every 30 minutes               | None                     | Fetch Airline Data             | How It Works: Step 1 - Periodic trigger for checking updates                                                          |
| Fetch Airline Data      | HTTP Request        | Fetches flight data from Aviation API            | Schedule Trigger         | Get Current Schedules          | How It Works: Step 2 - Retrieve flight information                                                                     |
| Get Current Schedules   | Postgres            | Retrieves recent flight schedules from DB        | Fetch Airline Data        | Process Changes                | How It Works: Step 3 - Pull existing schedules for comparison                                                          |
| Process Changes         | Code (JavaScript)   | Detects schedule changes and prepares notifications | Get Current Schedules     | Check for Changes              | How It Works: Step 4 - Compare API and DB data for updates                                                             |
| Check for Changes       | If                  | Checks if changes exist to proceed                 | Process Changes           | Update Database, Check Urgent Notifications, or Get Current Schedules (no changes) | How It Works: Step 5 - Determine if updates require action                                                             |
| Update Database         | Postgres            | Updates flight schedules with detected changes    | Check for Changes         | Notify Slack Channel           | How It Works: Step 6 - Save updated schedule data                                                                       |
| Notify Slack Channel    | HTTP Request        | Sends update summary to Slack channel              | Update Database           | Update Internal Systems        | How It Works: Step 7 - Notify flight operations team                                                                    |
| Update Internal Systems | HTTP Request        | Syncs changes with internal systems via webhook   | Notify Slack Channel      | Log Sync Activity              | How It Works: Step 12 - Sync with other airline systems                                                                |
| Log Sync Activity       | Postgres            | Logs the synchronization event                      | Update Internal Systems   | None                          | How It Works: Step 13 - Record audit trail                                                                              |
| Check Urgent Notifications | If               | Determines if urgent notifications exist          | Check for Changes         | Get Affected Passengers or Get Current Schedules (no urgent) | How It Works: Step 8 - Identify critical changes requiring passenger alerts                                             |
| Get Affected Passengers | Postgres            | Gets passenger contacts for affected flights       | Check Urgent Notifications | Send Email Notifications       | How It Works: Step 9 - Retrieve passenger info for notifications                                                      |
| Send Email Notifications | HTTP Request       | Sends personalized emails via SendGrid             | Get Affected Passengers   | Send SMS (Critical Only)       | How It Works: Step 10 - Dispatch email alerts to passengers                                                            |
| Send SMS (Critical Only) | HTTP Request       | Sends SMS alerts for critical changes via Twilio   | Send Email Notifications  | None                          | How It Works: Step 11 - Send urgent SMS alerts                                                                          |
| Sticky Note             | Sticky Note         | Documentation node                                  | None                     | None                          | See block descriptions                                                                                                  |
| Sticky Note1            | Sticky Note         | Introduction and high-level workflow explanation   | None                     | None                          | Introduction to workflow purpose and communication channels                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to run every 30 minutes.  
   - No credentials needed.  

2. **Create Fetch Airline Data Node**  
   - Type: HTTP Request  
   - Set URL to `https://api.aviationstack.com/v1/flights`  
   - Authentication: HTTP Query Auth with API key credential (set up with your Aviation Stack API key).  
   - Connect Schedule Trigger output to this node's input.

3. **Create Get Current Schedules Node**  
   - Type: Postgres  
   - Credentials: Set up Postgres connection to the airline's internal database.  
   - SQL Query:  
     ```sql
     SELECT flight_number, departure_time, arrival_time, status, gate, terminal 
     FROM flight_schedules 
     WHERE updated_at > NOW() - INTERVAL 1 HOUR
     ```  
   - Connect Fetch Airline Data output to this node's input.

4. **Create Process Changes Node**  
   - Type: Code  
   - Input: two inputs (API data from Fetch Airline Data and DB data from Get Current Schedules)  
   - Paste the provided JavaScript code that compares flights and generates change notifications.  
   - Connect Get Current Schedules output to this node.

5. **Create Check for Changes Node**  
   - Type: If  
   - Condition: Check if `total_changes` (from Process Changes JSON output) > 0  
   - Connect Process Changes output to this node.

6. **Create Update Database Node**  
   - Type: Postgres  
   - Credentials: Same Postgres connection as before.  
   - SQL Update Query (parameterized):  
     ```
     UPDATE flight_schedules SET 
       departure_time = $1,
       arrival_time = $2,
       status = $3,
       gate = $4,
       terminal = $5,
       updated_at = NOW()
     WHERE flight_number = $6
     ```  
   - Configure query batching for independent execution per record.  
   - Connect Check for Changes true branch to this node.

7. **Create Notify Slack Channel Node**  
   - Type: HTTP Request  
   - URL: Slack API `chat.postMessage` endpoint.  
   - Authentication: HTTP Header Auth with Slack Bot token credential.  
   - Body Parameters: JSON with channel `#flight-operations` and message formatted with total changes and details from Process Changes output.  
   - Connect Update Database output to this node.

8. **Create Update Internal Systems Node**  
   - Type: HTTP Request  
   - URL: Webhook URL for internal systems (e.g., Zapier webhook).  
   - Body Parameters: Include event_type, total_changes, changes array, timestamp, and workflow ID from Process Changes output.  
   - Connect Notify Slack Channel output to this node.

9. **Create Log Sync Activity Node**  
   - Type: Postgres  
   - Credentials: Same Postgres connection.  
   - SQL Insert Query:  
     ```
     INSERT INTO sync_logs (workflow_name, total_changes, sync_status, sync_timestamp, details) 
     VALUES ('airline-schedule-sync', $1, 'completed', NOW(), $2)
     ```  
   - Connect Update Internal Systems output to this node.

10. **Create Check Urgent Notifications Node**  
    - Type: If  
    - Condition: Check if `notifications.length` from Process Changes output > 0  
    - Connect Check for Changes true branch to this node.

11. **Create Get Affected Passengers Node**  
    - Type: Postgres  
    - Credentials: Same Postgres connection.  
    - SQL Query:  
      ```
      SELECT DISTINCT p.email, p.phone, p.name, t.flight_number 
      FROM passengers p 
      JOIN tickets t ON p.passenger_id = t.passenger_id 
      WHERE t.flight_number = ANY($1) 
      AND t.flight_date >= CURRENT_DATE 
      AND t.status = 'confirmed'
      ```  
    - Input Parameter: Array of affected flight numbers from notifications.  
    - Connect Check Urgent Notifications true branch to this node.

12. **Create Send Email Notifications Node**  
    - Type: HTTP Request  
    - URL: `https://api.sendgrid.com/v3/mail/send`  
    - Authentication: HTTP Header Auth with SendGrid API key credential.  
    - Body Parameters:  
      - Use SendGrid dynamic template with personalizations per passenger (email, name, flight number, changes).  
      - From email: noreply@airline.com, name: Airline Notifications  
      - Template ID: Use your configured SendGrid dynamic template ID for flight changes.  
    - Enable batching with batch size 10.  
    - Connect Get Affected Passengers output to this node.

13. **Create Send SMS (Critical Only) Node**  
    - Type: HTTP Request  
    - URL: `https://api.twilio.com/2010-04-01/Accounts/{{ AccountSID }}/Messages.json` (Account SID from credentials)  
    - Authentication: HTTP Basic Auth with Twilio account SID and auth token credentials.  
    - Body Parameters:  
      - To: passenger phone number  
      - From: your Twilio phone number  
      - Body: Urgent flight change message with flight number and change types  
    - Enable batching with batch size 5 and interval 2000 ms (2 seconds).  
    - Connect Send Email Notifications output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                   |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| **Introduction:** This workflow continuously monitors airline schedule changes, notifying internal teams and passengers.   | Sticky Note1 in workflow                                         |
| **How It Works:** Stepwise explanation from trigger to logging, with passenger alerts and multi-channel notifications.      | Sticky Note in workflow                                          |
| Slack API documentation for chat.postMessage: https://api.slack.com/methods/chat.postMessage                              | Slack node setup reference                                       |
| SendGrid Dynamic Templates: https://sendgrid.com/docs/ui/sending-email/how-to-send-an-email-with-dynamic-transactional-templates/ | For template ID and personalization setup                        |
| Twilio SMS API docs: https://www.twilio.com/docs/sms/send-messages                                                         | For SMS node configuration                                       |
| Aviation Stack API docs: https://aviationstack.com/documentation                                                          | For API data field references and authentication                 |

---

This documentation provides a complete, structured understanding of the Real-Time Flight Tracking and Alert workflow, enabling reproduction, troubleshooting, and extension by advanced users or automation agents.