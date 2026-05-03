Monitor Amazon Product Prices with Bright Data and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-amazon-product-prices-with-bright-data-and-google-sheets-3878


# Monitor Amazon Product Prices with Bright Data and Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of Amazon product prices using Bright Data’s Amazon Scraper API and Google Sheets. It reads product URLs, ZIP codes, and ASINs from a Google Sheet, queries Bright Data’s API for price and product data, and writes updated pricing information back to the sheet.

The workflow is designed for periodic execution and includes the following logical blocks:

- **1.1 Schedule and Input Retrieval**: Trigger the workflow periodically and read product data from Google Sheets.
- **1.2 Batch Processing and Request Initiation**: Split data into batches and send requests to Bright Data’s API for price scraping.
- **1.3 Snapshot Status Monitoring**: Poll Bright Data API to check if data scraping is complete.
- **1.4 Data Retrieval and Update**: Fetch the completed data snapshot and update Google Sheets with the latest prices.
- **1.5 Wait/Delay Handling**: Manage timing to space requests and polling intervals.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Input Retrieval

**Overview:**  
This block triggers the workflow on a schedule and reads product details (Product URL, ZIP code, ASIN) from a configured Google Sheet to serve as input data for the price monitoring.

**Nodes Involved:**  
- Schedule Trigger  
- Read data from Google Sheet  
- Sticky Note (contextual instruction)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution periodically (default interval set, can be customized).  
  - Configuration: Default interval with no specific time constraints shown; can be set to hourly, daily, etc.  
  - Connections: Outputs to "Read data from Google Sheet".  
  - Edge Cases: Misconfiguration can lead to too frequent or infrequent triggering; ensure correct timezone.  
  - Version: 1.2

- **Read data from Google Sheet**  
  - Type: Google Sheets Node  
  - Role: Reads rows from a Google Sheet containing Product URL, ZIP code, and ASIN columns.  
  - Configuration:  
    - Document ID and Sheet Name set to a specific Google Sheet.  
    - OAuth2 credentials linked to Google account.  
  - Inputs: From Schedule Trigger  
  - Outputs: Product data JSON array to next node.  
  - Edge Cases:  
    - Authentication errors if credentials expire.  
    - Incorrect sheet name or document ID causes failure.  
    - Empty or misformatted rows lead to empty or invalid data.  
  - Version: 4.5

- **Sticky Note**  
  - Provides instructions on sheet setup, especially the ASIN extraction formula:  
    ```
    =REGEXEXTRACT(A4, "/(?:dp|gp/product|product)/([A-Z0-9]{10})")
    ```
  - No direct workflow effect.

---

#### 2.2 Batch Processing and Request Initiation

**Overview:**  
Splits the input data into batches of 10 to efficiently send requests to Bright Data’s Amazon Scraper API. This controls API load and manages rate limits.

**Nodes Involved:**  
- Process URLs by batch of 10s  
- Combine the batch  
- Initiate request from URLs  
- Space the request by 1 second  
- Sticky Note (contextual instruction)

**Node Details:**

- **Process URLs by batch of 10s**  
  - Type: SplitInBatches  
  - Role: Splits input data into batches of 10 items.  
  - Configuration: Batch size set to 10.  
  - Inputs: From "Read data from Google Sheet" or from "Space the request by 1 second" (for loop).  
  - Outputs: Single batch of 10 rows per execution.  
  - Edge Cases:  
    - Last batch may have fewer than 10 items.  
    - If input is empty, node outputs nothing.  
  - Version: 3

- **Combine the batch**  
  - Type: Aggregate  
  - Role: Aggregates all batch outputs back into a single array to send as one request.  
  - Configuration: Aggregates all item data.  
  - Inputs: From "Process URLs by batch of 10s" (second output path).  
  - Outputs: Combined JSON array for API request.  
  - Edge Cases: Empty input leads to empty aggregation.  
  - Version: 1

- **Initiate request from URLs**  
  - Type: HTTP Request  
  - Role: Sends POST request to Bright Data’s API to trigger price scraping for the batch of product URLs and ZIP codes.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Method: POST  
    - Body: JSON array constructed from input batch with keys "url" and "zipcode" extracted from Google Sheet data.  
    - Query parameters: `dataset_id` set to Bright Data dataset, `include_errors=true`.  
    - Authentication: Bearer token from Bright Data API key credential.  
  - Inputs: From "Combine the batch".  
  - Outputs: Contains a `snapshot_id` for tracking the data extraction status.  
  - Edge Cases:  
    - API rate limits or authentication failures.  
    - Invalid URLs or ZIP codes may cause errors in response.  
  - Version: 4.2

- **Space the request by 1 second**  
  - Type: Wait  
  - Role: Inserts a delay of 1 second between batch processing cycles to avoid API overload or rate limiting.  
  - Configuration: Wait 1 second.  
  - Inputs: From "Initiate request from URLs".  
  - Outputs: Loops back to "Process URLs by batch of 10s" for next batch or ends if no batches remain.  
  - Edge Cases: Delays increase total workflow execution time.  
  - Version: 1.1

- **Sticky Note1**  
  - Instruction on obtaining Bright Data API key and setting up Bearer Authentication.  
  - No direct workflow effect.

---

#### 2.3 Snapshot Status Monitoring

**Overview:**  
Polls the Bright Data API using the snapshot ID to check if the scraping job is complete before attempting to retrieve data.

**Nodes Involved:**  
- Check Status by Snapshot ID  
- Check the status of snapshot (Switch)  
- Wait  
- Sticky Note (contextual instruction)

**Node Details:**

- **Check Status by Snapshot ID**  
  - Type: HTTP Request  
  - Role: Sends GET request to Bright Data API to check progress of the dataset snapshot using the snapshot ID from previous API response.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (dynamic snapshot ID)  
    - Authentication: Bearer token via Bright Data API key.  
  - Inputs: From "Wait".  
  - Outputs: JSON containing status field (e.g., "running", "ready").  
  - Edge Cases:  
    - Expired or invalid snapshot ID leads to errors.  
    - Network timeouts or API downtime.  
  - Version: 4.2

- **Check the status of snapshot**  
  - Type: Switch  
  - Role: Routes the workflow depending on snapshot status.  
  - Configuration:  
    - If status equals "ready" → proceed to data retrieval.  
    - If status equals "running" → loop back to wait and re-check.  
  - Inputs: From "Check Status by Snapshot ID".  
  - Outputs: True branch to "Get the data", False branch to "Wait".  
  - Edge Cases: Unexpected status values may cause workflow to stall.  
  - Version: 3.2

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow for a period before rechecking snapshot status.  
  - Configuration: Default wait (parameters not explicitly set, but typically a delay here).  
  - Inputs: From "Check the status of snapshot" (False branch).  
  - Outputs: Loops back to "Check Status by Snapshot ID".  
  - Edge Cases: Extended waiting if snapshot takes long to process; must balance delay time with responsiveness.  
  - Version: 1.1

- **Sticky Note2**  
  - Explains this block checks the snapshot status and proceeds only if data is ready.  
  - No direct workflow effect.

---

#### 2.4 Data Retrieval and Update

**Overview:**  
Once scraping is complete, this block retrieves the data snapshot and updates the Google Sheet with the latest price information keyed by ASIN.

**Nodes Involved:**  
- Get the data  
- Update the records by ASIN  
- Sticky Note (contextual instruction)

**Node Details:**

- **Get the data**  
  - Type: HTTP Request  
  - Role: Retrieves the scraped data JSON snapshot from Bright Data API using snapshot ID.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query parameter: format=json  
    - Authentication: Bearer token from Bright Data API key.  
  - Inputs: From "Check the status of snapshot" (True branch).  
  - Outputs: JSON data containing product prices and other metadata.  
  - Edge Cases:  
    - Snapshot ID invalid or expired.  
    - Large data payloads causing timeouts.  
  - Version: 4.2

- **Update the records by ASIN**  
  - Type: Google Sheets Node  
  - Role: Updates Google Sheet rows matching ASIN with the new price data.  
  - Configuration:  
    - Document ID and Sheet Name same as input sheet.  
    - Operation: Update existing rows by matching on ASIN column.  
    - Columns updated: "Price" and "ASIN" (ASIN used as key).  
  - Inputs: From "Get the data".  
  - Outputs: Updated rows confirmation.  
  - Edge Cases:  
    - ASIN mismatch causing failed updates.  
    - Sheet protection or permission errors.  
    - Data type mismatches.  
  - Version: 4.5

- **Sticky Note3**  
  - Informs that this block obtains data from snapshot and updates Google Sheets accordingly.  
  - No direct workflow effect.

---

#### 2.5 Wait/Delay Handling

**Overview:**  
Manages timing delays for pacing API requests and snapshot status polling.

**Nodes Involved:**  
- Wait (used in multiple parts)  
- Space the request by 1 second (dedicated wait for batch spacing)

**Node Details:**

- **Wait (general)**  
  - Type: Wait  
  - Role: Pauses workflow to avoid API rate limits or to wait for data readiness.  
  - Configuration: Varies by usage; one instance has 1 second delay for batch spacing, another used for polling delay.  
  - Version: 1.1  
  - Edge Cases: Excessive wait times can delay overall workflow completion.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                             |
|----------------------------|-------------------------|-----------------------------------------------|------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger        | Periodic workflow initiator                    |                              | Read data from Google Sheet   |                                                                                                       |
| Read data from Google Sheet | Google Sheets           | Reads product data (URL, ZIP, ASIN)            | Schedule Trigger             | Process URLs by batch of 10s  | ## Schedule and Read from Google Sheets: Set up sheet columns and ASIN extraction formula              |
| Process URLs by batch of 10s | SplitInBatches          | Splits input data into batches of 10            | Read data from Google Sheet / Space the request by 1 second | Wait / Combine the batch     | ## Process Data into Bright Data by batch: Obtain Bright Data API key and setup Bearer auth            |
| Combine the batch          | Aggregate               | Aggregates batch items into one array            | Process URLs by batch of 10s | Initiate request from URLs    | ## Process Data into Bright Data by batch: Obtain Bright Data API key and setup Bearer auth            |
| Initiate request from URLs | HTTP Request            | Sends batch request to Bright Data Amazon API   | Combine the batch            | Space the request by 1 second | ## Process Data into Bright Data by batch: Obtain Bright Data API key and setup Bearer auth            |
| Space the request by 1 second | Wait                    | Delays 1 second between batches                  | Initiate request from URLs   | Process URLs by batch of 10s  | ## Process Data into Bright Data by batch: Obtain Bright Data API key and setup Bearer auth            |
| Wait                       | Wait                    | General delay (used for polling snapshot status) | Check the status of snapshot (False branch) | Check Status by Snapshot ID | ## Check the requested data from Bright Data using snapshot ID: Checks if snapshot is ready            |
| Check Status by Snapshot ID | HTTP Request            | Polls Bright Data API for snapshot progress      | Wait                        | Check the status of snapshot  | ## Check the requested data from Bright Data using snapshot ID: Checks if snapshot is ready            |
| Check the status of snapshot | Switch                  | Routes based on snapshot status                    | Check Status by Snapshot ID  | Get the data / Wait           | ## Check the requested data from Bright Data using snapshot ID: Checks if snapshot is ready            |
| Get the data               | HTTP Request            | Retrieves scraped data snapshot                    | Check the status of snapshot (True branch) | Update the records by ASIN    | ## Obtain the data from the snapshot and update it to Google Sheets                                    |
| Update the records by ASIN | Google Sheets           | Updates Google Sheet rows with latest prices      | Get the data                |                              | ## Obtain the data from the snapshot and update it to Google Sheets                                    |
| Sticky Note                | Sticky Note             | Instructional notes                               |                              |                              | See content in section 2.1                                                                             |
| Sticky Note1               | Sticky Note             | Instructional notes                               |                              |                              | See content in section 2.2                                                                             |
| Sticky Note2               | Sticky Note             | Instructional notes                               |                              |                              | See content in section 2.3                                                                             |
| Sticky Note3               | Sticky Note             | Instructional notes                               |                              |                              | See content in section 2.4                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configuration: Set desired repeat interval (e.g., hourly, daily).  

2. **Create Google Sheets Node to Read Data**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Parameters:  
     - Document ID: Your Google Sheet ID  
     - Sheet Name: Sheet tab or gid (e.g., gid=0)  
   - Credentials: Configure Google OAuth2 credentials with access to the target sheet  
   - Connect input from Schedule Trigger  

3. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Parameters: Batch size 10  
   - Connect input from Google Sheets node  

4. **Create Aggregate Node**  
   - Type: Aggregate  
   - Parameters: Aggregate all item data into a single array  
   - Connect input from second output of SplitInBatches node (batches completion)  

5. **Create HTTP Request Node to Initiate Bright Data API Request**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://api.brightdata.com/datasets/v3/trigger  
   - Authentication: Bearer token using Bright Data API key credential  
   - Query Parameters:  
     - dataset_id: Your Bright Data dataset ID (e.g., gd_l7q7dkf244hwjntr0)  
     - include_errors: true  
   - Body: JSON array constructed from batch input using expressions:  
     ```js
     {{$json.data.map(i => ({ url: i['Product URL'], zipcode: i['ZIP code'] }))}}
     ```  
   - Content-Type: JSON  
   - Connect input from Aggregate node  

6. **Create Wait Node (Space the request by 1 second)**  
   - Type: Wait  
   - Parameters: Wait 1 second  
   - Connect input from HTTP Request (Initiate request) node  

7. **Connect Wait node output back to SplitInBatches**  
   - This creates a loop to process all batches sequentially with delay.  

8. **Create HTTP Request Node to Check Snapshot Status**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (use expression for snapshot_id)  
   - Authentication: Bearer token with Bright Data API key  
   - Connect input from Wait node which is triggered after batch processing  

9. **Create Switch Node to Check Snapshot Status**  
   - Type: Switch  
   - Rules:  
     - If `status` equals "ready" → proceed to next node  
     - If `status` equals "running" → loop back to Wait node  
   - Connect input from "Check Status by Snapshot ID" node  

10. **Create Wait Node for Polling Delay**  
    - Type: Wait  
    - Parameters: Set appropriate delay (e.g., several seconds) to avoid rapid polling  
    - Connect input from Switch node False branch (status "running")  
    - Connect output to "Check Status by Snapshot ID" node (poll again)  

11. **Create HTTP Request Node to Get Snapshot Data**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query Parameter: format=json  
    - Authentication: Bearer token with Bright Data API key  
    - Connect input from Switch node True branch (status "ready")  

12. **Create Google Sheets Node to Update Records**  
    - Type: Google Sheets  
    - Operation: Update rows  
    - Parameters:  
      - Document ID and Sheet Name same as input sheet  
      - Matching column: ASIN  
      - Columns to update: Price (mapped from snapshot data), ASIN  
    - Credentials: Google OAuth2 same as before  
    - Connect input from "Get the data" node  

13. **Create Sticky Notes** (optional but recommended for documentation)  
    - Add instructional sticky notes near relevant nodes describing:  
      - Sheet setup and ASIN extraction formula  
      - Bright Data API key and authentication setup  
      - Snapshot status checking logic  
      - Data update logic  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The ASIN extraction formula for Google Sheets is: `=REGEXEXTRACT(A2, "/(?:dp|gp/product|product)/([A-Z0-9]{10})")` to extract ASIN from product URLs.                      | Google Sheets formula for ASIN extraction                                                           |
| Ensure your Bright Data API key is securely stored as a Bearer Authentication Credential in n8n for authorized API calls.                                                 | Bright Data API key handling                                                                         |
| The workflow supports batching of 10 requests with 1-second spacing to avoid rate limits imposed by Bright Data API.                                                      | Rate limiting and batching best practices                                                           |
| Use Google Sheets OAuth2 credentials with appropriate permissions to read and update the monitoring sheet.                                                                 | Google Sheets API OAuth2 setup                                                                        |
| You can customize the workflow to add alerts (email or Slack) for price changes, or extend it to collect more product attributes like stock or ratings.                    | Workflow extensibility notes                                                                         |
| Bright Data API docs: https://brightdata.com/docs/api/datasets                                                                                                             | Official API documentation                                                                            |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively from an n8n automation workflow. All operations comply with applicable content policies and involve only legal and public data.