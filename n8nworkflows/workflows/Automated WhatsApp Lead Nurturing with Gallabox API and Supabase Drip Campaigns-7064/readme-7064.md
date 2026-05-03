Automated WhatsApp Lead Nurturing with Gallabox API and Supabase Drip Campaigns

https://n8nworkflows.xyz/workflows/automated-whatsapp-lead-nurturing-with-gallabox-api-and-supabase-drip-campaigns-7064


# Automated WhatsApp Lead Nurturing with Gallabox API and Supabase Drip Campaigns

### 1. Workflow Overview

This workflow automates WhatsApp lead nurturing by integrating Gallabox’s WhatsApp messaging API with Supabase as a data source and logging system. It targets contact records stored in Supabase, evaluates their current disposition and message count, and sends appropriate WhatsApp message templates in a drip campaign sequence, updating the contact status and logging message delivery results.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Data Retrieval:** Periodic trigger to fetch all leads from Supabase.
- **1.2 Disposition Routing:** Branching logic based on lead disposition/stage to target specific drip campaign flows.
- **1.3 Count-Based Message Selection:** Further routing based on the message count sent to each lead, selecting the appropriate message template.
- **1.4 Batch Processing Loops:** Iteration over leads in batches to process messages individually.
- **1.5 WhatsApp Messaging via Gallabox API:** HTTP requests to send templated WhatsApp messages to leads.
- **1.6 Logging and Status Updates:** Recording message send results to Supabase logs and updating lead records with new counts and timestamps.
- **1.7 Conditional Flow Control:** Decision nodes to handle intervals, response status codes, and error handling for robust execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

- **Overview:**  
  This block triggers the workflow hourly, fetching all records from the `contacts_ampere` table in Supabase to process leads.

- **Nodes Involved:**  
  - Schedule Trigger3  
  - Get many rows1

- **Node Details:**

  - **Schedule Trigger3**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every hour.  
    - Configuration: Interval set to 1 hour.  
    - Connections: Output to `Get many rows1`.

  - **Get many rows1**  
    - Type: Supabase node (getAll operation)  
    - Role: Retrieves all contacts from the `contacts_ampere` table without filters.  
    - Configuration: Returns all rows, no filter applied.  
    - Credentials: Uses configured Supabase API credentials.  
    - Connections: Output to `Disposition Switch`.  
    - Potential Failures: API connectivity issues, auth failure, empty table response.

---

#### 2.2 Disposition Routing

- **Overview:**  
  This block routes leads based on their `Disposition` field to different message drip flows targeting specific lead stages.

- **Nodes Involved:**  
  - Disposition Switch  
  - Switch2  
  - Switch3  
  - Switch4  
  - Switch5

- **Node Details:**

  - **Disposition Switch**  
    - Type: Switch  
    - Role: Routes leads according to `Disposition` value (e.g., `=new_lead`, `test_ride`, `Booking`, `walk_in`, `Sale`).  
    - Configuration: Exact string matching for disposition stages.  
    - Input: From `Get many rows1`.  
    - Outputs: To `Count Switch` and five separate `Switch` nodes (Switch2-Switch5) for further processing.  
    - Note: Helps target the correct message drip sequence per lead stage.  
    - Failures: Empty or unexpected disposition values may cause leads to be unprocessed.

  - **Switch2, Switch3, Switch4, Switch5**  
    - Type: Switch  
    - Role: Each handles routing based on the numeric `Count` of messages sent (0, 1, 2, 3).  
    - Configuration: Each switch routes leads to different batch processing loops depending on the `Count`.  
    - Input: From `Disposition Switch`.  
    - Outputs: Various batch loops (see below).  
    - Failures: Incorrect or missing `Count` values may cause no routing.

---

#### 2.3 Count-Based Message Selection & Batch Processing Loops

- **Overview:**  
  Depending on the message count for each lead, the workflow processes leads in batches and sends the next message in the drip.

- **Nodes Involved:**  
  - Count Switch  
  - Loop Over Items1  
  - Loop Over Items2  
  - Loop Over Items3  
  - Loop Over Items4

- **Node Details:**

  - **Count Switch**  
    - Type: Switch  
    - Role: Routes leads into one of four batch loops based on `Count` (0 to 3).  
    - Input: From `Disposition Switch`.  
    - Outputs: Each to one of the four batch split nodes.  
    - Failures: Erroneous `Count` values may skip processing.

  - **Loop Over Items1 to Loop Over Items4**  
    - Type: SplitInBatches  
    - Role: Process leads in batches for each message count level, enabling scalable processing.  
    - Configuration: Default batch size (not explicitly set; uses default).  
    - Inputs: From `Count Switch`.  
    - Outputs:  
      - For Loop Over Items1: directly to `new_lead_0` node (WhatsApp API call).  
      - For Loop Over Items2: to `If Interval`.  
      - For Loop Over Items3: to `If Interval1`.  
      - For Loop Over Items4: to `If2`.  
    - Failures: Batch size issues, large datasets might cause timeouts.

---

#### 2.4 Interval and Conditional Checks

- **Overview:**  
  This block verifies if the lead is eligible to receive another message based on interval conditions before sending.

- **Nodes Involved:**  
  - If Interval  
  - If Interval1  
  - If2

- **Node Details:**

  - **If Interval**  
    - Type: If  
    - Role: Checks if the `interval` field in lead data is greater-than-or-equal to 0 (likely to permit message sending).  
    - Input: From `Loop Over Items2`.  
    - Output: To `new_lead_` (HTTP request node).

  - **If Interval1**  
    - Type: If  
    - Role: Checks if `interval` exactly equals 24 (hours) before sending the next message.  
    - Input: From `Loop Over Items3`.  
    - Output: To `new_lead_3`.

  - **If2**  
    - Type: If  
    - Role: Checks if `interval` is greater-than-or-equal to 24, controlling message sending at this stage.  
    - Input: From `Loop Over Items4`.  
    - Output: To `new_lead_2`.

  - Failures: Incorrect or missing `interval` data can prevent message sending or cause unexpected flows.

---

#### 2.5 WhatsApp Messaging via Gallabox API

- **Overview:**  
  This block sends templated WhatsApp messages through the Gallabox API based on the lead’s current drip stage.

- **Nodes Involved:**  
  - new_lead_  
  - new_lead_0  
  - new_lead_2  
  - new_lead_3

- **Node Details:**

  - **General for all `new_lead_*` nodes:**  
    - Type: HTTP Request  
    - Role: Post templated WhatsApp message requests to Gallabox API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://server.gallabox.com/devapi/messages/whatsapp`  
      - Body: JSON template including recipient name and phone, template name `testing_rahi_1`, and button quick reply with payload "Show me Brochure".  
      - Headers: Includes API key and secret for Gallabox authentication via custom HTTP header authentication.  
      - Response: Full HTTP response captured for status checking.  
    - Input: From conditional nodes like `If Interval`, `If Interval1`, `If2`, or directly from batch loops.  
    - Output: To respective `Create Logs*` node for logging responses.  
    - Failures: API errors, invalid auth, network issues, template errors.

  - **Differences:**  
    - `new_lead_0` handles leads with Count 0.  
    - `new_lead_` handles leads with Count 1.  
    - `new_lead_2` and `new_lead_3` handle Counts 2 and 3 respectively, with slightly different interval checks.

  - **Sticky Note:**  
    - "Http request id of gallabox api, that will help you to send whatsapp messages."

---

#### 2.6 Logging and Status Updates

- **Overview:**  
  This block logs message delivery status to Supabase and updates the contact record with the increased message count and timestamp of last message sent.

- **Nodes Involved:**  
  - Create Logs  
  - Create Logs1  
  - Create Logs2  
  - Create Logs3  
  - Update a row1  
  - Update a row2  
  - Update a row3  
  - Update a row4  
  - If StatusCode  
  - If StatusCode 202  
  - If StatusCode 203  
  - If StatusCode 204

- **Node Details:**

  - **Create Logs, Create Logs1, Create Logs2, Create Logs3**  
    - Type: Supabase (insert)  
    - Role: Insert a new log record into `logs_nurture_ampere` table with message ID, phone, disposition, message count, last sent timestamp, status code, status message, and name.  
    - Input: From respective `new_lead_*` nodes.  
    - Output: To respective `If StatusCode` nodes to evaluate response success.  
    - Error Handling: Continue workflow even if logging fails.

  - **If StatusCode, If StatusCode 202, If StatusCode 203, If StatusCode 204**  
    - Type: If  
    - Role: Check if the HTTP response status code equals "202" (Accepted) indicating successful message send.  
    - Input: From respective `Create Logs*` nodes.  
    - Output: To respective `Update a row*` node if status is 202; otherwise stops update.  
    - Failures: Unexpected status codes or missing status code fields.

  - **Update a row1, Update a row2, Update a row3, Update a row4**  
    - Type: Supabase (update)  
    - Role: Update the lead record in `contacts_ampere` to increment `Count` by 1 and set `last_message_sent` to the HTTP response header `date`.  
    - Input: From respective `If StatusCode*` nodes.  
    - Output: Back to batch loops for next item processing (loop continuation).  
    - Error Handling: Continue workflow even if update fails.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                             | Input Node(s)            | Output Node(s)                      | Sticky Note                                                  |
|--------------------|---------------------|---------------------------------------------|--------------------------|-----------------------------------|--------------------------------------------------------------|
| Schedule Trigger3   | Schedule Trigger    | Trigger workflow hourly                      | -                        | Get many rows1                    |                                                              |
| Get many rows1     | Supabase (getAll)    | Retrieve all leads                           | Schedule Trigger3         | Disposition Switch               |                                                              |
| Disposition Switch | Switch              | Route leads by disposition                    | Get many rows1            | Count Switch, Switch2-5           | "This Tells you about the disposition/stage that you want to target" |
| Count Switch       | Switch              | Route leads by message count                  | Disposition Switch        | Loop Over Items1-4                | "This switch decides which drip to send means which message drip in sequence"  |
| Switch2            | Switch              | Count-based routing (alternate flows)        | Disposition Switch        | Loop Over Items (various)         | "This switch decides which drip to send means which message drip in sequence"  |
| Switch3            | Switch              | Count-based routing (alternate flows)        | Disposition Switch        | Loop Over Items (various)         | "This switch decides which drip to send means which message drip in sequence"  |
| Switch4            | Switch              | Count-based routing (alternate flows)        | Disposition Switch        | Loop Over Items (various)         | "This switch decides which drip to send means which message drip in sequence"  |
| Switch5            | Switch              | Count-based routing (alternate flows)        | Disposition Switch        | Loop Over Items (various)         | "This switch decides which drip to send means which message drip in sequence"  |
| Loop Over Items1   | SplitInBatches       | Batch process leads with Count=0             | Count Switch              | new_lead_0, new_lead_             |                                                              |
| Loop Over Items2   | SplitInBatches       | Batch process leads with Count=1             | Count Switch              | If Interval                      |                                                              |
| Loop Over Items3   | SplitInBatches       | Batch process leads with Count=2             | Count Switch              | If Interval1                     |                                                              |
| Loop Over Items4   | SplitInBatches       | Batch process leads with Count=3             | Count Switch              | If2                             |                                                              |
| If Interval        | If                   | Check interval before sending message (Count=1) | Loop Over Items2          | new_lead_                       |                                                              |
| If Interval1       | If                   | Check interval before sending message (Count=2) | Loop Over Items3          | new_lead_3                      |                                                              |
| If2                | If                   | Check interval before sending message (Count=3) | Loop Over Items4          | new_lead_2                      |                                                              |
| new_lead_          | HTTP Request         | Send WhatsApp message for Count=1            | If Interval               | Create Logs1                    | "Http request id of gallabox api, that will help you to send whatsapp messages" |
| new_lead_0         | HTTP Request         | Send WhatsApp message for Count=0            | Loop Over Items1 (alternative) | Create Logs                    | "Http request id of gallabox api, that will help you to send whatsapp messages" |
| new_lead_2         | HTTP Request         | Send WhatsApp message for Count=3            | If2                       | Create Logs3                   | "Http request id of gallabox api, that will help you to send whatsapp messages" |
| new_lead_3         | HTTP Request         | Send WhatsApp message for Count=2            | If Interval1              | Create Logs2                   | "Http request id of gallabox api, that will help you to send whatsapp messages" |
| Create Logs        | Supabase (insert)    | Log message send result for Count=0           | new_lead_0                | If StatusCode                  |                                                              |
| Create Logs1       | Supabase (insert)    | Log message send result for Count=1           | new_lead_                 | If StatusCode 202              |                                                              |
| Create Logs2       | Supabase (insert)    | Log message send result for Count=2           | new_lead_3                | If StatusCode 203              |                                                              |
| Create Logs3       | Supabase (insert)    | Log message send result for Count=3           | new_lead_2                | If StatusCode 204              |                                                              |
| If StatusCode      | If                   | Check if status code = 202 for Count=0        | Create Logs               | Update a row1                 |                                                              |
| If StatusCode 202  | If                   | Check if status code = 202 for Count=1        | Create Logs1              | Update a row2                 |                                                              |
| If StatusCode 203  | If                   | Check if status code = 202 for Count=2        | Create Logs2              | Update a row3                 |                                                              |
| If StatusCode 204  | If                   | Check if status code = 202 for Count=3        | Create Logs3              | Update a row4                 |                                                              |
| Update a row1      | Supabase (update)    | Update Count and last_message_sent for Count=0| If StatusCode             | Loop Over Items1              |                                                              |
| Update a row2      | Supabase (update)    | Update Count and last_message_sent for Count=1| If StatusCode 202         | Loop Over Items2              |                                                              |
| Update a row3      | Supabase (update)    | Update Count and last_message_sent for Count=2| If StatusCode 203         | Loop Over Items3              |                                                              |
| Update a row4      | Supabase (update)    | Update Count and last_message_sent for Count=3| If StatusCode 204         | Loop Over Items4              |                                                              |
| Sticky Note        | Sticky Note          | Comment on Gallabox HTTP requests             | -                        | -                             | "Http request id of gallabox api, that will help you to send whatsapp messages" |
| Sticky Note1       | Sticky Note          | Comment on Count Switch purpose                | -                        | -                             | "This switch decides which drip to send means which message drip in sequence"  |
| Sticky Note2       | Sticky Note          | Comment on Disposition Switch purpose          | -                        | -                             | "This Tells you about the disposition/stage that you want to target"           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Schedule Trigger3`  
   - Type: Schedule Trigger  
   - Configure interval to trigger every 1 hour.

2. **Add Supabase node to fetch leads**  
   - Name: `Get many rows1`  
   - Type: Supabase  
   - Operation: `getAll`  
   - Table: `contacts_ampere`  
   - Return all rows: true  
   - Credentials: Connect your Supabase API credentials.

3. **Add a Switch node for disposition routing**  
   - Name: `Disposition Switch`  
   - Type: Switch  
   - Field to evaluate: `Disposition` (string)  
   - Add rules for values:  
     - `=new_lead`  
     - `test_ride`  
     - `Booking`  
     - `walk_in`  
     - `Sale`

4. **Add a Switch node to route by message count**  
   - Name: `Count Switch`  
   - Type: Switch  
   - Field: `Count` (number)  
   - Rules for values: 0, 1, 2, 3

5. **Add batch processing nodes for each Count value**  
   - Names: `Loop Over Items1`, `Loop Over Items2`, `Loop Over Items3`, `Loop Over Items4`  
   - Type: SplitInBatches  
   - Connect each output of `Count Switch` to corresponding batch node.

6. **Add conditional interval check nodes**  
   - For Count=1: `If Interval` (check `interval >= 0`)  
   - For Count=2: `If Interval1` (check `interval == 24`)  
   - For Count=3: `If2` (check `interval >= 24`)  
   - Connect batch node outputs accordingly.

7. **Add HTTP Request nodes to send WhatsApp messages via Gallabox API**  
   - Names: `new_lead_0`, `new_lead_`, `new_lead_2`, `new_lead_3`  
   - Method: POST  
   - URL: `https://server.gallabox.com/devapi/messages/whatsapp`  
   - Body: JSON template with recipient name, phone prefixed with "91", template `testing_rahi_1`, including button quick reply.  
   - Authentication: Use HTTP Custom Auth with API key and secret headers.  
   - Connect each conditional node and batch node to the respective HTTP Request node.

8. **Add Supabase insert nodes to log message results**  
   - Names: `Create Logs`, `Create Logs1`, `Create Logs2`, `Create Logs3`  
   - Operation: Insert  
   - Table: `logs_nurture_ampere`  
   - Fields: message_id, phone, disposition, mes_count (Count+1), last_sent, status_code, status_message, name  
   - Connect each HTTP Request node to corresponding Create Logs node.

9. **Add If nodes to check HTTP response status code == 202**  
   - Names: `If StatusCode`, `If StatusCode 202`, `If StatusCode 203`, `If StatusCode 204`  
   - Condition: `$json.status_code == "202"`  
   - Connect each Create Logs node output to corresponding If node.

10. **Add Supabase update nodes to update contact record**  
    - Names: `Update a row1`, `Update a row2`, `Update a row3`, `Update a row4`  
    - Operation: Update  
    - Table: `contacts_ampere`  
    - Filters: `phone == $json.phone`  
    - Fields to update:  
      - `Count` incremented by 1 (`Count = previous + 1`)  
      - `last_message_sent` set to HTTP response header `date`  
    - Connect each If node’s True output to the respective Update node.

11. **Connect Update nodes back to batch nodes for loop continuation**  
    - Connect each Update node’s output to its corresponding batch node’s second input (for loop continuation).

12. **Connect initial Schedule Trigger3 to Get many rows1**  
    - Connect Get many rows1 to Disposition Switch.

13. **Connect Disposition Switch outputs to Count Switch and other Switch nodes**  
    - Connect to Count Switch for main drip routing.

14. **Add Sticky Notes as needed for documentation**  
    - E.g. Gallabox API HTTP request explanation near HTTP nodes, explanations near switches.

15. **Configure all credentials (Supabase API, Gallabox API keys) properly**  
    - Ensure the API keys are stored securely in n8n credentials and referenced in HTTP Request and Supabase nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                              |
|----------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| "Http request id of gallabox api, that will help you to send whatsapp messages"              | Sticky note near Gallabox HTTP Request nodes                 |
| "This switch decides which drip to send means which message drip in sequence"                | Sticky note near Count Switch and related Switch nodes       |
| "This Tells you about the disposition/stage that you want to target"                         | Sticky note near Disposition Switch node                      |
| Gallabox API Documentation for WhatsApp messaging integration recommended for template setup| https://www.gallabox.com/api-docs/whatsapp                    |
| Supabase API documentation for data operations                                              | https://supabase.com/docs/reference/javascript/supabase-client |
| Use of batching improves performance and avoids API rate limits                             | General workflow design note                                  |
| Proper error handling configured to continue workflow on logging or update failures          | Ensures resiliency but monitor for silent failures            |

---

This detailed analysis and reproduction guide enables both advanced users and automation agents to understand, modify, and deploy the Automated WhatsApp Lead Nurturing workflow effectively.