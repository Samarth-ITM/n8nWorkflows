Skip trace people with ScraperCity and save contact details to Google Sheets

https://n8nworkflows.xyz/workflows/skip-trace-people-with-scrapercity-and-save-contact-details-to-google-sheets-13883


# Skip trace people with ScraperCity and save contact details to Google Sheets

# 1. Workflow Overview

This workflow performs a **skip trace / people lookup** using the **ScraperCity People Finder API**, waits for the asynchronous job to complete, downloads the resulting CSV, parses and deduplicates the contact data, and then saves the output into **Google Sheets**.

Typical use cases include:
- Looking up contact details for one or more people from names, phones, or emails
- Enriching lead or prospect lists
- Creating a repeatable pipeline to store skip-trace results in a spreadsheet

The workflow is organized into the following logical blocks:

## 1.1 Input Reception and Search Configuration
The workflow starts manually. A Set node captures comma-separated search inputs such as names, phones, and emails, then a Code node converts these values into the request structure expected by ScraperCity.

## 1.2 Job Submission
The prepared payload is sent to the ScraperCity People Finder endpoint. The returned `runId` is extracted and stored for later status checks and file download.

## 1.3 Asynchronous Polling Loop
Because ScraperCity jobs are not immediate, the workflow enters a loop: wait 60 seconds, check job status, and decide whether to continue waiting or proceed.

## 1.4 Results Download, Parsing, and Storage
Once the job succeeds, the workflow downloads the CSV output, parses it into records, removes duplicates, and appends or updates rows in Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Search Configuration

### Overview
This block initializes the workflow and prepares the search criteria. It converts user-friendly comma-separated text fields into arrays required by the ScraperCity API.

### Nodes Involved
- When clicking 'Execute workflow'
- Configure Search Inputs
- Build Request Body

### Node Details

#### 1) When clicking 'Execute workflow'
- **Type and role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point used for testing or ad hoc execution.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: Configure Search Inputs
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:**  
  - No runtime failure expected unless workflow execution is interrupted manually.
- **Sub-workflow reference:** None.

#### 2) Configure Search Inputs
- **Type and role:** `n8n-nodes-base.set`  
  Defines the initial search values.
- **Configuration choices:** Creates four fields:
  - `names` = `"John Smith,Jane Doe"`
  - `phones` = `""`
  - `emails` = `""`
  - `max_results` = `1`
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**  
  - Input: When clicking 'Execute workflow'  
  - Output: Build Request Body
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases / failures:**  
  - Empty inputs are allowed, but sending no usable identifiers may produce poor or empty results from the API.
  - `max_results` should remain a positive number.
- **Sub-workflow reference:** None.

#### 3) Build Request Body
- **Type and role:** `n8n-nodes-base.code`  
  Transforms comma-separated text into arrays and builds the API request object.
- **Configuration choices:**  
  The script:
  - Splits `names`, `emails`, and `phones` on commas
  - Trims whitespace
  - Removes empty values
  - Maps the resulting arrays to:
    - `name`
    - `email`
    - `phone_number`
    - `street_citystatezip` as an empty array
    - `max_results`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `cfg.names`
  - `cfg.emails`
  - `cfg.phones`
  - `cfg.max_results`
- **Input and output connections:**  
  - Input: Configure Search Inputs  
  - Output: Submit Skip Trace Job
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases / failures:**  
  - If upstream fields are missing, the code still handles them reasonably by converting falsy values into empty arrays.
  - If `max_results` is missing, it defaults to `1`.
  - API-side validation may fail if all search arrays are empty.
- **Sub-workflow reference:** None.

---

## 2.2 Job Submission

### Overview
This block sends the skip trace request to ScraperCity and stores the returned job identifier. That `runId` is the central reference used by all later polling and download steps.

### Nodes Involved
- Submit Skip Trace Job
- Store Run ID

### Node Details

#### 4) Submit Skip Trace Job
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to create a ScraperCity People Finder job.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
  - Body type: JSON
  - Body content: serialized current item JSON
  - Authentication: generic credential type using header auth
- **Key expressions or variables used:**
  - `={{ JSON.stringify($json) }}`
- **Input and output connections:**  
  - Input: Build Request Body  
  - Output: Store Run ID
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**  
  - Invalid or missing API key
  - Incorrect header auth setup
  - API rejection due to malformed body or unsupported search criteria
  - Network timeout or remote service failure
- **Sub-workflow reference:** None.
- **Credential requirements:**  
  Requires a Header Auth credential named like **ScraperCity API Key**, typically:
  - Header name: `Authorization`
  - Header value: `Bearer YOUR_KEY`

#### 5) Store Run ID
- **Type and role:** `n8n-nodes-base.set`  
  Extracts `runId` from the API response and stores it in a predictable field.
- **Configuration choices:** Sets:
  - `runId = {{ $json.runId }}`
- **Key expressions or variables used:**
  - `={{ $json.runId }}`
- **Input and output connections:**  
  - Input: Submit Skip Trace Job  
  - Output: Polling Loop
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases / failures:**  
  - If the API response does not include `runId`, all downstream status/download expressions will fail.
  - A partial or error response from ScraperCity may cause this node to output an empty string.
- **Sub-workflow reference:** None.

---

## 2.3 Asynchronous Polling Loop

### Overview
This block repeatedly checks whether the submitted job has completed. It pauses for 60 seconds between checks and loops until the returned status equals `SUCCEEDED`.

### Nodes Involved
- Polling Loop
- Wait 60 Seconds
- Check Scrape Status
- Is Scrape Complete?

### Node Details

#### 6) Polling Loop
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Used here as a loop-control mechanism rather than for real batching.
- **Configuration choices:**
  - Batch size: `1`
  - Reset: `false`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - Input: Store Run ID, Is Scrape Complete? (false branch loopback)
  - Output 0: Wait 60 Seconds  
  - Output 1: unused
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases / failures:**  
  - This pattern works as a loop, but if workflow logic changes incorrectly it may create an unintended infinite polling cycle.
  - Since no max-attempt guard exists, jobs stuck in non-terminal states could cause very long-running executions.
- **Sub-workflow reference:** None.

#### 7) Wait 60 Seconds
- **Type and role:** `n8n-nodes-base.wait`  
  Introduces a 60-second delay before each status check.
- **Configuration choices:** Wait amount is `60`.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: Polling Loop  
  - Output: Check Scrape Status
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases / failures:**  
  - Long waits increase total execution duration.
  - Depending on n8n hosting mode, wait behavior may require proper persistence/execution resumption support.
- **Sub-workflow reference:** None.

#### 8) Check Scrape Status
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Queries the ScraperCity status endpoint for the submitted run.
- **Configuration choices:**
  - Method: `GET`
  - URL uses the stored run ID
  - Authentication: generic header auth
- **Key expressions or variables used:**
  - `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input: Wait 60 Seconds  
  - Output: Is Scrape Complete?
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**  
  - Missing `runId`
  - Invalid API key
  - Run not found or expired
  - API temporarily unavailable
- **Sub-workflow reference:** None.

#### 9) Is Scrape Complete?
- **Type and role:** `n8n-nodes-base.if`  
  Routes execution depending on whether the returned scrape status is complete.
- **Configuration choices:**
  - Condition checks whether `$json.status` equals `SUCCEEDED`
  - Strict type validation enabled
  - Case-sensitive comparison
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: Check Scrape Status  
  - True output: Download Results CSV  
  - False output: Polling Loop
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases / failures:**  
  - Any terminal non-success state such as `FAILED`, `CANCELLED`, or unexpected status will loop forever because only `SUCCEEDED` exits the loop.
  - If the API changes status naming or casing, the comparison may never match.
- **Sub-workflow reference:** None.

---

## 2.4 Results Download, Parsing, and Storage

### Overview
After success is confirmed, this block downloads the output CSV, converts it into structured contact records, removes duplicates, and writes the final dataset to Google Sheets.

### Nodes Involved
- Download Results CSV
- Parse and Format CSV Results
- Remove Duplicate Contacts
- Write Results to Google Sheets

### Node Details

#### 10) Download Results CSV
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed job results from ScraperCity.
- **Configuration choices:**
  - Method: `GET`
  - URL uses the stored `runId`
  - Response format: text
  - Authentication: generic header auth
- **Key expressions or variables used:**
  - `=https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input: Is Scrape Complete? (true branch)  
  - Output: Parse and Format CSV Results
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**  
  - Missing `runId`
  - File not yet available despite status check
  - Empty or non-CSV response
  - Auth or download endpoint failure
- **Sub-workflow reference:** None.

#### 11) Parse and Format CSV Results
- **Type and role:** `n8n-nodes-base.code`  
  Parses CSV text manually into JSON records and performs an initial deduplication.
- **Configuration choices:**  
  The script:
  - Reads CSV text from `json.data` or `json.body`
  - Returns an error object if no text is present
  - Splits content by newline
  - Parses the header row and data rows
  - Uses a custom CSV parser with basic quote handling
  - Builds one JSON item per row
  - Deduplicates using `name + primary_phone` (or fallback `phone`)
  - Returns an error item if no records are parsed
- **Key expressions or variables used:**
  - `$input.first().json.data`
  - `$input.first().json.body`
  - `$('Store Run ID').item.json.runId`
  - Record fields such as `name`, `primary_phone`, `phone`
- **Input and output connections:**  
  - Input: Download Results CSV  
  - Output: Remove Duplicate Contacts
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases / failures:**  
  - CSV parsing is custom and may fail on complex CSV cases such as escaped quotes, embedded line breaks, or unusual delimiters.
  - If the response is empty, an error object is emitted instead of normal contact rows.
  - If headers differ from expected field names, deduplication quality may decrease.
- **Sub-workflow reference:** None.

#### 12) Remove Duplicate Contacts
- **Type and role:** `n8n-nodes-base.removeDuplicates`  
  Performs a second deduplication step across parsed records.
- **Configuration choices:**
  - Compare mode: selected fields
  - Fields compared:
    - `name`
    - `primary_phone`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: Parse and Format CSV Results  
  - Output: Write Results to Google Sheets
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases / failures:**  
  - If parsed items contain error objects instead of contact records, those may still pass through and be treated as unique rows.
  - Records with same person but slightly different phone formatting may not be recognized as duplicates.
- **Sub-workflow reference:** None.

#### 13) Write Results to Google Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Appends or updates records in a target Google Sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Column mapping: auto-map input data
  - Sheet name: configured by ID expression placeholder
  - Document ID: configured by ID expression placeholder
- **Key expressions or variables used:**
  - `sheetName.value = "="`
  - `documentId.value = "="`
- **Input and output connections:**  
  - Input: Remove Duplicate Contacts  
  - Output: none
- **Version-specific requirements:** `typeVersion: 4.6`
- **Edge cases / failures:**  
  - As provided, both spreadsheet document ID and sheet identifier are not configured and must be filled in before use.
  - OAuth2 credential may be missing or unauthorized.
  - Auto-mapping depends on header names matching incoming JSON fields or sheet columns.
  - `appendOrUpdate` may require a defined matching column configuration depending on n8n version and target sheet structure.
- **Sub-workflow reference:** None.
- **Credential requirements:**  
  Requires a Google Sheets OAuth2 credential.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual start of the workflow |  | Configure Search Inputs | ## How it works<br>1. You enter names, phone numbers, or emails in the Configure Search Inputs node.<br>2. The workflow submits a skip trace job to the ScraperCity People Finder API.<br>3. It polls for completion every 60 seconds (jobs take 10-60 min).<br>4. Once done, it downloads the CSV, parses each contact, deduplicates, and writes rows to Google Sheets.<br><br>## Setup steps<br>1. Create a Header Auth credential named **ScraperCity API Key** -- set header to `Authorization`, value to `Bearer YOUR_KEY`.<br>2. Connect a Google Sheets OAuth2 credential.<br>3. Edit **Configure Search Inputs** with your lookup targets.<br>4. Set your Google Sheet ID in **Write Results to Google Sheets**.<br>5. Click Execute workflow. |
| Configure Search Inputs | Set | Defines names, phones, emails, and result limit | When clicking 'Execute workflow' | Build Request Body | ## How it works<br>1. You enter names, phone numbers, or emails in the Configure Search Inputs node.<br>2. The workflow submits a skip trace job to the ScraperCity People Finder API.<br>3. It polls for completion every 60 seconds (jobs take 10-60 min).<br>4. Once done, it downloads the CSV, parses each contact, deduplicates, and writes rows to Google Sheets.<br><br>## Setup steps<br>1. Create a Header Auth credential named **ScraperCity API Key** -- set header to `Authorization`, value to `Bearer YOUR_KEY`.<br>2. Connect a Google Sheets OAuth2 credential.<br>3. Edit **Configure Search Inputs** with your lookup targets.<br>4. Set your Google Sheet ID in **Write Results to Google Sheets**.<br>5. Click Execute workflow.<br>## Configuration<br>Enter comma-separated names, phones, or emails in **Configure Search Inputs**. **Build Request Body** converts them into arrays for the API. |
| Build Request Body | Code | Converts comma-separated inputs into API arrays | Configure Search Inputs | Submit Skip Trace Job | ## Configuration<br>Enter comma-separated names, phones, or emails in **Configure Search Inputs**. **Build Request Body** converts them into arrays for the API. |
| Submit Skip Trace Job | HTTP Request | Creates the ScraperCity skip trace job | Build Request Body | Store Run ID | ## Submit Job<br>**Submit Skip Trace Job** POSTs to the ScraperCity People Finder endpoint. **Store Run ID** saves the returned job ID for later polling. |
| Store Run ID | Set | Saves the returned run identifier | Submit Skip Trace Job | Polling Loop | ## Submit Job<br>**Submit Skip Trace Job** POSTs to the ScraperCity People Finder endpoint. **Store Run ID** saves the returned job ID for later polling. |
| Polling Loop | Split In Batches | Loop controller for repeated polling | Store Run ID, Is Scrape Complete? | Wait 60 Seconds | ## Async Polling Loop<br>**Wait 60 Seconds** pauses between checks. **Check Scrape Status** hits the status endpoint. **Is Scrape Complete?** routes to download on success or loops back. |
| Wait 60 Seconds | Wait | Delays before each status check | Polling Loop | Check Scrape Status | ## Async Polling Loop<br>**Wait 60 Seconds** pauses between checks. **Check Scrape Status** hits the status endpoint. **Is Scrape Complete?** routes to download on success or loops back. |
| Check Scrape Status | HTTP Request | Retrieves job execution status from ScraperCity | Wait 60 Seconds | Is Scrape Complete? | ## Async Polling Loop<br>**Wait 60 Seconds** pauses between checks. **Check Scrape Status** hits the status endpoint. **Is Scrape Complete?** routes to download on success or loops back. |
| Is Scrape Complete? | If | Routes success to download and non-success back to loop | Check Scrape Status | Download Results CSV, Polling Loop | ## Async Polling Loop<br>**Wait 60 Seconds** pauses between checks. **Check Scrape Status** hits the status endpoint. **Is Scrape Complete?** routes to download on success or loops back. |
| Download Results CSV | HTTP Request | Downloads the CSV output file | Is Scrape Complete? | Parse and Format CSV Results | ## Async Polling Loop<br>**Wait 60 Seconds** pauses between checks. **Check Scrape Status** hits the status endpoint. **Is Scrape Complete?** routes to download on success or loops back. |
| Parse and Format CSV Results | Code | Parses CSV text into structured contact items | Download Results CSV | Remove Duplicate Contacts | ## Download and Output<br>**Download Results CSV** fetches the file. **Parse and Format CSV Results** splits into records. **Remove Duplicate Contacts** dedupes. **Write Results to Google Sheets** appends rows. |
| Remove Duplicate Contacts | Remove Duplicates | Removes duplicate contact rows | Parse and Format CSV Results | Write Results to Google Sheets | ## Download and Output<br>**Download Results CSV** fetches the file. **Parse and Format CSV Results** splits into records. **Remove Duplicate Contacts** dedupes. **Write Results to Google Sheets** appends rows. |
| Write Results to Google Sheets | Google Sheets | Saves final records into a spreadsheet | Remove Duplicate Contacts |  | ## Download and Output<br>**Download Results CSV** fetches the file. **Parse and Format CSV Results** splits into records. **Remove Duplicate Contacts** dedupes. **Write Results to Google Sheets** appends rows. |
| Overview | Sticky Note | Workspace documentation / setup guidance |  |  |  |
| Section - Configuration | Sticky Note | Workspace annotation for input preparation block |  |  |  |
| Section - Submit Job | Sticky Note | Workspace annotation for submission block |  |  |  |
| Section - Async Polling Loop | Sticky Note | Workspace annotation for polling block |  |  |  |
| Section - Download and Output | Sticky Note | Workspace annotation for result handling block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**
   - Type: **Manual Trigger**
   - Name it: **When clicking 'Execute workflow'**

3. **Add a Set node**
   - Type: **Set**
   - Name it: **Configure Search Inputs**
   - Create these fields:
     - `names` as String, example: `John Smith,Jane Doe`
     - `phones` as String, example: leave blank if unused
     - `emails` as String, example: leave blank if unused
     - `max_results` as Number, set to `1`
   - Connect:
     - **When clicking 'Execute workflow'** → **Configure Search Inputs**

4. **Add a Code node**
   - Type: **Code**
   - Name it: **Build Request Body**
   - Paste logic that:
     - Splits comma-separated strings into arrays
     - Trims whitespace
     - Removes empty entries
     - Returns a single JSON object with:
       - `name`
       - `email`
       - `phone_number`
       - `street_citystatezip` as `[]`
       - `max_results`
   - Equivalent behavior:
     - Read from the Set node output
     - Convert `names`, `emails`, `phones`
     - Default `max_results` to `1` if missing
   - Connect:
     - **Configure Search Inputs** → **Build Request Body**

5. **Create the ScraperCity authentication credential**
   - Credential type: **Header Auth**
   - Suggested name: **ScraperCity API Key**
   - Header name: `Authorization`
   - Header value: `Bearer YOUR_KEY`
   - Replace `YOUR_KEY` with the real ScraperCity API key.

6. **Add an HTTP Request node**
   - Type: **HTTP Request**
   - Name it: **Submit Skip Trace Job**
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **Header Auth**
   - Select the **ScraperCity API Key** credential
   - Send Body: enabled
   - Body Content Type / Specify Body: JSON
   - JSON body expression: `{{ JSON.stringify($json) }}`
   - Connect:
     - **Build Request Body** → **Submit Skip Trace Job**

7. **Add a Set node**
   - Type: **Set**
   - Name it: **Store Run ID**
   - Add field:
     - `runId` as String
     - Value: `{{ $json.runId }}`
   - Connect:
     - **Submit Skip Trace Job** → **Store Run ID**

8. **Add a Split In Batches node**
   - Type: **Split In Batches**
   - Name it: **Polling Loop**
   - Batch Size: `1`
   - Reset: `false`
   - Use it purely as a loop controller
   - Connect:
     - **Store Run ID** → **Polling Loop**

9. **Add a Wait node**
   - Type: **Wait**
   - Name it: **Wait 60 Seconds**
   - Wait amount: `60` seconds
   - Connect:
     - **Polling Loop** → **Wait 60 Seconds**

10. **Add an HTTP Request node**
    - Type: **HTTP Request**
    - Name it: **Check Scrape Status**
    - Method: `GET`
    - URL expression:  
      `https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
    - Authentication: **Generic Credential Type**
    - Generic Auth Type: **Header Auth**
    - Use the same **ScraperCity API Key** credential
    - Connect:
      - **Wait 60 Seconds** → **Check Scrape Status**

11. **Add an If node**
    - Type: **If**
    - Name it: **Is Scrape Complete?**
    - Condition:
      - Left value: `{{ $json.status }}`
      - Operator: equals
      - Right value: `SUCCEEDED`
    - Keep case-sensitive comparison enabled
    - Connect:
      - **Check Scrape Status** → **Is Scrape Complete?**

12. **Create the polling loopback**
    - Connect the **false** output of **Is Scrape Complete?** back to **Polling Loop**
    - Connect the **true** output later to the download step
    - Note: this design keeps polling every 60 seconds until the status becomes exactly `SUCCEEDED`

13. **Add an HTTP Request node**
    - Type: **HTTP Request**
    - Name it: **Download Results CSV**
    - Method: `GET`
    - URL expression:  
      `https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
    - Authentication: **Generic Credential Type**
    - Generic Auth Type: **Header Auth**
    - Use the same **ScraperCity API Key** credential
    - Response format: **Text**
    - Connect:
      - **Is Scrape Complete?** true output → **Download Results CSV**

14. **Add a Code node**
    - Type: **Code**
    - Name it: **Parse and Format CSV Results**
    - Implement logic that:
      - Reads the text response from `json.data` or `json.body`
      - Returns an error item if no CSV text exists
      - Splits the CSV into lines
      - Parses headers and data rows
      - Builds a JSON object for each record
      - Deduplicates using `name` + `primary_phone`, falling back to `phone`
      - Returns at least one error item if parsing yields nothing
    - Connect:
      - **Download Results CSV** → **Parse and Format CSV Results**

15. **Add a Remove Duplicates node**
    - Type: **Remove Duplicates**
    - Name it: **Remove Duplicate Contacts**
    - Compare mode: **Selected Fields**
    - Fields to compare:
      - `name`
      - `primary_phone`
    - Connect:
      - **Parse and Format CSV Results** → **Remove Duplicate Contacts**

16. **Create a Google Sheets OAuth2 credential**
    - Credential type: **Google Sheets OAuth2 API**
    - Authenticate against the Google account that owns or can edit the target spreadsheet

17. **Prepare the destination spreadsheet**
    - Create or choose a Google Sheet
    - Add column headers matching expected CSV keys where possible, such as:
      - `name`
      - `primary_phone`
      - plus any additional fields returned by ScraperCity
    - Ensure the target tab exists

18. **Add a Google Sheets node**
    - Type: **Google Sheets**
    - Name it: **Write Results to Google Sheets**
    - Credential: your Google Sheets OAuth2 credential
    - Operation: **Append or Update**
    - Mapping mode: **Auto-map input data**
    - Set:
      - **Document ID** to your spreadsheet ID
      - **Sheet Name** or sheet identifier to the correct tab
    - Connect:
      - **Remove Duplicate Contacts** → **Write Results to Google Sheets**

19. **Optional but recommended hardening**
    - Add a maximum retry counter to avoid endless polling
    - Add explicit handling for `FAILED`, `CANCELLED`, or unknown statuses
    - Add an error filter before Google Sheets so error objects are not written as rows
    - Normalize phone number formatting before duplicate removal

20. **Add sticky notes if desired**
    - One overview note with setup instructions
    - One note for configuration
    - One note for submission
    - One note for polling
    - One note for download/output

21. **Test the workflow**
    - Enter real names, phones, or emails in **Configure Search Inputs**
    - Run the workflow manually
    - Confirm:
      - A `runId` is returned
      - Status polling works
      - CSV downloads after success
      - Parsed rows appear in Google Sheets

### Expected Inputs and Outputs
- **Workflow input:** manual execution plus configured values in the Set node
- **External API input:** JSON payload with arrays for `name`, `email`, `phone_number`
- **Intermediate key output:** `runId`
- **Final output:** one or more rows appended/updated in Google Sheets

### Sub-workflow Setup
This workflow contains **no sub-workflows** and does not invoke any external n8n workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How it works: Enter names, phone numbers, or emails in the Configure Search Inputs node; the workflow submits a skip trace job to the ScraperCity People Finder API; it polls for completion every 60 seconds (jobs take 10–60 min); once done, it downloads the CSV, parses each contact, deduplicates, and writes rows to Google Sheets. | Workspace overview note |
| Setup steps: Create a Header Auth credential named **ScraperCity API Key** with header `Authorization` and value `Bearer YOUR_KEY`; connect a Google Sheets OAuth2 credential; edit **Configure Search Inputs**; set your Google Sheet ID in **Write Results to Google Sheets**; click Execute workflow. | Workspace setup guidance |
| Configuration: Enter comma-separated names, phones, or emails in **Configure Search Inputs**. **Build Request Body** converts them into arrays for the API. | Configuration block note |
| Submit Job: **Submit Skip Trace Job** POSTs to the ScraperCity People Finder endpoint. **Store Run ID** saves the returned job ID for later polling. | Submission block note |
| Async Polling Loop: **Wait 60 Seconds** pauses between checks. **Check Scrape Status** hits the status endpoint. **Is Scrape Complete?** routes to download on success or loops back. | Polling block note |
| Download and Output: **Download Results CSV** fetches the file. **Parse and Format CSV Results** splits into records. **Remove Duplicate Contacts** dedupes. **Write Results to Google Sheets** appends rows. | Output block note |