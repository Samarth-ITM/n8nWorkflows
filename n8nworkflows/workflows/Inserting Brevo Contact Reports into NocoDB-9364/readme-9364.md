Inserting Brevo Contact Reports into NocoDB

https://n8nworkflows.xyz/workflows/inserting-brevo-contact-reports-into-nocodb-9364


# Inserting Brevo Contact Reports into NocoDB

### 1. Workflow Overview

This workflow automates the process of retrieving contact engagement reports from the Brevo API and inserting or updating this data into a NocoDB table. It is designed for users who maintain email contact records in NocoDB and want to regularly synchronize engagement metrics such as messages sent, delivered, opened, and clicked events from Brevo.

The workflow’s logic is structured into the following main blocks:

- **1.1 Input Reception and Data Retrieval:**  
  Fetches all email records from NocoDB and iterates over them to get detailed Brevo contact reports.

- **1.2 Data Filtering and Branching:**  
  Checks if contacts are blacklisted and branches accordingly to either update blacklist status or process engagement data.

- **1.3 Engagement Data Processing:**  
  Splits the engagement statistics (messages sent, delivered, opened, clicked) into separate data streams for individual processing.

- **1.4 Data Grouping and Formatting:**  
  Uses JavaScript code nodes to group multiple campaign records per email and prepare aggregated strings for each engagement type.

- **1.5 Data Merging and Cleanup:**  
  Merges the processed data streams back into unified records, cleans nulls and duplicates, and prepares final payloads.

- **1.6 Database Updating:**  
  Updates the corresponding NocoDB records with the aggregated engagement data and marks them as processed.

- **1.7 Blacklist Updating:**  
  Separately updates records marked as blacklisted in NocoDB.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Data Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule, retrieves all email records from NocoDB, filters out already processed entries, and loops through the remaining emails to pull detailed contact reports from Brevo.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Insert Emails from NocoDB  
  - Check if Email is Inserted Before  
  - Loop  
  - Get Brevo Contact Report

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger  
    - Role: Starts workflow every 2 seconds.  
    - Config: Interval set to 2 seconds.  
    - Input/Output: No input; outputs trigger event.  
    - Edge Cases: Frequent triggering could cause API rate limits.

  - **Insert Emails from NocoDB**  
    - Type: NocoDB (Get All)  
    - Role: Fetches all contact records from a NocoDB table including fields like email, messagesSent, delivered, opened, clicked, done, blacklisted.  
    - Credential: NocoDB API token.  
    - Input/Output: Receives trigger; outputs array of records.  
    - Edge Cases: API failure, authentication errors.

  - **Check if Email is Inserted Before**  
    - Type: Filter  
    - Role: Filters out records already marked as done (done ≠ 1).  
    - Logic: Checks field “done” not equal to 1 in strict mode.  
    - Input/Output: Input from NocoDB; outputs filtered records.  
    - Edge Cases: Missing or malformed “done” field.

  - **Loop**  
    - Type: SplitInBatches  
    - Role: Processes emails in batches of 10 for rate limiting and efficiency.  
    - Config: Batch size 10.  
    - Input/Output: Receives filtered emails; outputs one batch at a time.  
    - Edge Cases: Large datasets may slow processing.

  - **Get Brevo Contact Report**  
    - Type: HTTP Request  
    - Role: Calls Brevo API for detailed contact report per email.  
    - Config: URL uses expression `https://api.brevo.com/v3/contacts/{{ $json.email }}`; HTTP header authentication with Brevo API token; Accept header set to application/json.  
    - Input/Output: Receives batch items; outputs detailed contact JSON data.  
    - Credential: Brevo API token (HTTP header auth).  
    - Edge Cases: API rate limits, network errors, invalid emails, expired tokens.

---

#### 2.2 Data Filtering and Branching

- **Overview:**  
  After retrieving contact data, this block checks if the email is blacklisted. If so, it updates the blacklist status in NocoDB; otherwise, it proceeds to process engagement data.

- **Nodes Involved:**  
  - If Email Blacklisted  
  - Update NocoDB - blacklisted  
  - Split Out - messagesSent  
  - Split Out - delivered  
  - Split Out - opened  
  - Split Out - clicked

- **Node Details:**  

  - **If Email Blacklisted**  
    - Type: If  
    - Role: Branches workflow based on boolean field `emailBlacklisted` from Brevo response.  
    - Logic: True branch if emailBlacklisted is true; false branch otherwise.  
    - Input/Output: Input from Brevo API node; outputs two branches.  
    - Edge Cases: Missing field or unexpected data type.

  - **Update NocoDB - blacklisted**  
    - Type: NocoDB (Update)  
    - Role: Marks the contact as blacklisted and done in NocoDB.  
    - Config: Updates table row id from Loop node, sets “blacklisted”=1 and “done”=1.  
    - Credential: NocoDB API token.  
    - Input/Output: Input from If node true branch; outputs update confirmation.  
    - Edge Cases: Record not found, update failure.

  - **Split Out - messagesSent / delivered / opened / clicked**  
    - Type: SplitOut  
    - Role: Each splits the corresponding engagement statistics array from Brevo’s contact report.  
    - Config: Splits field `statistics.messagesSent`, `statistics.delivered`, `statistics.opened`, `statistics.clicked` respectively; includes “email” field in output.  
    - Input/Output: Input from If node false branch; outputs split items for each engagement type.  
    - Edge Cases: Empty arrays, missing fields, large data arrays.

---

#### 2.3 Engagement Data Processing

- **Overview:**  
  For each engagement type, this block sets key fields and groups multiple campaign entries by email to create aggregated strings for insertion.

- **Nodes Involved:**  
  - Edit Fields (for messagesSent, delivered, opened, clicked)  
  - Code (for messagesSent, delivered, opened, clicked)

- **Node Details:**  

  - **Edit Fields Nodes (Edit Fields, Edit Fields2, Edit Fields3, Edit Fields4)**  
    - Type: Set  
    - Role: Assigns fields “email”, “campaignId” (from the respective statistics entry), and “NocoDB_id” (fetched from “Insert Emails from NocoDB” node).  
    - Config: Uses expressions to map input JSON fields and reference external node data for “NocoDB_id”.  
    - Input/Output: Input from Split Out nodes; output to Code nodes.  
    - Edge Cases: Missing campaignId, missing NocoDB_id reference.

  - **Code Nodes (Code, Code1, Code2, Code3)**  
    - Type: Code (JavaScript)  
    - Role: Groups entries by email, concatenates campaignIds into comma-separated strings for each engagement type.  
    - Logic: Iterates all items, groups by email, collects campaignId arrays, and returns aggregated string.  
    - Input/Output: Input from respective Edit Fields nodes; output to Merge node.  
    - Edge Cases: Empty inputs, malformed data.

---

#### 2.4 Data Merging and Cleanup

- **Overview:**  
  Merges all four engagement data streams into unified records, removes null or missing fields, and consolidates data for database update.

- **Nodes Involved:**  
  - Merge  
  - Delete Nulls

- **Node Details:**  

  - **Merge**  
    - Type: Merge  
    - Role: Combines four input streams (messagesSent, delivered, opened, clicked) by matching indexes.  
    - Config: NumberInputs=4; merges inputs from the four Code nodes.  
    - Input/Output: Inputs from Code nodes; outputs merged records.  
    - Edge Cases: Unequal input lengths causing misaligned merges.

  - **Delete Nulls**  
    - Type: Code (JavaScript)  
    - Role: Cleans merged records by grouping again by email, concatenating all engagement fields, and removing null or empty values.  
    - Logic: Ensures no duplicate or empty strings are inserted.  
    - Input/Output: Input from Merge; outputs cleaned records.  
    - Edge Cases: Null or undefined fields, duplicates.

---

#### 2.5 Database Updating

- **Overview:**  
  Updates the NocoDB table with the aggregated engagement data and marks each record as done; then loops back to process the next batch.

- **Nodes Involved:**  
  - Update NocoDB  
  - Loop

- **Node Details:**  

  - **Update NocoDB**  
    - Type: NocoDB (Update)  
    - Role: Updates the contact record with the new aggregated engagement metrics and sets “done”=1.  
    - Config: Matches record by “id” from “NocoDB_id” field; updates fields messagesSent, delivered, opened, clicked, done.  
    - Credential: NocoDB API token.  
    - Input/Output: Input from Delete Nulls node; outputs update confirmation.  
    - Edge Cases: Record locking, update failure.

  - **Loop**  
    - Type: SplitInBatches  
    - Role: Triggers next batch processing after update completion, cycles back to Get Brevo Contact Report.  
    - Config: Batch size 10 (same as above).  
    - Input/Output: Input from Update NocoDB; outputs batches to Get Brevo Contact Report.  
    - Edge Cases: Infinite loops if not handled carefully.

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                                 | Input Node(s)                             | Output Node(s)                        | Sticky Note                                                                                       |
|------------------------------|----------------------|------------------------------------------------|------------------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger     | Starts workflow on schedule (every 2 seconds) | None                                     | Insert Emails from NocoDB            |                                                                                                 |
| Insert Emails from NocoDB     | NocoDB               | Fetch all email records from NocoDB table      | Schedule Trigger                         | Check if Email is Inserted Before    | This node retrieves email data from your **NocoDB** table including key fields (email, stats).  |
| Check if Email is Inserted Before | Filter           | Filters out emails already processed (done=1) | Insert Emails from NocoDB                | Loop                                |                                                                                                 |
| Loop                         | SplitInBatches       | Processes emails in batches of 10               | Check if Email is Inserted Before       | Get Brevo Contact Report             |                                                                                                 |
| Get Brevo Contact Report      | HTTP Request         | Calls Brevo API to get detailed contact report | Loop                                    | If Email Blacklisted                 | Connects to **Brevo API** to retrieve engagement data (messagesSent, delivered, opened, clicked)|
| If Email Blacklisted          | If                   | Branches logic based on blacklist status        | Get Brevo Contact Report                 | Update NocoDB - blacklisted, Split Out nodes |                                                                                     |
| Update NocoDB - blacklisted  | NocoDB               | Marks contact as blacklisted in database        | If Email Blacklisted (true branch)      | None                               |                                                                                                 |
| Split Out - messagesSent      | SplitOut             | Splits messagesSent stats into separate items   | If Email Blacklisted (false branch)     | Edit Fields                        | Splits Brevo contact report data into separate branches for processing                            |
| Split Out - delivered         | SplitOut             | Splits delivered stats                           | If Email Blacklisted (false branch)     | Edit Fields2                       |                                                                                                 |
| Split Out - opened            | SplitOut             | Splits opened stats                              | If Email Blacklisted (false branch)     | Edit Fields3                       |                                                                                                 |
| Split Out - clicked           | SplitOut             | Splits clicked stats                             | If Email Blacklisted (false branch)     | Edit Fields4                       |                                                                                                 |
| Edit Fields                  | Set                  | Sets fields for messagesSent data                | Split Out - messagesSent                 | Code                             |                                                                                                 |
| Edit Fields2                 | Set                  | Sets fields for delivered data                    | Split Out - delivered                    | Code1                            |                                                                                                 |
| Edit Fields3                 | Set                  | Sets fields for opened data                       | Split Out - opened                       | Code2                            |                                                                                                 |
| Edit Fields4                 | Set                  | Sets fields for clicked data                      | Split Out - clicked                      | Code3                            |                                                                                                 |
| Code                        | Code (JavaScript)    | Groups messagesSent by email, aggregates campaignIds | Edit Fields                             | Merge                            | Uses JS to group and organize Brevo data by email for messagesSent                              |
| Code1                       | Code (JavaScript)    | Groups delivered by email                         | Edit Fields2                            | Merge                            | Groups and organizes Brevo data by email for delivered                                         |
| Code2                       | Code (JavaScript)    | Groups opened by email                            | Edit Fields3                            | Merge                            | Groups and organizes Brevo data by email for opened                                            |
| Code3                       | Code (JavaScript)    | Groups clicked by email                           | Edit Fields4                            | Merge                            | Groups and organizes Brevo data by email for clicked                                           |
| Merge                       | Merge                | Merges all engagement data streams                | Code, Code1, Code2, Code3               | Delete Nulls                     |                                                                                                 |
| Delete Nulls                | Code (JavaScript)    | Cleans merged data, removes nulls, duplicates     | Merge                                  | Update NocoDB                    |                                                                                                 |
| Update NocoDB               | NocoDB               | Updates contact records with aggregated data       | Delete Nulls                           | Loop                            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 2 seconds.

2. **Create a NocoDB node to fetch emails**  
   - Type: NocoDB (Get All)  
   - Configure with your NocoDB API token credential.  
   - Set table to your contacts table (e.g., "mz06ar1jy2ca287").  
   - Return all records.  
   - This node fetches fields: email, messagesSent, delivered, opened, clicked, done, blacklisted.

3. **Add a Filter node (Check if Email is Inserted Before)**  
   - Use condition: `done` field not equal to `1` (strict).  
   - This filters out already processed contacts.

4. **Add a SplitInBatches node (Loop)**  
   - Batch size: 10.  
   - This batches contacts for rate limiting.

5. **Add an HTTP Request node (Get Brevo Contact Report)**  
   - URL: `https://api.brevo.com/v3/contacts/{{ $json.email }}` (use expression).  
   - Authentication: HTTP Header Auth with Brevo API token.  
   - Headers: Accept = application/json.

6. **Add an If node (If Email Blacklisted)**  
   - Condition: Check if `emailBlacklisted` field is true (boolean).  
   - True branch: email is blacklisted.  
   - False branch: email is not blacklisted.

7. **On True branch:** Add a NocoDB Update node (Update NocoDB - blacklisted)  
   - Update the record using `id` from current Loop item.  
   - Set fields: blacklisted = 1, done = 1.

8. **On False branch:** Add four SplitOut nodes for each engagement type:  
   - Split Out - messagesSent: split `statistics.messagesSent`  
   - Split Out - delivered: split `statistics.delivered`  
   - Split Out - opened: split `statistics.opened`  
   - Split Out - clicked: split `statistics.clicked`  
   - Include the email field in each.

9. **For each SplitOut node, add a Set node (Edit Fields, Edit Fields2, Edit Fields3, Edit Fields4):**  
   - Set fields:  
     - email = `{{$json.email}}`  
     - campaignId = `{{$json["statistics.<type>"].campaignId}}` (replace `<type>` accordingly)  
     - NocoDB_id = `{{$('Insert Emails from NocoDB').item.json.Id}}` (reference from earlier fetch)  

10. **For each Set node, add a Code node (Code, Code1, Code2, Code3):**  
    - JavaScript code to group items by email and aggregate campaignIds into a comma-separated string.  
    - Use the provided code logic for grouping and returning aggregated results.

11. **Add a Merge node:**  
    - Number of inputs: 4 (one from each Code node).  
    - This merges all engagement data streams into single records.

12. **Add a Code node (Delete Nulls):**  
    - JavaScript to combine and clean merged data, remove nulls, and concatenate arrays.

13. **Add a NocoDB Update node (Update NocoDB):**  
    - Update contact record by `id` with fields: messagesSent, delivered, opened, clicked, done=1.

14. **Connect Update NocoDB node back to Loop node:**  
    - This creates a loop to continue processing batches.

15. **Credential Setup:**  
    - Brevo API token configured in HTTP Request node with HTTP Header Auth.  
    - NocoDB API token configured in all NocoDB nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow splits the Brevo contact report into separate branches for each statistic type before merging them back.    | See Sticky Note3 in workflow notes for detailed explanation.                                             |
| JavaScript code nodes perform grouping and aggregation of multiple campaign records per email to prepare data for NocoDB. | Refer to Sticky Note4 for an overview of code logic used.                                                |
| The workflow requires valid API credentials for both Brevo (email marketing platform) and NocoDB (database backend).      | Credentials must be set up and tested before running the workflow.                                        |
| The workflow is designed to run on a schedule but can be modified for manual triggers or event-based triggers as needed.  |                                                                                                          |
| For more info on Brevo API, visit: https://developers.brevo.com/docs/contact-api                                         |                                                                                                          |
| For NocoDB API documentation, visit: https://docs.nocodb.com/noco-api-reference                                          |                                                                                                          |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.