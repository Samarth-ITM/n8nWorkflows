Skip trace property owners with ScraperCity and sync contacts to Airtable

https://n8nworkflows.xyz/workflows/skip-trace-property-owners-with-scrapercity-and-sync-contacts-to-airtable-14078


# Skip trace property owners with ScraperCity and sync contacts to Airtable

# 1. Workflow Overview

This workflow performs a skip-trace style people search for a property owner using ScraperCity, waits for the asynchronous job to finish, downloads the resulting contact data, normalizes and filters the records, then creates contact rows in Airtable.

Typical use cases include:
- Finding updated phone/email details for a property owner
- Enriching owner data before outreach
- Sending cleaned contact records into an Airtable-based lead or CRM table

## 1.1 Input Reception and Search Configuration

The workflow starts manually and uses a Set node to define the owner search parameters:
- `ownerName`
- `ownerPhone`
- `ownerEmail`
- `streetCityStateZip`
- `maxResults`

This block is the main user-editable entry area.

## 1.2 ScraperCity Job Submission

The configured search values are sent to ScraperCity’s People Finder API. The API responds with a `runId`, which is stored for later polling.

## 1.3 Asynchronous Polling Loop

Because the ScraperCity search runs asynchronously, the workflow waits 60 seconds, checks the job status, and loops until the status becomes `SUCCEEDED`.

## 1.4 Result Download and Data Normalization

Once complete, the workflow downloads the trace results and parses them into a standard schema. The parsing logic supports multiple possible response shapes, including JSON arrays, wrapped JSON objects, plain strings, and simple CSV.

## 1.5 Deduplication, Contact Filtering, and Airtable Sync

The parsed contacts are deduplicated by name and phone, filtered to keep only rows that contain at least one usable contact field (phone or email), and then inserted into Airtable.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Search Configuration

### Overview
This block initializes the workflow and defines the search criteria sent to ScraperCity. It is designed so the operator only needs to edit a single node before execution.

### Nodes Involved
- When clicking 'Execute workflow'
- Configure Search Inputs

### Node Details

#### 1. When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for test or ad hoc runs.
- **Configuration choices:** No special parameters; it simply starts the workflow when executed from the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Configure Search Inputs`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No runtime failure expected.
  - Only usable for manual execution, not automated scheduling or webhook-based ingestion.
- **Sub-workflow reference:** None.

#### 2. Configure Search Inputs
- **Type and technical role:** `n8n-nodes-base.set`; defines the owner lookup payload fields.
- **Configuration choices:**
  - Sets:
    - `ownerName` = `John Smith`
    - `ownerPhone` = empty string
    - `ownerEmail` = empty string
    - `streetCityStateZip` = empty string
    - `maxResults` = `3`
  - Acts as the editable configuration surface for the search.
- **Key expressions or variables used:** Static values only in this export.
- **Input and output connections:**
  - Input: `When clicking 'Execute workflow'`
  - Output: `Submit Skip Trace Job`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If `ownerName`, `ownerPhone`, `ownerEmail`, and `streetCityStateZip` are all blank, the downstream API request may be rejected or return no useful results.
  - `maxResults` should remain a sensible integer; invalid values may produce API-side validation issues.
- **Sub-workflow reference:** None.

---

## Block 2 — ScraperCity Job Submission

### Overview
This block sends the people-finder request to ScraperCity and captures the returned asynchronous job ID. That `runId` is essential for polling status and later downloading the completed results.

### Nodes Involved
- Submit Skip Trace Job
- Store Run ID

### Node Details

#### 3. Submit Skip Trace Job
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a POST request to the ScraperCity People Finder endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
  - Body type: JSON
  - Authentication: generic credential using HTTP Header Auth
  - Credential expected: `ScraperCity API Key`
- **Key expressions or variables used:**
  - `name`: includes `ownerName` as a one-element JSON array string if present, otherwise `[]`
  - `email`: includes `ownerEmail` similarly
  - `phone_number`: includes `ownerPhone` similarly
  - `street_citystatezip`: includes `streetCityStateZip` similarly
  - `max_results`: from `{{$json.maxResults}}`
- **Interpretation of body behavior:**
  - The node dynamically builds a JSON payload where each search field is represented as an array.
  - Empty values are converted into empty arrays.
- **Input and output connections:**
  - Input: `Configure Search Inputs`
  - Output: `Store Run ID`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Authentication failure if the HTTP Header Auth credential is missing or invalid.
  - API validation errors if all search fields are empty.
  - Possible malformed JSON risk because the body is manually assembled with expressions rather than built from native structured fields.
  - Timeout or rate limiting from ScraperCity.
- **Sub-workflow reference:** None.

#### 4. Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`; preserves the returned job identifier in a clean field.
- **Configuration choices:**
  - Creates `runId` from `{{$json.runId}}`
- **Key expressions or variables used:**
  - `={{ $json.runId }}`
- **Input and output connections:**
  - Input: `Submit Skip Trace Job`
  - Output: `Wait 60 Seconds Before Poll`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If ScraperCity returns a different response shape or no `runId`, the polling loop will fail downstream.
- **Sub-workflow reference:** None.

---

## Block 3 — Asynchronous Polling Loop

### Overview
This block handles the long-running asynchronous nature of the ScraperCity job. It waits, checks the job status, and repeats until success.

### Nodes Involved
- Wait 60 Seconds Before Poll
- Check Scrape Status
- Is Scrape Complete?
- Pass Run ID to Retry
- Wait 60 Seconds Before Retry

### Node Details

#### 5. Wait 60 Seconds Before Poll
- **Type and technical role:** `n8n-nodes-base.wait`; pauses workflow execution before the first status check.
- **Configuration choices:**
  - Wait amount: `60`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Store Run ID`
  - Output: `Check Scrape Status`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Long waits can be affected by n8n execution persistence configuration.
  - If wait/resume handling is not properly configured in the n8n environment, delayed execution may not resume as expected.
- **Sub-workflow reference:** None.

#### 6. Check Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; queries the status endpoint for the current run.
- **Configuration choices:**
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
  - Authentication: generic credential using HTTP Header Auth
- **Key expressions or variables used:**
  - URL interpolation with `{{$json.runId}}`
- **Input and output connections:**
  - Inputs:
    - `Wait 60 Seconds Before Poll`
    - `Wait 60 Seconds Before Retry`
  - Output: `Is Scrape Complete?`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid `runId`
  - Authentication failure
  - API transient errors
  - Non-success statuses not explicitly handled beyond looping
- **Sub-workflow reference:** None.

#### 7. Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`; routes execution based on whether the scrape has completed successfully.
- **Configuration choices:**
  - Condition: `{{$json.status}}` equals `SUCCEEDED`
  - Strict string comparison
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: `Check Scrape Status`
  - True output: `Download Trace Results`
  - False output: `Pass Run ID to Retry`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Jobs with terminal statuses like `FAILED`, `CANCELLED`, or unknown states will loop forever because only `SUCCEEDED` exits the loop.
  - Status field name changes would break the condition.
- **Sub-workflow reference:** None.

#### 8. Pass Run ID to Retry
- **Type and technical role:** `n8n-nodes-base.set`; reintroduces `runId` into the payload before the retry wait.
- **Configuration choices:**
  - Sets `runId` from `{{$json.runId}}`
- **Key expressions or variables used:**
  - `={{ $json.runId }}`
- **Input and output connections:**
  - Input: false branch of `Is Scrape Complete?`
  - Output: `Wait 60 Seconds Before Retry`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If the status endpoint response omits `runId`, the loop will break on the next request.
- **Sub-workflow reference:** None.

#### 9. Wait 60 Seconds Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`; delays the next polling attempt.
- **Configuration choices:**
  - Wait amount: `60`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Pass Run ID to Retry`
  - Output: `Check Scrape Status`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Same delayed-execution considerations as the first Wait node.
  - Infinite retries are possible because there is no loop counter or timeout guard.
- **Sub-workflow reference:** None.

---

## Block 4 — Result Download and Data Normalization

### Overview
After the ScraperCity job succeeds, this block downloads the raw result payload and converts it to a normalized contact structure. It is designed to tolerate different response formats from the upstream source.

### Nodes Involved
- Download Trace Results
- Parse Contact Records

### Node Details

#### 10. Download Trace Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches the completed trace output.
- **Configuration choices:**
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
  - Authentication: generic credential using HTTP Header Auth
- **Key expressions or variables used:**
  - URL interpolation with `{{$json.runId}}`
- **Input and output connections:**
  - Input: true branch of `Is Scrape Complete?`
  - Output: `Parse Contact Records`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Authentication failure
  - File or payload not yet available despite `SUCCEEDED`
  - Unexpected content type or binary response format
  - Missing `runId`
- **Sub-workflow reference:** None.

#### 11. Parse Contact Records
- **Type and technical role:** `n8n-nodes-base.code`; transforms the ScraperCity response into a standard schema for downstream processing.
- **Configuration choices:**
  - JavaScript code inspects `items[0].json`
  - Handles several input shapes:
    - already a JSON array
    - object containing `data` array
    - raw string treated as CSV
    - fallback single record
  - Produces one item per record with fields:
    - `full_name`
    - `phone`
    - `email`
    - `address`
    - `city`
    - `state`
    - `zip`
    - `age`
    - `raw_source`
- **Key expressions or variables used:**
  - Standard Code node JavaScript; no n8n expressions inside the script
  - `raw_source` stores `JSON.stringify(r)`
- **Input and output connections:**
  - Input: `Download Trace Results`
  - Output: `Remove Duplicate Contacts`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - The CSV parser is simplistic and may break on quoted commas, embedded line breaks, or irregular CSV escaping.
  - If the HTTP node returns binary rather than JSON/string, this script may not parse it correctly.
  - If the payload is an object but not shaped as expected, normalization may produce sparse fields.
  - `JSON.stringify` can fail only in unusual cases such as circular references, which are unlikely here.
- **Sub-workflow reference:** None.

---

## Block 5 — Deduplication, Filtering, and Airtable Sync

### Overview
This block cleans the normalized records, removes duplicates, keeps only actionable contacts, and creates Airtable records. It is the final persistence layer of the workflow.

### Nodes Involved
- Remove Duplicate Contacts
- Filter Records with Contact Info
- Sync Contacts to Airtable

### Node Details

#### 12. Remove Duplicate Contacts
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`; eliminates repeated contact entries.
- **Configuration choices:**
  - Compares the fields:
    - `full_name`
    - `phone`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Parse Contact Records`
  - Output: `Filter Records with Contact Info`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Records with same person but differently formatted phone numbers may not deduplicate.
  - Empty `phone` plus same `full_name` may merge records that should stay distinct.
  - Duplicate emails are not considered in deduplication logic.
- **Sub-workflow reference:** None.

#### 13. Filter Records with Contact Info
- **Type and technical role:** `n8n-nodes-base.filter`; keeps only rows that contain at least one usable contact method.
- **Configuration choices:**
  - OR logic:
    - `phone` is not empty
    - `email` is not empty
  - Loose validation, case-insensitive mode
- **Key expressions or variables used:**
  - `={{ $json.phone }}`
  - `={{ $json.email }}`
- **Input and output connections:**
  - Input: `Remove Duplicate Contacts`
  - Output: `Sync Contacts to Airtable`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Strings containing only whitespace may still behave unexpectedly depending on upstream normalization.
  - Records with valid address but no phone/email are intentionally dropped.
- **Sub-workflow reference:** None.

#### 14. Sync Contacts to Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`; creates new Airtable rows for each filtered contact.
- **Configuration choices:**
  - Operation: `create`
  - Credential: Airtable Personal Access Token
  - Base ID: not filled in this export
  - Table ID: not filled in this export
  - Field mapping:
    - `Full Name` ← `{{$json.full_name}}`
    - `Phone` ← `{{$json.phone}}`
    - `Email` ← `{{$json.email}}`
    - `Address` ← `{{$json.address}}`
    - `City` ← `{{$json.city}}`
    - `State` ← `{{$json.state}}`
    - `ZIP` ← `{{$json.zip}}`
    - `Age` ← `{{$json.age}}`
- **Key expressions or variables used:**
  - Expression mappings for all Airtable columns
- **Input and output connections:**
  - Input: `Filter Records with Contact Info`
  - Output: none
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Base ID and Table ID are blank in the provided workflow and must be configured.
  - Airtable field names must exactly match the mapped columns unless adjusted.
  - Auth failures if the PAT lacks required scopes.
  - Record creation can fail on field type mismatches or rate limits.
  - This node creates records only; it does not upsert or prevent re-insertion across separate workflow runs.
- **Sub-workflow reference:** None.

---

## Additional Non-Executable Nodes — Sticky Notes

These nodes are documentation aids in the canvas and do not affect execution, but they contain important operating context.

### 15. Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Contains a full summary of workflow behavior and setup instructions.
- **Connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases:** None.
- **Sub-workflow reference:** None.

### 16. Section - Configuration
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation for search input setup.
- **Configuration choices:** Explains that at least one of `ownerName`, `ownerPhone`, or `ownerEmail` must be set and that `maxResults` controls result count.
- **Connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases:** None.
- **Sub-workflow reference:** None.

### 17. Section - Job Submission
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documents API submission and `runId` handling.
- **Configuration choices:** Describes how submission returns a `runId` and how it is stored.
- **Connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases:** None.
- **Sub-workflow reference:** None.

### 18. Section - Async Polling Loop
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documents the retry loop.
- **Configuration choices:** Explains the wait-check-loop behavior until completion.
- **Connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases:** None.
- **Sub-workflow reference:** None.

### 19. Section - Results Processing
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documents download, parsing, deduplication, and filtering.
- **Configuration choices:** Describes the post-processing flow before Airtable sync.
- **Connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual start of the workflow |  | Configure Search Inputs | ## How it works<br>1. **Configure Search Inputs** -- set the property owner name, phone, or email you want to trace.<br>2. **Submit Skip Trace Job** posts the lookup to ScraperCity People Finder and receives a `runId`.<br>3. The polling loop waits 60 seconds, then checks job status via **Check Scrape Status** until `SUCCEEDED`.<br>4. **Download Trace Results** fetches the completed data, which **Parse Contact Records** normalises into clean fields.<br>5. Duplicates are removed and records with at least one contact detail are synced to Airtable.<br><br>## Setup steps<br>1. In n8n Credentials, create an **HTTP Header Auth** credential named `ScraperCity API Key` -- header name `Authorization`, value `Bearer YOUR_KEY`.<br>2. Open **Configure Search Inputs** and enter the owner name (or phone/email) you want to look up.<br>3. In **Sync Contacts to Airtable**, connect your Airtable credential and set the correct Base ID and Table ID.<br>4. Make sure your Airtable table has columns: Full Name, Phone, Email, Address, City, State, ZIP, Age. |
| Configure Search Inputs | Set | Defines owner lookup inputs | When clicking 'Execute workflow' | Submit Skip Trace Job | ## Configuration<br>**Configure Search Inputs** is the only node you need to edit before running. Set `ownerName`, `ownerPhone`, or `ownerEmail` -- at least one is required. Adjust `maxResults` (default 3) to control how many contact matches ScraperCity returns per person. |
| Submit Skip Trace Job | HTTP Request | Starts ScraperCity people-finder job | Configure Search Inputs | Store Run ID | ## Job Submission<br>**Submit Skip Trace Job** sends the lookup payload to the ScraperCity People Finder endpoint and receives a `runId`. **Store Run ID** saves that ID so the polling loop can reference it throughout execution. |
| Store Run ID | Set | Extracts and stores `runId` | Submit Skip Trace Job | Wait 60 Seconds Before Poll | ## Job Submission<br>**Submit Skip Trace Job** sends the lookup payload to the ScraperCity People Finder endpoint and receives a `runId`. **Store Run ID** saves that ID so the polling loop can reference it throughout execution. |
| Wait 60 Seconds Before Poll | Wait | Initial delay before status polling | Store Run ID | Check Scrape Status | ## Async Polling Loop<br>ScraperCity jobs run asynchronously and may take several minutes. **Wait 60 Seconds Before Poll** pauses execution. **Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Pass Run ID to Retry** feeds the ID into **Wait 60 Seconds Before Retry**, which loops back to the status check. |
| Check Scrape Status | HTTP Request | Polls ScraperCity job status | Wait 60 Seconds Before Poll; Wait 60 Seconds Before Retry | Is Scrape Complete? | ## Async Polling Loop<br>ScraperCity jobs run asynchronously and may take several minutes. **Wait 60 Seconds Before Poll** pauses execution. **Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Pass Run ID to Retry** feeds the ID into **Wait 60 Seconds Before Retry**, which loops back to the status check. |
| Is Scrape Complete? | If | Branches on `SUCCEEDED` status | Check Scrape Status | Download Trace Results; Pass Run ID to Retry | ## Async Polling Loop<br>ScraperCity jobs run asynchronously and may take several minutes. **Wait 60 Seconds Before Poll** pauses execution. **Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Pass Run ID to Retry** feeds the ID into **Wait 60 Seconds Before Retry**, which loops back to the status check. |
| Pass Run ID to Retry | Set | Re-stores `runId` for retry loop | Is Scrape Complete? | Wait 60 Seconds Before Retry | ## Async Polling Loop<br>ScraperCity jobs run asynchronously and may take several minutes. **Wait 60 Seconds Before Poll** pauses execution. **Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Pass Run ID to Retry** feeds the ID into **Wait 60 Seconds Before Retry**, which loops back to the status check. |
| Wait 60 Seconds Before Retry | Wait | Delay before next polling attempt | Pass Run ID to Retry | Check Scrape Status | ## Async Polling Loop<br>ScraperCity jobs run asynchronously and may take several minutes. **Wait 60 Seconds Before Poll** pauses execution. **Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Pass Run ID to Retry** feeds the ID into **Wait 60 Seconds Before Retry**, which loops back to the status check. |
| Download Trace Results | HTTP Request | Downloads completed trace data | Is Scrape Complete? | Parse Contact Records | ## Results Processing<br>**Download Trace Results** fetches the completed CSV/JSON payload. **Parse Contact Records** normalises fields into a consistent schema (name, phone, email, address). **Remove Duplicate Contacts** deduplicates by name and phone. **Filter Records with Contact Info** drops rows with no usable contact detail before writing to Airtable. |
| Parse Contact Records | Code | Parses and normalizes response records | Download Trace Results | Remove Duplicate Contacts | ## Results Processing<br>**Download Trace Results** fetches the completed CSV/JSON payload. **Parse Contact Records** normalises fields into a consistent schema (name, phone, email, address). **Remove Duplicate Contacts** deduplicates by name and phone. **Filter Records with Contact Info** drops rows with no usable contact detail before writing to Airtable. |
| Remove Duplicate Contacts | Remove Duplicates | Deduplicates contacts by name and phone | Parse Contact Records | Filter Records with Contact Info | ## Results Processing<br>**Download Trace Results** fetches the completed CSV/JSON payload. **Parse Contact Records** normalises fields into a consistent schema (name, phone, email, address). **Remove Duplicate Contacts** deduplicates by name and phone. **Filter Records with Contact Info** drops rows with no usable contact detail before writing to Airtable. |
| Filter Records with Contact Info | Filter | Keeps only contacts with phone or email | Remove Duplicate Contacts | Sync Contacts to Airtable | ## Results Processing<br>**Download Trace Results** fetches the completed CSV/JSON payload. **Parse Contact Records** normalises fields into a consistent schema (name, phone, email, address). **Remove Duplicate Contacts** deduplicates by name and phone. **Filter Records with Contact Info** drops rows with no usable contact detail before writing to Airtable. |
| Sync Contacts to Airtable | Airtable | Creates Airtable records from cleaned contacts | Filter Records with Contact Info |  | ## Results Processing<br>**Download Trace Results** fetches the completed CSV/JSON payload. **Parse Contact Records** normalises fields into a consistent schema (name, phone, email, address). **Remove Duplicate Contacts** deduplicates by name and phone. **Filter Records with Contact Info** drops rows with no usable contact detail before writing to Airtable. |
| Overview | Sticky Note | Canvas documentation and setup notes |  |  | ## How it works<br>1. **Configure Search Inputs** -- set the property owner name, phone, or email you want to trace.<br>2. **Submit Skip Trace Job** posts the lookup to ScraperCity People Finder and receives a `runId`.<br>3. The polling loop waits 60 seconds, then checks job status via **Check Scrape Status** until `SUCCEEDED`.<br>4. **Download Trace Results** fetches the completed data, which **Parse Contact Records** normalises into clean fields.<br>5. Duplicates are removed and records with at least one contact detail are synced to Airtable.<br><br>## Setup steps<br>1. In n8n Credentials, create an **HTTP Header Auth** credential named `ScraperCity API Key` -- header name `Authorization`, value `Bearer YOUR_KEY`.<br>2. Open **Configure Search Inputs** and enter the owner name (or phone/email) you want to look up.<br>3. In **Sync Contacts to Airtable**, connect your Airtable credential and set the correct Base ID and Table ID.<br>4. Make sure your Airtable table has columns: Full Name, Phone, Email, Address, City, State, ZIP, Age. |
| Section - Configuration | Sticky Note | Canvas note for input setup |  |  | ## Configuration<br>**Configure Search Inputs** is the only node you need to edit before running. Set `ownerName`, `ownerPhone`, or `ownerEmail` -- at least one is required. Adjust `maxResults` (default 3) to control how many contact matches ScraperCity returns per person. |
| Section - Job Submission | Sticky Note | Canvas note for API submission |  |  | ## Job Submission<br>**Submit Skip Trace Job** sends the lookup payload to the ScraperCity People Finder endpoint and receives a `runId`. **Store Run ID** saves that ID so the polling loop can reference it throughout execution. |
| Section - Async Polling Loop | Sticky Note | Canvas note for polling behavior |  |  | ## Async Polling Loop<br>ScraperCity jobs run asynchronously and may take several minutes. **Wait 60 Seconds Before Poll** pauses execution. **Check Scrape Status** queries the job. **Is Scrape Complete?** routes to download on success. Otherwise **Pass Run ID to Retry** feeds the ID into **Wait 60 Seconds Before Retry**, which loops back to the status check. |
| Section - Results Processing | Sticky Note | Canvas note for parsing and sync |  |  | ## Results Processing<br>**Download Trace Results** fetches the completed CSV/JSON payload. **Parse Contact Records** normalises fields into a consistent schema (name, phone, email, address). **Remove Duplicate Contacts** deduplicates by name and phone. **Filter Records with Contact Info** drops rows with no usable contact detail before writing to Airtable. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Name it: `When clicking 'Execute workflow'`

3. **Add a Set node for search configuration**
   - Node type: **Set**
   - Name it: `Configure Search Inputs`
   - Add these fields:
     - `ownerName` as String, default example: `John Smith`
     - `ownerPhone` as String, default: empty
     - `ownerEmail` as String, default: empty
     - `streetCityStateZip` as String, default: empty
     - `maxResults` as Number, default: `3`
   - Connect:
     - `When clicking 'Execute workflow'` → `Configure Search Inputs`

4. **Create the ScraperCity API credential**
   - Go to **Credentials**
   - Create credential type: **HTTP Header Auth**
   - Name it: `ScraperCity API Key`
   - Header name: `Authorization`
   - Header value: `Bearer YOUR_KEY`

5. **Add an HTTP Request node to submit the skip trace job**
   - Node type: **HTTP Request**
   - Name it: `Submit Skip Trace Job`
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Header Auth**
   - Select credential: `ScraperCity API Key`
   - Enable **Send Body**
   - Specify body as **JSON**
   - Set the JSON body to:
     ```json
     {
       "name": {{ $json.ownerName ? '["' + $json.ownerName + '"]' : '[]' }},
       "email": {{ $json.ownerEmail ? '["' + $json.ownerEmail + '"]' : '[]' }},
       "phone_number": {{ $json.ownerPhone ? '["' + $json.ownerPhone + '"]' : '[]' }},
       "street_citystatezip": {{ $json.streetCityStateZip ? '["' + $json.streetCityStateZip + '"]' : '[]' }},
       "max_results": {{ $json.maxResults }}
     }
     ```
   - Connect:
     - `Configure Search Inputs` → `Submit Skip Trace Job`

6. **Add a Set node to store the returned run ID**
   - Node type: **Set**
   - Name it: `Store Run ID`
   - Add field:
     - `runId` as String with expression `{{$json.runId}}`
   - Connect:
     - `Submit Skip Trace Job` → `Store Run ID`

7. **Add the first Wait node**
   - Node type: **Wait**
   - Name it: `Wait 60 Seconds Before Poll`
   - Wait amount: `60` seconds
   - Connect:
     - `Store Run ID` → `Wait 60 Seconds Before Poll`

8. **Add an HTTP Request node to poll job status**
   - Node type: **HTTP Request**
   - Name it: `Check Scrape Status`
   - Method: `GET`
   - URL expression:
     `https://app.scrapercity.com/api/v1/scrape/status/{{$json.runId}}`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Header Auth**
   - Credential: `ScraperCity API Key`
   - Connect:
     - `Wait 60 Seconds Before Poll` → `Check Scrape Status`

9. **Add an If node to test completion**
   - Node type: **If**
   - Name it: `Is Scrape Complete?`
   - Condition:
     - Left value: `{{$json.status}}`
     - Operator: **equals**
     - Right value: `SUCCEEDED`
   - Connect:
     - `Check Scrape Status` → `Is Scrape Complete?`

10. **Add a Set node for retry path**
    - Node type: **Set**
    - Name it: `Pass Run ID to Retry`
    - Add field:
      - `runId` as String with expression `{{$json.runId}}`
    - Connect the **false** output of `Is Scrape Complete?` to this node.

11. **Add the retry Wait node**
    - Node type: **Wait**
    - Name it: `Wait 60 Seconds Before Retry`
    - Wait amount: `60` seconds
    - Connect:
      - `Pass Run ID to Retry` → `Wait 60 Seconds Before Retry`

12. **Close the polling loop**
    - Connect:
      - `Wait 60 Seconds Before Retry` → `Check Scrape Status`

13. **Add an HTTP Request node to download results**
    - Node type: **HTTP Request**
    - Name it: `Download Trace Results`
    - Method: `GET`
    - URL expression:
      `https://app.scrapercity.com/api/downloads/{{$json.runId}}`
    - Authentication: **Generic Credential Type**
    - Generic auth type: **HTTP Header Auth**
    - Credential: `ScraperCity API Key`
    - Connect the **true** output of `Is Scrape Complete?` to this node.

14. **Add a Code node to normalize records**
    - Node type: **Code**
    - Name it: `Parse Contact Records`
    - Language: JavaScript
    - Paste this logic:
      ```javascript
      const raw = items[0].json;

      let records = [];

      if (Array.isArray(raw)) {
        records = raw;
      } else if (typeof raw === 'object' && raw.data && Array.isArray(raw.data)) {
        records = raw.data;
      } else if (typeof raw === 'string') {
        const lines = raw.trim().split('\n');
        const headers = lines[0].split(',').map(h => h.trim().replace(/"/g, ''));
        for (let i = 1; i < lines.length; i++) {
          const values = lines[i].split(',').map(v => v.trim().replace(/"/g, ''));
          const row = {};
          headers.forEach((h, idx) => { row[h] = values[idx] || ''; });
          records.push(row);
        }
      } else {
        records = [raw];
      }

      return records.map(r => ({
        json: {
          full_name: r.full_name || r.name || r.Name || '',
          phone: r.phone || r.phone_number || r.primary_phone || r.Phone || '',
          email: r.email || r.Email || '',
          address: r.address || r.current_address || r.Address || '',
          city: r.city || r.City || '',
          state: r.state || r.State || '',
          zip: r.zip || r.zip_code || r.Zip || '',
          age: r.age || r.Age || '',
          raw_source: JSON.stringify(r)
        }
      }));
      ```
   - Connect:
     - `Download Trace Results` → `Parse Contact Records`

15. **Add a Remove Duplicates node**
    - Node type: **Remove Duplicates**
    - Name it: `Remove Duplicate Contacts`
    - Set fields to compare:
      - `full_name`
      - `phone`
    - Connect:
      - `Parse Contact Records` → `Remove Duplicate Contacts`

16. **Add a Filter node**
    - Node type: **Filter**
    - Name it: `Filter Records with Contact Info`
    - Use **OR** logic with these conditions:
      - `{{$json.phone}}` is not empty
      - `{{$json.email}}` is not empty
    - Connect:
      - `Remove Duplicate Contacts` → `Filter Records with Contact Info`

17. **Create the Airtable credential**
    - Go to **Credentials**
    - Create an **Airtable Personal Access Token** credential
    - Ensure the token has access to the target base and permissions to create records

18. **Prepare the Airtable table**
    - In Airtable, create or identify a table with these fields:
      - `Full Name`
      - `Phone`
      - `Email`
      - `Address`
      - `City`
      - `State`
      - `ZIP`
      - `Age`

19. **Add the Airtable node**
    - Node type: **Airtable**
    - Name it: `Sync Contacts to Airtable`
    - Operation: `Create`
    - Select the Airtable credential
    - Set the **Base ID**
    - Set the **Table ID**
    - Map columns:
      - `Full Name` ← `{{$json.full_name}}`
      - `Phone` ← `{{$json.phone}}`
      - `Email` ← `{{$json.email}}`
      - `Address` ← `{{$json.address}}`
      - `City` ← `{{$json.city}}`
      - `State` ← `{{$json.state}}`
      - `ZIP` ← `{{$json.zip}}`
      - `Age` ← `{{$json.age}}`
    - Connect:
      - `Filter Records with Contact Info` → `Sync Contacts to Airtable`

20. **Optionally add canvas notes**
    - Add Sticky Notes if you want the same on-canvas operational guidance as the original workflow.

21. **Test the workflow**
    - Enter at least one search field in `Configure Search Inputs`
    - Run the workflow manually
    - Confirm:
      - ScraperCity returns a `runId`
      - Polling loops until success
      - Results normalize into contact items
      - Airtable rows are created

22. **Recommended hardening improvements**
    - Add a max retry counter to prevent endless polling
    - Add an explicit branch for `FAILED` or `CANCELLED`
    - Improve CSV parsing if ScraperCity returns complex CSV
    - Consider upsert logic in Airtable if duplicate inserts across runs are a concern

### Credential and setup expectations
- **ScraperCity**
  - Credential type: HTTP Header Auth
  - Header: `Authorization`
  - Value: `Bearer YOUR_KEY`
- **Airtable**
  - Credential type: Airtable Personal Access Token
  - Must have base/table access and record creation scope

### Sub-workflow setup
This workflow does **not** use any Execute Workflow / sub-workflow nodes. There are no sub-workflows to create.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is intended to be edited primarily through the `Configure Search Inputs` node before each run. | Operational usage |
| ScraperCity requests are asynchronous, so two Wait nodes are used to implement polling. | Runtime behavior |
| Airtable Base ID and Table ID are not configured in the exported workflow and must be set manually. | Deployment requirement |
| The Airtable schema expected by the workflow is: Full Name, Phone, Email, Address, City, State, ZIP, Age. | Data model |
| The included Code node uses a simple CSV parser and may require improvement for quoted commas or complex CSV output. | Technical limitation |
| The on-canvas setup note specifies the ScraperCity auth format: `Authorization: Bearer YOUR_KEY`. | Credential requirement |