Automatic Email Unsubscribe Handler: Outlook to BigQuery Integration

https://n8nworkflows.xyz/workflows/automatic-email-unsubscribe-handler--outlook-to-bigquery-integration-6333


# Automatic Email Unsubscribe Handler: Outlook to BigQuery Integration

### 1. Workflow Overview

This workflow is designed to automate the process of handling email unsubscribe requests received via Outlook and syncing this data with Google BigQuery. It targets marketing or sales teams who want to maintain clean email lists by automatically detecting unsubscribe messages, logging unsubscribe emails into a BigQuery table, and removing these emails from their leads table.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Date Range Setup:** Runs every 4 hours and calculates the current date and 7 days prior for filtering emails.
- **1.2 Email Retrieval from Outlook:** Fetches all emails from the last 7 days that arrived in a specified Outlook folder.
- **1.3 Filtering Unsubscribe Messages:** Filters emails containing the word "unsubscribe" (case-insensitive).
- **1.4 Unsubscribe Email Extraction and Deduplication:** Extracts sender emails from unsubscribe messages, compares them with existing unsubscribe records in BigQuery to keep only new unsubscribes, and aggregates them by email.
- **1.5 BigQuery Integration:** Inserts new unsubscribe emails into the unsubscribes table and ensures these emails are removed from the leads table in BigQuery.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Date Range Setup

- **Overview:** This block triggers the workflow every 4 hours and calculates the current timestamp and timestamp from 5 days ago to limit the email search window.
- **Nodes Involved:**  
  - Run Ever 4 Hours  
  - Today & 7 Days Ago (Code node)  
- **Node Details:**

  - **Run Ever 4 Hours**  
    - Type: Schedule Trigger  
    - Role: Periodic trigger to start the workflow every 4 hours.  
    - Configuration: Interval set to 4 hours.  
    - Inputs: None (trigger node)  
    - Outputs: Passes control to Today & 7 Days Ago node.  
    - Edge cases: Scheduler misconfiguration or time zone issues could cause missed or overlapping runs.

  - **Today & 7 Days Ago**  
    - Type: Code (JavaScript)  
    - Role: Produces two formatted ISO8601 timestamps: current time and 5 days before current time (note: description says 7 days, but code uses 5 days).  
    - Configuration: Uses Date object and timezone offset calculation to generate timestamps with timezone offsets.  
    - Key expressions: `formatDate` function formats dates to `YYYY-MM-DDTHH:mm:ss.0000000±HH:MM` format.  
    - Inputs: Trigger from Run Ever 4 Hours.  
    - Outputs: JSON object with `now` and `sevenDaysAgo` fields used for filtering emails.  
    - Edge cases: Timezone offsets and daylight savings may affect date accuracy.

#### 1.2 Email Retrieval from Outlook

- **Overview:** Fetches all Outlook emails received between the calculated date range from a specific folder (Inbox).  
- **Nodes Involved:**  
  - Get Emails in Past 7 Days  
- **Node Details:**

  - **Get Emails in Past 7 Days**  
    - Type: Microsoft Outlook node  
    - Role: Retrieves emails with specified fields (body, createdDateTime, from, sender) received in the last 5 days.  
    - Configuration:  
      - Folder: Inbox (specific folder ID used)  
      - Filters: `receivedAfter` = sevenDaysAgo, `receivedBefore` = now (dynamic expressions)  
      - Return all emails matching criteria.  
      - Credentials: Microsoft Outlook OAuth2.  
    - Inputs: Output of Today & 7 Days Ago node.  
    - Outputs: List of emails filtered by date.  
    - Edge cases: Credential expiry, API rate limits, folder ID changes, large mailbox volume causing timeouts.

#### 1.3 Filtering Unsubscribe Messages

- **Overview:** Filters retrieved emails to keep only those whose body contains the word "unsubscribe" (case insensitive).  
- **Nodes Involved:**  
  - Filter for Unsubscribes  
- **Node Details:**

  - **Filter for Unsubscribes**  
    - Type: Filter node  
    - Role: Applies string containment check on email body content for "unsubscribe".  
    - Configuration:  
      - Condition: `body.content` contains "unsubscribe"  
      - Case insensitive enabled.  
    - Inputs: Emails output from Get Emails in Past 7 Days.  
    - Outputs: Emails matching unsubscribe condition.  
    - Edge cases: False positives if "unsubscribe" appears in unrelated contexts; missing unsubscribe requests due to different wording.

#### 1.4 Unsubscribe Email Extraction and Deduplication

- **Overview:** Extracts sender email addresses from unsubscribe emails, queries BigQuery for existing unsubscribes, and filters to keep only new unsubscribes before aggregation.  
- **Nodes Involved:**  
  - Edit Fields  
  - Query Bigquery for all Unsubscribes  
  - Edit Fields1  
  - Keep Only New Unsubs (Merge node)  
  - Aggregate to email level  
  - Loop Over Items1  
- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Role: Extracts sender email address from filtered unsubscribe messages and assigns it to field `email`.  
    - Configuration: Sets `email` to `sender.emailAddress.address`.  
    - Inputs: From Filter for Unsubscribes.  
    - Outputs: JSON with extracted `email`.  
    - Edge cases: Missing or malformed sender email could cause blank fields.

  - **Query Bigquery for all Unsubscribes**  
    - Type: Google BigQuery node  
    - Role: Retrieves all existing unsubscribe emails from BigQuery table `Unsubscribes`.  
    - Configuration:  
      - SQL: `SELECT * FROM n8nautomation-453001.email_leads_schema.Unsubscribes`  
      - Credentials: Google BigQuery OAuth2.  
    - Inputs: From Today & 7 Days Ago node (parallel branch).  
    - Outputs: List of already unsubscribed emails.  
    - Edge cases: Credential errors, network issues, query syntax errors.

  - **Edit Fields1**  
    - Type: Set node  
    - Role: Prepares BigQuery unsubscribe emails for comparison by mapping `email` field to `queryemail`.  
    - Configuration: Sets `queryemail` = `$json.email`.  
    - Inputs: Output of Query Bigquery for all Unsubscribes.  
    - Outputs: JSON with `queryemail`.  
    - Edge cases: Empty or null emails in BigQuery data.

  - **Keep Only New Unsubs**  
    - Type: Merge node  
    - Role: Performs anti-join (keepNonMatches) to filter unsubscribe emails that do not exist in BigQuery unsubscribes.  
    - Configuration:  
      - Join by `email` (from unsubscribes extracted) and `queryemail` (from BigQuery).  
      - Output data from input1 (unsubscribe emails).  
    - Inputs:  
      - Input1: Output from Edit Fields (new unsubscribe emails).  
      - Input2: Output from Edit Fields1 (existing unsubscribes).  
    - Outputs: New unsubscribe emails not already in BigQuery.  
    - Edge cases: Email case sensitivity mismatch, missing fields.

  - **Aggregate to email level**  
    - Type: Summarize node  
    - Role: Groups new unsubscribes by unique email to remove duplicates.  
    - Configuration: Split by `email` field, summarize field "1" (count).  
    - Inputs: Output from Keep Only New Unsubs.  
    - Outputs: Aggregated unique unsubscribe emails.  
    - Edge cases: Emails with different cases or whitespace may be treated as distinct.

  - **Loop Over Items1**  
    - Type: SplitInBatches node  
    - Role: Processes each unique unsubscribe email one at a time for insertion.  
    - Configuration: Batch size = 1.  
    - Inputs: Output from Aggregate to email level.  
    - Outputs: Single unsubscribe email per iteration.  
    - Edge cases: Large batch sizes may cause timeouts or throttling.

#### 1.5 BigQuery Integration

- **Overview:** Inserts each new unsubscribe email into the BigQuery unsubscribes table and (not shown explicitly in nodes) is intended to delete these emails from the leads table.  
- **Nodes Involved:**  
  - Add unsubscribes to Table  
- **Node Details:**

  - **Add unsubscribes to Table**  
    - Type: Google BigQuery node  
    - Role: Executes a MERGE SQL query to insert email into `Unsubscribes` table if not already present.  
    - Configuration:  
      - SQL query uses MERGE statement with parameterized email (`{{ $json.email }}`)  
      - Credentials: Google BigQuery OAuth2.  
    - Inputs: From Loop Over Items1 (one email at a time).  
    - Outputs: Confirmation of insertion.  
    - Edge cases: SQL syntax errors, credential issues, concurrency conflicts if duplicate inserts occur simultaneously.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                           | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                                               |
|----------------------------|------------------------|-----------------------------------------|-------------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Run Ever 4 Hours            | Schedule Trigger       | Triggers workflow every 4 hours         | None                                | Today & 7 Days Ago                 |                                                                                                                            |
| Today & 7 Days Ago          | Code                   | Calculates current and past timestamps  | Run Ever 4 Hours                    | Get Emails in Past 7 Days, Query Bigquery for all Unsubscribes |                                                                                                                            |
| Get Emails in Past 7 Days   | Microsoft Outlook      | Retrieves emails from last 5 days       | Today & 7 Days Ago                  | Filter for Unsubscribes            |                                                                                                                            |
| Filter for Unsubscribes     | Filter                 | Filters emails containing "unsubscribe"| Get Emails in Past 7 Days           | Edit Fields                      |                                                                                                                            |
| Edit Fields                | Set                    | Extracts sender email from unsubscribes | Filter for Unsubscribes             | Keep Only New Unsubs (input1)     |                                                                                                                            |
| Query Bigquery for all Unsubscribes | Google BigQuery    | Gets existing unsubscribes from BigQuery | Today & 7 Days Ago                  | Edit Fields1                     |                                                                                                                            |
| Edit Fields1               | Set                    | Maps BigQuery emails for comparison     | Query Bigquery for all Unsubscribes| Keep Only New Unsubs (input2)     |                                                                                                                            |
| Keep Only New Unsubs        | Merge                  | Filters new unsubscribes only            | Edit Fields, Edit Fields1           | Aggregate to email level          |                                                                                                                            |
| Aggregate to email level    | Summarize              | Deduplicates unsubscribe emails          | Keep Only New Unsubs                | Loop Over Items1                  |                                                                                                                            |
| Loop Over Items1            | SplitInBatches         | Processes each unsubscribe email individually | Aggregate to email level          | Summarize, Add unsubscribes to Table |                                                                                                                            |
| Add unsubscribes to Table   | Google BigQuery        | Inserts new unsubscribes into BigQuery   | Loop Over Items1                   | Loop Over Items1                  |                                                                                                                            |
| Sticky Note8                | Sticky Note            | Empty note for layout                    | None                              | None                            |                                                                                                                            |
| Sticky Note9                | Sticky Note            | Contact info for support                  | None                              | None                            | ## Email Unsubscribe Handler for Outlook \n\n** Feel free to contact me if you need help implementing (rbreen@ynteractive.com) **  |
| Sticky Note10               | Sticky Note            | Implementation instructions and setup    | None                              | None                            | ## How to Implement This n8n Workflow\n\nDetailed steps on Outlook and BigQuery setup, scheduling, and expected workflow behavior |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Name: "Run Ever 4 Hours"  
   - Type: Schedule Trigger  
   - Configure: Set interval to every 4 hours.

2. **Create a Code Node to Calculate Date Range:**  
   - Name: "Today & 7 Days Ago"  
   - Type: Code (JavaScript)  
   - Code: Use JavaScript to create ISO8601 timestamps for current time and 5 days ago, including timezone offset (see provided code logic).  
   - Connect output of "Run Ever 4 Hours" to this node.

3. **Create Microsoft Outlook Node to Fetch Emails:**  
   - Name: "Get Emails in Past 7 Days"  
   - Type: Microsoft Outlook  
   - Credentials: Set up Microsoft Outlook OAuth2 with appropriate account.  
   - Configure:  
     - Folder: Inbox (use the provided folder ID or your own)  
     - Fields: body, createdDateTime, from, sender  
     - Filter: receivedAfter = `{{$json.sevenDaysAgo}}`, receivedBefore = `{{$json.now}}`  
     - Return all matching emails.  
   - Connect output of "Today & 7 Days Ago" to this node.

4. **Create Filter Node to Detect Unsubscribe Emails:**  
   - Name: "Filter for Unsubscribes"  
   - Type: Filter  
   - Condition: Check if `body.content` contains "unsubscribe" (case-insensitive).  
   - Connect output of "Get Emails in Past 7 Days" to this node.

5. **Create Set Node to Extract Sender Email:**  
   - Name: "Edit Fields"  
   - Type: Set  
   - Assignment: Set field `email` to `{{$json.sender.emailAddress.address}}`.  
   - Connect output of "Filter for Unsubscribes" to this node.

6. **Create Google BigQuery Node to Query Existing Unsubscribes:**  
   - Name: "Query Bigquery for all Unsubscribes"  
   - Type: Google BigQuery  
   - Credentials: Set up Google BigQuery OAuth2 credentials.  
   - SQL: `SELECT * FROM \`[project_id].[dataset].Unsubscribes\`` (replace with your project and dataset)  
   - Connect output of "Today & 7 Days Ago" to this node.

7. **Create Set Node to Prepare BigQuery Emails:**  
   - Name: "Edit Fields1"  
   - Type: Set  
   - Assignment: Set field `queryemail` = `{{$json.email}}`.  
   - Connect output of "Query Bigquery for all Unsubscribes" to this node.

8. **Create Merge Node to Filter Only New Unsubscribes:**  
   - Name: "Keep Only New Unsubs"  
   - Type: Merge  
   - Mode: Combine  
   - Join Mode: keepNonMatches (anti-join)  
   - Merge By Fields: `email` (from Edit Fields) and `queryemail` (from Edit Fields1)  
   - Connect "Edit Fields" to Input1, "Edit Fields1" to Input2.

9. **Create Summarize Node to Deduplicate Emails:**  
   - Name: "Aggregate to email level"  
   - Type: Summarize  
   - Split by field: `email`  
   - Summarize: Count occurrences (field "1")  
   - Connect output of "Keep Only New Unsubs" to this node.

10. **Create SplitInBatches Node to Process Emails One by One:**  
    - Name: "Loop Over Items1"  
    - Type: SplitInBatches  
    - Batch Size: 1  
    - Connect output of "Aggregate to email level" to this node.

11. **Create Google BigQuery Node to Insert New Unsubscribes:**  
    - Name: "Add unsubscribes to Table"  
    - Type: Google BigQuery  
    - Credentials: Use Google BigQuery OAuth2 credentials.  
    - SQL Query:  
      ```
      MERGE INTO `[project_id].[dataset].Unsubscribes` AS target
      USING (
        SELECT '{{ $json.email }}' AS email
      ) AS source
      ON target.email = source.email
      WHEN NOT MATCHED THEN
        INSERT(email) VALUES(source.email);
      ```  
    - Connect output of "Loop Over Items1" to this node.

12. **(Optional) Extend Workflow:**  
    - Add logic to delete unsubscribed emails from the leads table in BigQuery as needed.

13. **Set Credentials:**  
    - Microsoft Outlook OAuth2 for Outlook node.  
    - Google BigQuery OAuth2 for BigQuery nodes.

14. **Optional:** Add Sticky Notes for documentation and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Email Unsubscribe Handler for Outlook. Feel free to contact me if you need help implementing (rbreen@ynteractive.com)                                                                                                                                                                                                                                                                                                                             | Contact info in Sticky Note9                                                                                     |
| How to Implement This n8n Workflow: Steps include Outlook account connection, BigQuery tables setup (unsubscribes and leads), schedule setup (every 4 hours), and workflow behavior overview. Includes links to Google Cloud Console and instructions for credentials setup.                                                                                                                                                                         | Sticky Note10 content includes detailed instructions and links: https://console.cloud.google.com/                 |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.