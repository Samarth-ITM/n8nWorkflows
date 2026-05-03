Look up contact details and addresses from names using ScraperCity

https://n8nworkflows.xyz/workflows/look-up-contact-details-and-addresses-from-names-using-scrapercity-13929


# Look up contact details and addresses from names using ScraperCity

# 1. Workflow Overview

This workflow performs an asynchronous people-search lookup using the ScraperCity People Finder API. It accepts one or more search inputs for a person—name, phone number, and/or email—submits a lookup job, polls the API until the job completes, downloads the resulting CSV file, parses it into structured records, removes duplicates, and appends the final rows to Google Sheets.

Typical use cases include:
- Enriching a lead or contact from limited known details
- Finding likely addresses and contact details from a person’s name
- Storing lookup results for manual review or later processing in a spreadsheet

The workflow is organized into four logical blocks.

## 1.1 Input Configuration

The workflow begins with a manual trigger and a Set node used to define the search criteria. This is the operator-controlled entry point.

## 1.2 ScraperCity Job Submission

The configured search inputs are sent to ScraperCity’s `/scrape/people-finder` endpoint. The returned `runId` is extracted and stored for later status checks and final download.

## 1.3 Asynchronous Polling Loop

Because ScraperCity jobs are not immediate, the workflow waits before the first status check, then repeatedly polls the status endpoint. If the scrape is complete, execution continues; otherwise it waits 60 seconds and loops again. A Split In Batches node limits the maximum polling iterations.

## 1.4 Result Download, Parsing, Deduplication, and Storage

Once the scrape succeeds, the workflow downloads the result file, parses CSV text into JSON records using a Code node, removes duplicates based on name and address, and appends the final records to Google Sheets.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Configuration

### Overview
This block defines how the workflow starts and where the user enters the search criteria. It is designed for manual execution and easy modification before each run.

### Nodes Involved
- When clicking 'Execute workflow'
- Configure Search Inputs

### Node Details

#### 1. When clicking 'Execute workflow'
- **Type and role:** `n8n-nodes-base.manualTrigger`; manual entry point for ad hoc execution.
- **Configuration choices:** No parameters are configured. It simply starts the workflow when the user clicks Execute.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Configure Search Inputs`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No runtime failure is likely here.
  - Only usable in manual or test-style executions, not as an autonomous production trigger.
- **Sub-workflow reference:** None.

#### 2. Configure Search Inputs
- **Type and role:** `n8n-nodes-base.set`; defines the lookup parameters for the person search.
- **Configuration choices:**
  - Creates four fields:
    - `searchName` = `"Jane Doe"`
    - `searchPhone` = `""`
    - `searchEmail` = `""`
    - `maxResults` = `3`
  - This node acts as the main configurable input surface for the workflow.
- **Key expressions or variables used:**
  - Downstream nodes use:
    - `$json.searchName`
    - `$json.searchPhone`
    - `$json.searchEmail`
    - `$json.maxResults`
- **Input and output connections:**
  - Input: `When clicking 'Execute workflow'`
  - Output: `Start People Finder Scrape`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If all three search fields are empty, the API request may still be sent but may return no results or an error depending on ScraperCity validation.
  - `maxResults` should be a reasonable positive number; extreme values may cause API rejection or unnecessary processing time.
  - Inputs are single strings, but the downstream JSON body wraps non-empty values into one-element arrays.
- **Sub-workflow reference:** None.

---

## Block 2 — ScraperCity Job Submission

### Overview
This block sends the search request to ScraperCity and stores the returned job identifier. The `runId` becomes the central reference for all subsequent polling and result retrieval.

### Nodes Involved
- Start People Finder Scrape
- Store Run ID

### Node Details

#### 3. Start People Finder Scrape
- **Type and role:** `n8n-nodes-base.httpRequest`; submits a people-finder scrape job to ScraperCity.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
  - Body format: JSON
  - Authentication: Generic credential type using HTTP Header Auth
  - Credential expected: `ScraperCity API Key`
- **Key expressions or variables used:**
  - Request body is dynamically built from Set node output:
    - `name`: one-element JSON array if `searchName` exists, otherwise `[]`
    - `email`: one-element JSON array if `searchEmail` exists, otherwise `[]`
    - `phone_number`: one-element JSON array if `searchPhone` exists, otherwise `[]`
    - `street_citystatezip`: always `[]`
    - `max_results`: `$json.maxResults`
  - Expressions:
    - `{{ $json.searchName ? '["' + $json.searchName + '"]' : '[]' }}`
    - `{{ $json.searchEmail ? '["' + $json.searchEmail + '"]' : '[]' }}`
    - `{{ $json.searchPhone ? '["' + $json.searchPhone + '"]' : '[]' }}`
    - `{{ $json.maxResults }}`
- **Input and output connections:**
  - Input: `Configure Search Inputs`
  - Output: `Store Run ID`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid HTTP Header Auth credential will cause authentication failure.
  - If the body expression renders malformed JSON, the request will fail before or during submission.
  - Special characters such as quotes in the search values may break the string-built JSON body because values are concatenated directly rather than encoded safely.
  - API rate limits, invalid input, or service outages can cause non-2xx responses.
  - If ScraperCity changes field names or endpoint behavior, downstream nodes may break.
- **Sub-workflow reference:** None.

#### 4. Store Run ID
- **Type and role:** `n8n-nodes-base.set`; extracts and preserves the `runId` returned by ScraperCity.
- **Configuration choices:**
  - Assigns:
    - `runId` = `{{ $json.runId }}`
- **Key expressions or variables used:**
  - `={{ $json.runId }}`
- **Input and output connections:**
  - Input: `Start People Finder Scrape`
  - Output: `Wait Before First Status Check`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If the API response does not include `runId`, later nodes will generate invalid URLs.
  - If ScraperCity returns an error payload instead of the expected structure, this node may silently set an empty or undefined `runId`.
- **Sub-workflow reference:** None.

---

## Block 3 — Asynchronous Polling Loop

### Overview
This block handles the long-running asynchronous nature of the scrape job. It delays the first status check, then polls repeatedly until the status becomes `SUCCEEDED`, with a 60-second wait between attempts and a maximum loop count enforced by Split In Batches.

### Nodes Involved
- Wait Before First Status Check
- Poll Loop
- Check Scrape Status
- Is Scrape Complete?
- Wait 60 Seconds Before Retry

### Node Details

#### 5. Wait Before First Status Check
- **Type and role:** `n8n-nodes-base.wait`; pauses the workflow before the initial status poll.
- **Configuration choices:**
  - Wait amount: `30`
  - No explicit unit is shown in the JSON payload excerpt; in practice this node is intended as an initial delay before the first check.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Store Run ID`
  - Output: `Poll Loop`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If n8n is not configured to support wait/resume correctly, delayed execution may not resume as expected.
  - Long waits rely on the instance’s persistence and execution-resume configuration.
- **Sub-workflow reference:** None.

#### 6. Poll Loop
- **Type and role:** `n8n-nodes-base.splitInBatches`; used here as an iteration controller rather than for classic item batching.
- **Configuration choices:**
  - `maxIterations`: `30`
  - It feeds the current item into the poll-check branch and accepts the retry branch back into itself.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Inputs:
    - `Wait Before First Status Check`
    - `Wait 60 Seconds Before Retry`
  - Outputs:
    - Main output 0: `Check Scrape Status`
    - Output 1: unused
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If the scrape never reaches `SUCCEEDED` within 30 iterations, the loop will stop without a dedicated timeout-handling branch.
  - Because no explicit failed/completed timeout branch exists, the workflow may end silently after iteration exhaustion.
  - This pattern assumes one item only; multiple items could produce more complex iteration behavior.
- **Sub-workflow reference:** None.

#### 7. Check Scrape Status
- **Type and role:** `n8n-nodes-base.httpRequest`; checks the status of the submitted scrape job.
- **Configuration choices:**
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
  - Authentication: Generic HTTP Header Auth
  - Credential expected: `ScraperCity API Key`
- **Key expressions or variables used:**
  - URL expression: `=https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
- **Input and output connections:**
  - Input: `Poll Loop`
  - Output: `Is Scrape Complete?`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid `runId` causes invalid endpoint requests.
  - Auth failures, API downtime, or rate limiting can interrupt the polling loop.
  - If the API returns an unexpected structure, the IF node may not evaluate as intended.
- **Sub-workflow reference:** None.

#### 8. Is Scrape Complete?
- **Type and role:** `n8n-nodes-base.if`; routes execution based on whether the scrape has completed successfully.
- **Configuration choices:**
  - Condition checks whether `$json.status` equals `"SUCCEEDED"`.
  - Strict type validation and case-sensitive comparison are enabled.
- **Key expressions or variables used:**
  - Left value: `={{ $json.status }}`
  - Right value: `SUCCEEDED`
- **Input and output connections:**
  - Input: `Check Scrape Status`
  - Output 0 (true): `Download Results`
  - Output 1 (false): `Wait 60 Seconds Before Retry`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Only `SUCCEEDED` is treated as terminal success.
  - Other statuses such as `FAILED`, `CANCELLED`, or unexpected values are all treated as “not complete” and retried, which may be undesirable.
  - If `status` is missing or null, the node will route to retry.
- **Sub-workflow reference:** None.

#### 9. Wait 60 Seconds Before Retry
- **Type and role:** `n8n-nodes-base.wait`; delays the next poll attempt.
- **Configuration choices:**
  - Wait amount: `60`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Is Scrape Complete?` false branch
  - Output: `Poll Loop`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Same persistence/resume considerations as the first Wait node.
  - Large numbers of delayed executions can affect queue/storage load on busy n8n instances.
- **Sub-workflow reference:** None.

---

## Block 4 — Result Download, Parsing, Deduplication, and Storage

### Overview
After the scrape completes successfully, this block retrieves the CSV export, converts it to structured JSON rows, removes duplicate contacts using name and address, and appends the remaining records to a Google Sheet.

### Nodes Involved
- Download Results
- Parse CSV Results
- Remove Duplicate Contacts
- Save Results to Google Sheets

### Node Details

#### 10. Download Results
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches the completed scrape output.
- **Configuration choices:**
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
  - Authentication: Generic HTTP Header Auth
  - Credential expected: `ScraperCity API Key`
- **Key expressions or variables used:**
  - URL expression: `=https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
- **Input and output connections:**
  - Input: `Is Scrape Complete?` true branch
  - Output: `Parse CSV Results`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid or expired `runId` may return 404 or similar errors.
  - API may return raw text, JSON with a `data` field, or another response shape; the Code node attempts to tolerate several possibilities.
  - If the result file is very large, memory usage may become significant.
- **Sub-workflow reference:** None.

#### 11. Parse CSV Results
- **Type and role:** `n8n-nodes-base.code`; parses CSV text into one item per row.
- **Configuration choices:**
  - JavaScript code manually:
    - Reads raw content from `items[0].json.data`, or `items[0].json.body`, or `items[0].json`
    - Converts the payload to a string if needed
    - Rejects empty CSV
    - Splits into lines
    - Uses a custom `parseCsvLine()` function to support quoted values and escaped double quotes
    - Uses the first row as headers
    - Builds one JSON object per data row
- **Key expressions or variables used:**
  - Internal logic references:
    - `items[0].json.data`
    - `items[0].json.body`
    - `items[0].json`
  - Output rows are plain objects using CSV headers as keys.
- **Input and output connections:**
  - Input: `Download Results`
  - Output: `Remove Duplicate Contacts`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the response is not valid CSV text, parsing may produce invalid rows.
  - Splitting on `\n` alone may leave trailing `\r` characters in some environments.
  - Multiline CSV fields are not handled correctly because the parser splits the entire file on line breaks before parsing quotes.
  - If there is only a header row and no data rows, it returns an error item instead of zero rows.
  - If column counts vary per row, missing values become empty strings.
- **Sub-workflow reference:** None.

#### 12. Remove Duplicate Contacts
- **Type and role:** `n8n-nodes-base.removeDuplicates`; removes duplicate result rows before persistence.
- **Configuration choices:**
  - Compares records using:
    - `full_name`
    - `address`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Parse CSV Results`
  - Output: `Save Results to Google Sheets`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the CSV headers do not contain `full_name` or `address`, deduplication may be ineffective or based on empty values.
  - Slight differences in formatting, casing, or whitespace can prevent expected deduplication.
  - Different people at the same address with the same name representation may be collapsed unintentionally.
- **Sub-workflow reference:** None.

#### 13. Save Results to Google Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`; appends the final contact rows into a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Mapping mode: automatic mapping from input data
  - Google Sheet document ID: left empty in the provided workflow and must be configured
  - Sheet name / sheet ID: left empty in the provided workflow and must be configured
  - Credential: Google Sheets OAuth2
- **Key expressions or variables used:**
  - No custom expressions beyond unresolved resource locator placeholders.
- **Input and output connections:**
  - Input: `Remove Duplicate Contacts`
  - Output: none
- **Version-specific requirements:** Type version `4.6`.
- **Edge cases or potential failure types:**
  - The node is incomplete as provided; both target spreadsheet and target sheet must be selected.
  - Missing or invalid Google OAuth2 credentials will cause auth failure.
  - Auto-mapping expects the destination sheet headers to align reasonably with incoming field names.
  - Google API quotas, permission issues, or sheet structure changes may cause failures.
  - If the previous node outputs an error item rather than normal records, that item may also be appended unless additional filtering is added.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual start of the workflow |  | Configure Search Inputs | ## Configuration<br>Set your search target in **Configure Search Inputs** -- name, phone, or email. Add your ScraperCity API key credential to the HTTP nodes. |
| Configure Search Inputs | Set | Defines search criteria and result limit | When clicking 'Execute workflow' | Start People Finder Scrape | ## How it works<br>1. Enter a name, phone, or email in the Configure Search Inputs node.<br>2. The workflow submits a people-finder job to the ScraperCity API.<br>3. It polls for completion every 60 seconds (jobs take 2-20 min).<br>4. Results are downloaded, parsed from CSV, deduped, and saved to Google Sheets.<br><br>## Setup steps<br>1. Create a Header Auth credential named **ScraperCity API Key** -- set header to `Authorization`, value to `Bearer YOUR_KEY`.<br>2. Connect a Google Sheets OAuth2 credential.<br>3. Edit **Configure Search Inputs** with your target person.<br>4. Set your Google Sheet ID in **Save Results to Google Sheets**.<br>5. Click Execute workflow.<br><br>## Configuration<br>Set your search target in **Configure Search Inputs** -- name, phone, or email. Add your ScraperCity API key credential to the HTTP nodes. |
| Start People Finder Scrape | HTTP Request | Submits the ScraperCity people-finder job | Configure Search Inputs | Store Run ID | ## Submit Scrape Job<br>**Start People Finder Scrape** POSTs to the API and returns a job ID. **Store Run ID** saves it for polling and download references. |
| Store Run ID | Set | Extracts and stores the ScraperCity run ID | Start People Finder Scrape | Wait Before First Status Check | ## Submit Scrape Job<br>**Start People Finder Scrape** POSTs to the API and returns a job ID. **Store Run ID** saves it for polling and download references. |
| Wait Before First Status Check | Wait | Initial delay before polling begins | Store Run ID | Poll Loop |  |
| Poll Loop | Split In Batches | Controls repeated polling iterations | Wait Before First Status Check; Wait 60 Seconds Before Retry | Check Scrape Status | ## Async Polling Loop<br>**Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Wait 60 Seconds Before Retry** loops back through the poll. |
| Check Scrape Status | HTTP Request | Polls ScraperCity for job status | Poll Loop | Is Scrape Complete? | ## Async Polling Loop<br>**Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Wait 60 Seconds Before Retry** loops back through the poll. |
| Is Scrape Complete? | IF | Branches on ScraperCity job completion status | Check Scrape Status | Download Results; Wait 60 Seconds Before Retry | ## Async Polling Loop<br>**Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Wait 60 Seconds Before Retry** loops back through the poll. |
| Wait 60 Seconds Before Retry | Wait | Delay between polling attempts | Is Scrape Complete? | Poll Loop | ## Async Polling Loop<br>**Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Wait 60 Seconds Before Retry** loops back through the poll. |
| Download Results | HTTP Request | Downloads completed CSV results | Is Scrape Complete? | Parse CSV Results | ## Parse and Save Results<br>**Download Results** fetches the CSV. **Parse CSV Results** converts to JSON records. **Remove Duplicate Contacts** dedupes. **Save Results to Google Sheets** writes each row. |
| Parse CSV Results | Code | Parses CSV text into JSON rows | Download Results | Remove Duplicate Contacts | ## Parse and Save Results<br>**Download Results** fetches the CSV. **Parse CSV Results** converts to JSON records. **Remove Duplicate Contacts** dedupes. **Save Results to Google Sheets** writes each row. |
| Remove Duplicate Contacts | Remove Duplicates | Deduplicates contacts by full name and address | Parse CSV Results | Save Results to Google Sheets | ## Parse and Save Results<br>**Download Results** fetches the CSV. **Parse CSV Results** converts to JSON records. **Remove Duplicate Contacts** dedupes. **Save Results to Google Sheets** writes each row. |
| Save Results to Google Sheets | Google Sheets | Appends final results to a spreadsheet | Remove Duplicate Contacts |  | ## How it works<br>1. Enter a name, phone, or email in the Configure Search Inputs node.<br>2. The workflow submits a people-finder job to the ScraperCity API.<br>3. It polls for completion every 60 seconds (jobs take 2-20 min).<br>4. Results are downloaded, parsed from CSV, deduped, and saved to Google Sheets.<br><br>## Setup steps<br>1. Create a Header Auth credential named **ScraperCity API Key** -- set header to `Authorization`, value to `Bearer YOUR_KEY`.<br>2. Connect a Google Sheets OAuth2 credential.<br>3. Edit **Configure Search Inputs** with your target person.<br>4. Set your Google Sheet ID in **Save Results to Google Sheets**.<br>5. Click Execute workflow.<br><br>## Parse and Save Results<br>**Download Results** fetches the CSV. **Parse CSV Results** converts to JSON records. **Remove Duplicate Contacts** dedupes. **Save Results to Google Sheets** writes each row. |
| Overview | Sticky Note | Workspace documentation |  |  |  |
| Section - Configuration | Sticky Note | Workspace documentation for input setup |  |  |  |
| Section - Submit and Store | Sticky Note | Workspace documentation for submission phase |  |  |  |
| Section - Async Polling Loop | Sticky Note | Workspace documentation for polling phase |  |  |  |
| Section - Parse and Save | Sticky Note | Workspace documentation for result handling |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Name it: **When clicking 'Execute workflow'**
   - No extra configuration is required.

3. **Add a Set node** after the trigger
   - Node type: **Set**
   - Name it: **Configure Search Inputs**
   - Add these fields:
     1. `searchName` as String, default example: `Jane Doe`
     2. `searchPhone` as String, default example: empty string
     3. `searchEmail` as String, default example: empty string
     4. `maxResults` as Number, default value: `3`
   - Connect:
     - `When clicking 'Execute workflow'` → `Configure Search Inputs`

4. **Create the ScraperCity credential**
   - Credential type: **HTTP Header Auth**
   - Credential name: **ScraperCity API Key**
   - Header name: `Authorization`
   - Header value: `Bearer YOUR_KEY`
   - Use the exact credential on all ScraperCity HTTP Request nodes.

5. **Add an HTTP Request node** to submit the scrape job
   - Node type: **HTTP Request**
   - Name it: **Start People Finder Scrape**
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **HTTP Header Auth**
   - Select credential: **ScraperCity API Key**
   - Enable body sending
   - Specify body as **JSON**
   - Use this expression-based JSON body logic:
     - `name`: one-element array when `searchName` is non-empty, else empty array
     - `email`: one-element array when `searchEmail` is non-empty, else empty array
     - `phone_number`: one-element array when `searchPhone` is non-empty, else empty array
     - `street_citystatezip`: empty array
     - `max_results`: value from `maxResults`
   - Recreate the same behavior as:
     - name from `$json.searchName`
     - email from `$json.searchEmail`
     - phone from `$json.searchPhone`
     - max results from `$json.maxResults`
   - Connect:
     - `Configure Search Inputs` → `Start People Finder Scrape`

6. **Add a Set node** to preserve the run ID
   - Node type: **Set**
   - Name it: **Store Run ID**
   - Add one field:
     - `runId` as String
     - Value expression: `{{ $json.runId }}`
   - Connect:
     - `Start People Finder Scrape` → `Store Run ID`

7. **Add a Wait node** for the first delay
   - Node type: **Wait**
   - Name it: **Wait Before First Status Check**
   - Configure it to wait **30 seconds**
   - Connect:
     - `Store Run ID` → `Wait Before First Status Check`

8. **Add a Split In Batches node** to control polling iterations
   - Node type: **Split In Batches**
   - Name it: **Poll Loop**
   - In options, set **Max Iterations** to `30`
   - Connect:
     - `Wait Before First Status Check` → `Poll Loop`

9. **Add an HTTP Request node** to check job status
   - Node type: **HTTP Request**
   - Name it: **Check Scrape Status**
   - Method: `GET`
   - URL expression:
     - `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **HTTP Header Auth**
   - Credential: **ScraperCity API Key**
   - Connect:
     - `Poll Loop` main output → `Check Scrape Status`

10. **Add an IF node** to test completion
    - Node type: **IF**
    - Name it: **Is Scrape Complete?**
    - Condition:
      - Left value: `{{ $json.status }}`
      - Operator: equals
      - Right value: `SUCCEEDED`
    - Use strict/case-sensitive comparison as in the original workflow.
    - Connect:
      - `Check Scrape Status` → `Is Scrape Complete?`

11. **Add a second Wait node** for retries
    - Node type: **Wait**
    - Name it: **Wait 60 Seconds Before Retry**
    - Configure it to wait **60 seconds**
    - Connect:
      - `Is Scrape Complete?` false output → `Wait 60 Seconds Before Retry`

12. **Close the loop**
    - Connect:
      - `Wait 60 Seconds Before Retry` → `Poll Loop`
    - This creates the polling cycle.
    - The Split In Batches node’s max iterations prevents infinite looping.

13. **Add an HTTP Request node** to download results
    - Node type: **HTTP Request**
    - Name it: **Download Results**
    - Method: `GET`
    - URL expression:
      - `https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
    - Authentication: **Generic Credential Type**
    - Generic Auth Type: **HTTP Header Auth**
    - Credential: **ScraperCity API Key**
    - Connect:
      - `Is Scrape Complete?` true output → `Download Results`

14. **Add a Code node** to parse the CSV
    - Node type: **Code**
    - Name it: **Parse CSV Results**
    - Language: JavaScript
    - Paste logic equivalent to the provided workflow:
      - Read the raw response from `items[0].json.data`, fallback to `items[0].json.body`, fallback to `items[0].json`
      - Convert non-string payloads to a string
      - Return an error object if the CSV text is empty
      - Split the first line into headers
      - Parse each remaining line into values with support for quoted commas and escaped quotes
      - Build one output item per row
    - Connect:
      - `Download Results` → `Parse CSV Results`

15. **Add a Remove Duplicates node**
    - Node type: **Remove Duplicates**
    - Name it: **Remove Duplicate Contacts**
    - Configure fields to compare:
      - `full_name`
      - `address`
    - Connect:
      - `Parse CSV Results` → `Remove Duplicate Contacts`

16. **Create the Google Sheets credential**
    - Credential type: **Google Sheets OAuth2 API**
    - Authenticate with a Google account that has edit access to the destination spreadsheet.

17. **Prepare the destination Google Sheet**
    - Create or choose a spreadsheet.
    - Create the target tab.
    - Ideally add header columns matching the CSV-derived field names from ScraperCity, or at least compatible columns for auto-mapping.
    - Common fields may include name, address, phone, email, or similar depending on the API export structure.

18. **Add a Google Sheets node**
    - Node type: **Google Sheets**
    - Name it: **Save Results to Google Sheets**
    - Operation: **Append**
    - Mapping mode: **Auto-map input data**
    - Select the spreadsheet document ID
    - Select the target sheet/tab
    - Credential: your Google Sheets OAuth2 credential
    - Connect:
      - `Remove Duplicate Contacts` → `Save Results to Google Sheets`

19. **Optional but recommended: add sticky notes**
    - Add one overview note describing the full process.
    - Add section notes for:
      - Configuration
      - Submit and Store
      - Async Polling Loop
      - Parse and Save

20. **Test the workflow**
    - Set at least one search field in **Configure Search Inputs**
    - Verify the ScraperCity credential works
    - Verify the Google Sheets node points to a writable spreadsheet
    - Run the workflow manually
    - Expect job completion to take roughly 2–20 minutes based on the sticky note guidance

21. **Recommended improvements when rebuilding**
    - Add validation to ensure at least one of name, phone, or email is provided.
    - Add explicit handling for failed statuses like `FAILED` or `CANCELLED`.
    - Add a timeout/failure branch if max polling iterations are exceeded.
    - Use safer JSON construction in the submission body to avoid malformed JSON when values contain quotes.
    - Filter out parser error rows before writing to Google Sheets.

### Credential Summary
- **ScraperCity API Key**
  - Type: HTTP Header Auth
  - Header: `Authorization`
  - Value: `Bearer YOUR_KEY`

- **Google Sheets OAuth2**
  - Type: Google Sheets OAuth2 API
  - Must have append/write permission on the selected spreadsheet

### Sub-workflow Setup
- This workflow does **not** invoke any sub-workflows.
- It has a single entry point: **When clicking 'Execute workflow'**.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Jobs are expected to complete asynchronously and may take approximately 2–20 minutes. | Operational note from workspace documentation |
| Before use, configure a Header Auth credential named **ScraperCity API Key** with `Authorization: Bearer YOUR_KEY`. | ScraperCity authentication setup |
| Before use, connect a Google Sheets OAuth2 credential and set the destination spreadsheet in **Save Results to Google Sheets**. | Google Sheets setup |
| User-editable search criteria are centralized in **Configure Search Inputs**. | Workflow operation |
| The workflow description area is empty in the provided export. | Metadata note |