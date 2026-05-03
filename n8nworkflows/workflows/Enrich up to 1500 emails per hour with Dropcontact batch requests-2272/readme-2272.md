Enrich up to 1500 emails per hour with Dropcontact batch requests

https://n8nworkflows.xyz/workflows/enrich-up-to-1500-emails-per-hour-with-dropcontact-batch-requests-2272


# Enrich up to 1500 emails per hour with Dropcontact batch requests

### 1. Workflow Overview

This n8n workflow automates the enrichment of up to 1500 emails per hour using Dropcontact's batch API. It is designed to process high volumes of contact data efficiently by batching requests (up to 250 per batch every 10 minutes) and updating a database with enriched email and contact information. The workflow is ideal for users needing scalable, automated email enrichment, especially when working with large datasets from corporate databases or CRM systems.

The logical structure is organized into the following blocks:

- **1.1 Input Reception and Querying**: Fetch raw profile data (names, domains) from a Postgres source with a limit to control batch size.
- **1.2 Data Aggregation and Transformation**: Aggregate and transform queried data into the JSON format required by Dropcontact’s batch API.
- **1.3 Bulk Batch Request Submission**: Submit the batch request to Dropcontact asynchronously and handle response IDs.
- **1.4 Wait and Batch Result Retrieval**: Pause to allow Dropcontact to process, then retrieve batch results.
- **1.5 Data Splitting and Database Update**: Split batch results, map fields, and update the Postgres database with enriched data.
- **1.6 Optional Error Notification**: Send Slack notifications in case of errors during batch downloading.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Querying

- **Overview:**  
  This block queries the source database (Postgres) to extract profile data (first name, last name, domain, full name) for contacts that need enrichment. It limits the results to a maximum of 1000 but the batch size is controlled downstream.

- **Nodes Involved:**  
  - Schedule Trigger  
  - PROFILES QUERY  
  - Loop Over Items2  
  - Aggregate

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Role: Initiates the workflow on a configured schedule (default interval unspecified, user-configurable).  
    - Input/Output: No input, output triggers PROFILES QUERY.  
    - Edge Cases: Ensure schedule is set to avoid overlapping runs causing batch overload.

  - **PROFILES QUERY**  
    - Type: Postgres node  
    - Role: Executes a SQL query to retrieve profiles matching criteria (e.g., title 'Bestuurder', no existing email, domain filtering).  
    - Configuration: SQL query limits results to 1000; filters out common public email domains; looks for profiles without emails and where dropcontact_found is null.  
    - Key Variables: first_name, last_name, domain, full_name  
    - Input: Trigger from Schedule Trigger  
    - Output: JSON array of profile data to Loop Over Items2  
    - Failure Modes: DB connection errors, query syntax errors, empty result sets.

  - **Loop Over Items2**  
    - Type: SplitInBatches  
    - Role: Splits the query results into batches of 250 items for Dropcontact batch limits.  
    - Configuration: batchSize set to 250 (max allowed per Dropcontact batch)  
    - Input: PROFILES QUERY output  
    - Output: First output to Aggregate (for batch aggregation), second output unused (no connection)  
    - Edge Cases: Batch size must not exceed 250; empty batches possible if no data.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates fields from batch items into arrays to prepare for transformation.  
    - Configuration: Aggregates first_name, last_name, domain, phantom_linkedin (not used downstream), full_name  
    - Input: Loop Over Items2 batch data  
    - Output: Passes aggregated arrays to DATA TRANSFORMATION  
    - Edge Cases: Missing or null fields could cause incomplete aggregation.

---

#### 2.2 Data Aggregation and Transformation

- **Overview:**  
  Transforms aggregated input arrays into the JSON structure expected by Dropcontact’s batch API, including custom fields for tracking.

- **Nodes Involved:**  
  - DATA TRANSFORMATION

- **Node Details:**

  - **DATA TRANSFORMATION**  
    - Type: Code node (Python)  
    - Role: Converts arrays of first_name, last_name, domain, full_name into a JSON object with the key "data" containing an array of contact objects formatted for Dropcontact.  
    - Configuration: Runs once per batch; builds list of dictionaries with keys: first_name, last_name, website (from domain), custom_fields containing full_name as a unique identifier.  
    - Key Expression: Uses Python zip() to iterate over arrays and build objects.  
    - Input: Aggregated arrays from Aggregate node  
    - Output: JSON formatted batch request body for Dropcontact  
    - Edge Cases: Assumes arrays are of equal length; mismatch or missing values may cause errors or malformed requests.  
    - Version Requirements: Python environment enabled in n8n, version compatible with node.

---

#### 2.3 Bulk Batch Request Submission

- **Overview:**  
  Sends the batch request JSON to Dropcontact’s batch API endpoint and triggers waiting for processing.

- **Nodes Involved:**  
  - BULK DROPCONTACT REQUESTS  
  - Wait2

- **Node Details:**

  - **BULK DROPCONTACT REQUESTS**  
    - Type: HTTP Request  
    - Role: Submits the batch data to Dropcontact asynchronously via POST.  
    - Configuration:  
      - URL: https://api.dropcontact.io/batch  
      - Method: POST  
      - Headers: X-Access-Token with Dropcontact API key (predefined credential)  
      - Body: JSON from DATA TRANSFORMATION node output  
      - Retries: 3 tries with 10 minutes wait between (600 seconds) on failure  
      - OnError: Continues workflow to allow graceful degradation  
    - Input: JSON batch request from DATA TRANSFORMATION  
    - Output: Batch request response containing request_id to Wait2  
    - Edge Cases: API rate limiting, authentication failure, network timeout.

  - **Wait2**  
    - Type: Wait node  
    - Role: Pauses workflow for 10 minutes (600 seconds) to allow Dropcontact to process batch asynchronously.  
    - Configuration: Fixed wait time 600 seconds  
    - Input: Output of BULK DROPCONTACT REQUESTS  
    - Output: Triggers BULK DROPCONTACT DOWNLOAD  
    - Edge Cases: Fixed wait assumes 10 minutes suffices; longer processing times may cause premature download attempts.

---

#### 2.4 Wait and Batch Result Retrieval

- **Overview:**  
  After waiting, fetches the processed batch results from Dropcontact using the request_id.

- **Nodes Involved:**  
  - BULK DROPCONTACT DOWNLOAD  
  - Slack (optional error notification)

- **Node Details:**

  - **BULK DROPCONTACT DOWNLOAD**  
    - Type: HTTP Request  
    - Role: Fetches batch results using the asynchronous request_id from Dropcontact.  
    - Configuration:  
      - URL: https://api.dropcontact.io/batch/{{ $json.request_id }} (dynamic)  
      - Authentication: Dropcontact API key (same credentials as above)  
      - OnError: Continues workflow, but triggers Slack notification on failure  
    - Input: Output from Wait2 (contains request_id)  
    - Output: JSON batch response with enriched contact data forwarded to Split Out node  
    - Edge Cases: Request ID invalid or expired, partial results, API quota errors.

  - **Slack (Optional)**  
    - Type: Slack node  
    - Role: Sends error notification if batch download fails (e.g., due to credit issues).  
    - Configuration: User must connect Slack credentials; text message set to "Dropcontact Credits issue: url"  
    - Input: Error output from BULK DROPCONTACT DOWNLOAD  
    - Edge Cases: Slack API connectivity issues, invalid user/channel configuration.

---

#### 2.5 Data Splitting and Database Update

- **Overview:**  
  Splits the batch results into individual contact records and updates the Postgres database with enriched email and phone data.

- **Nodes Involved:**  
  - Split Out  
  - Postgres

- **Node Details:**

  - **Split Out**  
    - Type: SplitOut node  
    - Role: Splits the batch response “data” array into individual items for separate processing.  
    - Configuration: Field to split out is "data" (the array of contact info)  
    - Input: BULK DROPCONTACT DOWNLOAD output  
    - Output: Splits individual records forwarded to Postgres node  
    - Edge Cases: Empty data arrays, malformed response JSON.

  - **Postgres**  
    - Type: Postgres node  
    - Role: Updates the profiles table with enriched data fields for each contact.  
    - Configuration:  
      - Operation: Update by matching on "full_name" field (unique identifier)  
      - Columns updated: email, phone, dropcontact_found (true), email_qualification  
      - Maps output JSON fields (e.g., $json.email[0].email) to DB columns  
      - Retries: 2 max, continues on error output  
    - Input: Individual contact records from Split Out  
    - Output: Loops back to Loop Over Items2 to potentially process next batch  
    - Edge Cases: DB update conflicts, missing matching full_name, connectivity issues.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                          | Input Node(s)          | Output Node(s)             | Sticky Note                                                 |
|--------------------------|--------------------|----------------------------------------|-----------------------|----------------------------|-------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger   | Initiates workflow on schedule          | -                     | PROFILES QUERY              |                                                             |
| PROFILES QUERY           | Postgres           | Queries profile data                     | Schedule Trigger       | Loop Over Items2            |                                                             |
| Loop Over Items2         | SplitInBatches     | Splits data into batches of 250          | PROFILES QUERY         | Aggregate                  |                                                             |
| Aggregate                | Aggregate          | Aggregates fields into arrays            | Loop Over Items2       | DATA TRANSFORMATION         |                                                             |
| DATA TRANSFORMATION      | Code (Python)      | Formats data into Dropcontact batch JSON | Aggregate              | BULK DROPCONTACT REQUESTS   |                                                             |
| BULK DROPCONTACT REQUESTS| HTTP Request       | Submits batch request to Dropcontact     | DATA TRANSFORMATION    | Wait2                      |                                                             |
| Wait2                    | Wait               | Waits 10 minutes for batch processing    | BULK DROPCONTACT REQUESTS | BULK DROPCONTACT DOWNLOAD |                                                             |
| BULK DROPCONTACT DOWNLOAD| HTTP Request       | Downloads batch results                   | Wait2                  | Split Out, Slack            |                                                             |
| Split Out                | SplitOut           | Splits batch results into individual items | BULK DROPCONTACT DOWNLOAD | Postgres                 |                                                             |
| Postgres                 | Postgres           | Updates database with enriched info      | Split Out              | Loop Over Items2            |                                                             |
| Slack                    | Slack              | Sends error notification on failure      | BULK DROPCONTACT DOWNLOAD (error) | -               | Optional: Notify if Dropcontact credits issue occurs        |
| Sticky Note2             | Sticky Note        | Documentation and guide                   | -                      | -                          | **DROPCONTACT 250 BATCH ASYNCHRONOUSLY 1500/HOUR REQUESTS** [Guide](https://docs.n8n.io/workflows/sticky-notes/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Set the desired interval for workflow execution (e.g., every 10 minutes)  
   - No inputs, connects to PROFILES QUERY.

2. **Create PROFILES QUERY node (Postgres)**  
   - Configure PostgreSQL credentials (your DB connection)  
   - Use SQL query to fetch profiles:  
     ```sql
     select first_name, last_name, domain, full_name
     from accounts a 
     left join profiles p on a.company_id = p.company_id 
     where title = 'Bestuurder' and p.email is null and a.domain != ''
     and domain NOT IN ('gmail.com', 'hotmail.com', 'hotmail.be', 'hotmail%','outlook.com','telenet.be', 'live.be', 'skynet.be','SKYNET%', 'yahoo.com' , 'yahoo%', 'msn%', 'hotmail', 'belgacom%') 
     and dropcontact_found is null 
     limit 1000
     ```  
   - Connect Schedule Trigger main output to this node’s input.

3. **Create Loop Over Items2 node (SplitInBatches)**  
   - Set batchSize to 250  
   - Connect PROFILES QUERY main output to this node.

4. **Create Aggregate node**  
   - Configure to aggregate fields: first_name, last_name, domain, phantom_linkedin (optional), full_name  
   - Connect Loop Over Items2 first output to Aggregate.

5. **Create DATA TRANSFORMATION node (Code node, Python)**  
   - Code to transform aggregated arrays into Dropcontact batch format:  
     ```python
     import json

     for item in _input.all():
       data = item.json 

       output_data = {"data": [], "siren": True}

       first_names = data["first_name"]
       last_names = data["last_name"]
       domain = data["domain"]
       full_name = data["full_name"]

       transformed_data = []
       for i, (first_name, last_name, domain_name, full_name_value) in enumerate(zip(first_names, last_names, domain, full_name)):
         transformed_data.append({
           "first_name": first_name,
           "last_name": last_name,
           "website": domain_name,
           "custom_fields": {
             "full_name": full_name_value}
         })

       output_data["data"] = transformed_data

       return output_data 
     ```  
   - Connect Aggregate main output to this node.

6. **Create BULK DROPCONTACT REQUESTS node (HTTP Request)**  
   - URL: https://api.dropcontact.io/batch  
   - Method: POST  
   - Authentication: Use Dropcontact API credentials (predefined credential type)  
   - Headers: Add header `X-Access-Token` with value from credentials  
   - Body: JSON, send the output of DATA TRANSFORMATION as JSON body  
   - Retry on failure: 3 times with 600 seconds delay  
   - On error: continue workflow  
   - Connect DATA TRANSFORMATION output to this node.

7. **Create Wait2 node (Wait)**  
   - Set wait time: 600 seconds (10 minutes)  
   - Connect BULK DROPCONTACT REQUESTS main output to Wait2.

8. **Create BULK DROPCONTACT DOWNLOAD node (HTTP Request)**  
   - URL: Set dynamically to `https://api.dropcontact.io/batch/{{ $json.request_id }}` where request_id is from the previous node's response  
   - Authentication: Same Dropcontact credentials as above  
   - On error: Continue workflow and trigger Slack node  
   - Connect Wait2 output to this node.

9. **Create Split Out node (SplitOut)**  
   - Field to split out: `data`  
   - Connect BULK DROPCONTACT DOWNLOAD main output to Split Out.

10. **Create Postgres node**  
    - Operation: Update  
    - Credentials: Same Postgres account as PROFILES QUERY  
    - Table: profiles in schema public  
    - Matching column: full_name  
    - Columns to update:  
      - email → `$json.email[0].email`  
      - phone → `$json.phone`  
      - dropcontact_found → `true`  
      - email_qualification → `$json.email[0].qualification`  
    - Retry on failure: 2 times  
    - On error: continue  
    - Connect Split Out output to this node  
    - Connect Postgres output back to Loop Over Items2 to continue processing remaining batches.

11. **Create Slack node (Optional)**  
    - Connect to BULK DROPCONTACT DOWNLOAD error output  
    - Configure Slack credentials and set message text: "Dropcontact Credits issue: url"  
    - Optional notification for batch request failures or credit issues.

12. **Create Sticky Note node**  
    - Add documentation content as per workflow overview and link to [n8n Sticky Notes guide](https://docs.n8n.io/workflows/sticky-notes/)  
    - Place visually near key nodes for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| The workflow enforces a maximum of 250 items per batch to comply with Dropcontact API limits (1500 requests/hour). | Important for API rate limiting and batch processing logic.         |
| Make sure the source query returns only the required fields and respects the batch size limit.                   | See Step 1 notes in workflow overview.                              |
| Dropcontact's batch API requires asynchronous POST submission plus delayed GET retrieval of results.            | See Step 3 and 4 notes; ensures proper batch processing.            |
| Use a unique identifier (full_name) to reconcile returned data with source records.                              | Crucial for accurate database updates in Step 5.                    |
| Slack notifications are optional but recommended for monitoring API credit issues or errors.                    | Helps maintain operational awareness without manual checks.         |
| Initial test runs with small batch sizes (e.g., 10) advised before scaling up to 250 per batch.                  | Minimizes risk of mapping errors and API quota overruns.             |
| [n8n Sticky Notes Documentation](https://docs.n8n.io/workflows/sticky-notes/)                                    | Useful for embedding workflow documentation inside n8n workflows.  |

---

This comprehensive reference provides full insight into the workflow’s structure, function, and reproduction steps, enabling users and automation agents to understand, replicate, and maintain the email enrichment process with Dropcontact at scale.