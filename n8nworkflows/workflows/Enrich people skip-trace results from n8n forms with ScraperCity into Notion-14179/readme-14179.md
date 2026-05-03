Enrich people skip-trace results from n8n forms with ScraperCity into Notion

https://n8nworkflows.xyz/workflows/enrich-people-skip-trace-results-from-n8n-forms-with-scrapercity-into-notion-14179


# Enrich people skip-trace results from n8n forms with ScraperCity into Notion

# 1. Workflow Overview

This workflow receives a person search request from an n8n form, submits that request to ScraperCity’s People Finder API, polls until the scrape job finishes, downloads the enriched results, formats and deduplicates them, and finally creates records in a Notion database.

Typical use cases include lead enrichment, skip tracing, contact research, and form-driven people lookup workflows where a non-technical user submits a name, email, or phone number and expects structured output in Notion.

## 1.1 Input Reception and Search Setup
The workflow starts with an n8n form where a user enters a full name, email address, and/or phone number. A Set node then normalizes these inputs and adds default search settings such as `max_results`.

## 1.2 Job Submission and Run ID Capture
The prepared payload is sent to the ScraperCity People Finder API. The response’s `runId` is extracted and stored for later status polling and result download.

## 1.3 Polling Loop and Completion Detection
The workflow enters a controlled loop using Split in Batches plus Wait. Every 60 seconds it checks the job status via the ScraperCity status endpoint until the returned status equals `SUCCEEDED`.

## 1.4 Result Download, Parsing, and Deduplication
Once complete, the workflow downloads the enriched result set, transforms the API response into one clean n8n item per person, and removes duplicate people based on `full_name`.

## 1.5 Notion Record Creation
Each unique processed person is written as a new page into a configured Notion database with mapped fields such as full name, email, phone, address, age, and source run ID.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Search Setup

**Overview:**  
This block captures user-submitted search criteria and converts them into a normalized structure suitable for the ScraperCity API. It also applies a default limit on the number of results requested.

**Nodes Involved:**  
- People Finder Form  
- Configure Search Defaults  

### Node: People Finder Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that exposes a hosted form and starts the workflow on submission.
- **Configuration choices:**  
  - Form title: `People Finder -- Skip Trace Search`
  - Description instructs the user to enter at least one field
  - Three form fields:
    - `Full Name` as text, optional
    - `Email Address` as email, optional
    - `Phone Number` as text, optional
- **Key expressions or variables used:**  
  None in the node itself; it outputs submitted field values as JSON keys using the field labels.
- **Input and output connections:**  
  - No input
  - Output to `Configure Search Defaults`
- **Version-specific requirements:**  
  Uses `typeVersion` 2.2 of the Form Trigger node.
- **Edge cases or potential failure types:**  
  - All fields are optional, so a user can submit an empty form unless additional validation is added elsewhere.
  - Downstream API may reject empty search criteria or return low-value/no results.
  - Webhook/form URL availability depends on workflow activation state.
- **Sub-workflow reference:**  
  None

### Node: Configure Search Defaults
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps incoming form labels to internal API-ready fields and assigns defaults.
- **Configuration choices:**  
  - Creates:
    - `search_name` from `Full Name`
    - `search_email` from `Email Address`
    - `search_phone` from `Phone Number`
    - `max_results` fixed at `5`
  - Empty string fallback is used if a field is absent
- **Key expressions or variables used:**  
  - `={{ $json['Full Name'] ?? '' }}`
  - `={{ $json['Email Address'] ?? '' }}`
  - `={{ $json['Phone Number'] ?? '' }}`
- **Input and output connections:**  
  - Input from `People Finder Form`
  - Output to `Submit People Finder Job`
- **Version-specific requirements:**  
  Uses Set node `typeVersion` 3.4.
- **Edge cases or potential failure types:**  
  - Empty strings are allowed and will later produce empty arrays in the API payload.
  - No validation enforces at least one populated search field.
  - `max_results` is static; if the API has account-level limits, this may need adjustment.
- **Sub-workflow reference:**  
  None

---

## 2.2 Job Submission and Run ID Storage

**Overview:**  
This block sends the actual search request to ScraperCity and captures the returned run identifier. That identifier is the key state value used by the polling and download stages.

**Nodes Involved:**  
- Submit People Finder Job  
- Store Run ID  

### Node: Submit People Finder Job
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to ScraperCity’s People Finder scrape endpoint.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
  - Authentication: generic credential type using HTTP Header Auth
  - Body is sent as JSON
  - Request body dynamically constructs arrays for name, email, and phone number
- **Key expressions or variables used:**  
  - `name`: includes `search_name` as a one-element JSON array if present, else `[]`
  - `email`: includes `search_email` as a one-element JSON array if present, else `[]`
  - `phone_number`: includes `search_phone` as a one-element JSON array if present, else `[]`
  - `max_results`: `{{ $json.max_results }}`
- **Input and output connections:**  
  - Input from `Configure Search Defaults`
  - Output to `Store Run ID`
- **Version-specific requirements:**  
  Uses HTTP Request node `typeVersion` 4.2.
- **Edge cases or potential failure types:**  
  - Authentication failure if the ScraperCity API key is missing or invalid.
  - 4xx response if the API requires at least one populated search array.
  - 5xx or timeout if the external service is unavailable.
  - The body uses string-built JSON fragments inside an expression; malformed values containing quotes could cause formatting issues if not escaped by n8n as expected.
- **Sub-workflow reference:**  
  None

### Node: Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts `runId` from the API response and stores it in a simplified field.
- **Configuration choices:**  
  - Creates `runId` = response `runId`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
- **Input and output connections:**  
  - Input from `Submit People Finder Job`
  - Output to `Loop Controller`
- **Version-specific requirements:**  
  Uses Set node `typeVersion` 3.4.
- **Edge cases or potential failure types:**  
  - If the API response shape changes or does not contain `runId`, later status and download nodes will fail.
  - No validation checks whether `runId` is empty.
- **Sub-workflow reference:**  
  None

---

## 2.3 Polling Loop and Status Check

**Overview:**  
This block implements a repeat-until-complete pattern. It waits 60 seconds between checks, calls the job status endpoint, and branches based on whether the scrape is finished.

**Nodes Involved:**  
- Loop Controller  
- Wait 60 Seconds Before Poll  
- Check Scrape Job Status  
- Is Scrape Complete?  

### Node: Loop Controller
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Used here as a loop control mechanism rather than for traditional batch processing.
- **Configuration choices:**  
  - `batchSize`: `1`
  - `reset`: `false`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from `Store Run ID`
  - Also receives loop-back input from `Is Scrape Complete?` false branch
  - Output to `Wait 60 Seconds Before Poll`
- **Version-specific requirements:**  
  Uses Split In Batches node `typeVersion` 3.
- **Edge cases or potential failure types:**  
  - This pattern works but can be confusing to maintain if the workflow is modified.
  - No explicit max retry count means jobs stuck in non-terminal states could loop indefinitely.
- **Sub-workflow reference:**  
  None

### Node: Wait 60 Seconds Before Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before the next status request.
- **Configuration choices:**  
  - Wait unit: seconds
  - Amount: `60`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from `Loop Controller`
  - Output to `Check Scrape Job Status`
- **Version-specific requirements:**  
  Uses Wait node `typeVersion` 1.1.
- **Edge cases or potential failure types:**  
  - Long-running workflows may consume execution history and require retention planning.
  - If shorter or longer job times are typical, this fixed delay may be suboptimal.
- **Sub-workflow reference:**  
  None

### Node: Check Scrape Job Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries the ScraperCity status endpoint for the current run.
- **Configuration choices:**  
  - Method: `GET`
  - URL dynamically includes the stored run ID
  - Authentication via HTTP Header Auth
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input from `Wait 60 Seconds Before Poll`
  - Output to `Is Scrape Complete?`
- **Version-specific requirements:**  
  Uses HTTP Request node `typeVersion` 4.2.
- **Edge cases or potential failure types:**  
  - If `runId` is missing, the URL becomes invalid.
  - API auth or connectivity issues can interrupt the loop.
  - If the API returns terminal failure states such as `FAILED`, this workflow currently does not branch specifically for them.
- **Sub-workflow reference:**  
  None

### Node: Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the scrape status is exactly `SUCCEEDED`.
- **Configuration choices:**  
  - Strict string equality comparison
  - Condition: `$json.status === 'SUCCEEDED'`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input from `Check Scrape Job Status`
  - True output to `Download Enriched Results`
  - False output back to `Loop Controller`
- **Version-specific requirements:**  
  Uses If node `typeVersion` 2.2 with condition version 2.
- **Edge cases or potential failure types:**  
  - Any non-`SUCCEEDED` status loops again, including `FAILED`, `CANCELLED`, or malformed responses.
  - If status casing differs, strict comparison will fail.
- **Sub-workflow reference:**  
  None

---

## 2.4 Results Download, Parsing, and Deduplication

**Overview:**  
Once ScraperCity marks the run as complete, this block downloads the result payload and converts it into normalized person records. It also removes duplicate records before persistence.

**Nodes Involved:**  
- Download Enriched Results  
- Parse and Format Records  
- Remove Duplicate People  

### Node: Download Enriched Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed result set for the stored run ID.
- **Configuration choices:**  
  - Method: `GET`
  - URL: downloads endpoint with run ID
  - Authentication: HTTP Header Auth
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input from `Is Scrape Complete?` true branch
  - Output to `Parse and Format Records`
- **Version-specific requirements:**  
  Uses HTTP Request node `typeVersion` 4.2.
- **Edge cases or potential failure types:**  
  - If the download endpoint returns a file or non-JSON content in some account configurations, this node may need response format adjustments.
  - Missing or invalid `runId` will cause request failure.
- **Sub-workflow reference:**  
  None

### Node: Parse and Format Records
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation node that standardizes the result shape into one item per person.
- **Configuration choices:**  
  - Reads the first input JSON object
  - Supports several possible response structures:
    - raw array
    - object with `data`
    - object with `results`
    - single object fallback
  - Filters out records without a recognizable name signal
  - Produces normalized fields:
    - `full_name`
    - `email`
    - `phone`
    - `address`
    - `city`
    - `state`
    - `zip`
    - `age`
    - `source_run_id`
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `$('Store Run ID').item.json.runId`
  - Fallbacks such as:
    - `r.full_name ?? [r.first_name, r.last_name].filter(Boolean).join(' ') ?? r.name ?? ''`
    - `r.email ?? r.emails?.[0] ?? ''`
- **Input and output connections:**  
  - Input from `Download Enriched Results`
  - Output to `Remove Duplicate People`
- **Version-specific requirements:**  
  Uses Code node `typeVersion` 2.
- **Edge cases or potential failure types:**  
  - If the HTTP response is not parsed into JSON, `raw` may not match expected shapes.
  - The `full_name` expression prefers the joined first/last name even when it becomes an empty string; this can suppress fallback to `r.name` because `??` does not treat empty string as nullish.
  - Records lacking any name-like fields are discarded.
  - Unexpected nested field naming from the API may omit data unless the code is extended.
- **Sub-workflow reference:**  
  None

### Node: Remove Duplicate People
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Eliminates duplicate person items before they are inserted into Notion.
- **Configuration choices:**  
  - Comparison mode: selected fields
  - Field used: `full_name`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from `Parse and Format Records`
  - Output to `Save Person Record to Notion`
- **Version-specific requirements:**  
  Uses Remove Duplicates node `typeVersion` 2.
- **Edge cases or potential failure types:**  
  - Deduplication on `full_name` alone can merge distinct people who share the same name.
  - Minor formatting differences in names may prevent duplicate detection.
- **Sub-workflow reference:**  
  None

---

## 2.5 Notion Record Creation

**Overview:**  
This final block persists the cleaned person records into Notion. Each person becomes a new database page with mapped properties.

**Nodes Involved:**  
- Save Person Record to Notion  

### Node: Save Person Record to Notion
- **Type and technical role:** `n8n-nodes-base.notion`  
  Creates a Notion database page for each enriched person record.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `create`
  - Title set from `full_name`
  - Database ID is configured via resource locator but currently appears unset in the JSON (`value: "="`), so it must be replaced
  - Property mappings:
    - `Full Name|title` ← `full_name`
    - `Email|rich_text` ← `email`
    - `Phone|rich_text` ← `phone`
    - `Address|rich_text` ← joined address, city, state, zip
    - `Age|rich_text` ← `age`
    - `Run ID|rich_text` ← `source_run_id`
- **Key expressions or variables used:**  
  - `={{ $json.full_name }}`
  - `={{ $json.email }}`
  - `={{ $json.phone }}`
  - `={{ [$json.address, $json.city, $json.state, $json.zip].filter(v => v).join(', ') }}`
  - `={{ $json.age }}`
  - `={{ $json.source_run_id }}`
- **Input and output connections:**  
  - Input from `Remove Duplicate People`
  - No downstream node
- **Version-specific requirements:**  
  Uses Notion node `typeVersion` 2.2.
- **Edge cases or potential failure types:**  
  - Will fail if the Notion credential is invalid or the integration is not shared with the target database.
  - Will fail if the database property names do not exactly match the configured keys.
  - Database ID must be filled in manually.
  - Notion rate limits may matter if many records are written in a single execution.
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| People Finder Form | Form Trigger | Receives end-user search input from a hosted form |  | Configure Search Defaults | ## Form input and setup |
| People Finder Form | Form Trigger | Receives end-user search input from a hosted form |  | Configure Search Defaults | Accepts a form submission from a user and sets default search parameters (name, email, phone, max results) before the job is dispatched. |
| Configure Search Defaults | Set | Normalizes form input and adds default API parameters | People Finder Form | Submit People Finder Job | ## Form input and setup |
| Configure Search Defaults | Set | Normalizes form input and adds default API parameters | People Finder Form | Submit People Finder Job | Accepts a form submission from a user and sets default search parameters (name, email, phone, max results) before the job is dispatched. |
| Submit People Finder Job | HTTP Request | Sends scrape job request to ScraperCity | Configure Search Defaults | Store Run ID | ## Job submission and ID storage |
| Submit People Finder Job | HTTP Request | Sends scrape job request to ScraperCity | Configure Search Defaults | Store Run ID | POSTs the search request to the ScraperCity people-finder API and stores the returned run ID for use in the polling loop. |
| Store Run ID | Set | Extracts and stores the ScraperCity run identifier | Submit People Finder Job | Loop Controller | ## Job submission and ID storage |
| Store Run ID | Set | Extracts and stores the ScraperCity run identifier | Submit People Finder Job | Loop Controller | POSTs the search request to the ScraperCity people-finder API and stores the returned run ID for use in the polling loop. |
| Loop Controller | Split In Batches | Controls the polling loop | Store Run ID, Is Scrape Complete? (false) | Wait 60 Seconds Before Poll | ## Polling loop and status check |
| Loop Controller | Split In Batches | Controls the polling loop | Store Run ID, Is Scrape Complete? (false) | Wait 60 Seconds Before Poll | Repeatedly waits 60 seconds then polls the ScraperCity API for job status, looping back until the scrape is confirmed complete. |
| Wait 60 Seconds Before Poll | Wait | Delays before next poll request | Loop Controller | Check Scrape Job Status | ## Polling loop and status check |
| Wait 60 Seconds Before Poll | Wait | Delays before next poll request | Loop Controller | Check Scrape Job Status | Repeatedly waits 60 seconds then polls the ScraperCity API for job status, looping back until the scrape is confirmed complete. |
| Check Scrape Job Status | HTTP Request | Queries current scrape job status | Wait 60 Seconds Before Poll | Is Scrape Complete? | ## Polling loop and status check |
| Check Scrape Job Status | HTTP Request | Queries current scrape job status | Wait 60 Seconds Before Poll | Is Scrape Complete? | Repeatedly waits 60 seconds then polls the ScraperCity API for job status, looping back until the scrape is confirmed complete. |
| Is Scrape Complete? | If | Branches on scrape completion state | Check Scrape Job Status | Download Enriched Results; Loop Controller | ## Polling loop and status check |
| Is Scrape Complete? | If | Branches on scrape completion state | Check Scrape Job Status | Download Enriched Results; Loop Controller | Repeatedly waits 60 seconds then polls the ScraperCity API for job status, looping back until the scrape is confirmed complete. |
| Download Enriched Results | HTTP Request | Downloads completed enriched result set | Is Scrape Complete? | Parse and Format Records | ## Results download and processing |
| Download Enriched Results | HTTP Request | Downloads completed enriched result set | Is Scrape Complete? | Parse and Format Records | Downloads the enriched results file, parses and formats each person record with custom code, then removes any duplicates. |
| Parse and Format Records | Code | Normalizes the API response into person items | Download Enriched Results | Remove Duplicate People | ## Results download and processing |
| Parse and Format Records | Code | Normalizes the API response into person items | Download Enriched Results | Remove Duplicate People | Downloads the enriched results file, parses and formats each person record with custom code, then removes any duplicates. |
| Remove Duplicate People | Remove Duplicates | Deduplicates formatted people before persistence | Parse and Format Records | Save Person Record to Notion | ## Results download and processing |
| Remove Duplicate People | Remove Duplicates | Deduplicates formatted people before persistence | Parse and Format Records | Save Person Record to Notion | Downloads the enriched results file, parses and formats each person record with custom code, then removes any duplicates. |
| Save Person Record to Notion | Notion | Creates one Notion database page per unique person | Remove Duplicate People |  | ## Notion record creation |
| Save Person Record to Notion | Notion | Creates one Notion database page per unique person | Remove Duplicate People |  | Saves each unique, enriched person record as a new page in the configured Notion database. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Find and enrich people records from form submissions into Notion`.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: `People Finder Form`
   - Set:
     - Form title: `People Finder -- Skip Trace Search`
     - Description: `Enter at least one field to begin a people search. Results will be saved to Notion automatically.`
   - Add three fields:
     1. `Full Name` → Text → optional
     2. `Email Address` → Email → optional
     3. `Phone Number` → Text → optional
   - This is the workflow entry point.

3. **Add a Set node after the form**
   - Node type: **Set**
   - Name: `Configure Search Defaults`
   - Add the following fields:
     - `search_name` as string: `{{ $json['Full Name'] ?? '' }}`
     - `search_email` as string: `{{ $json['Email Address'] ?? '' }}`
     - `search_phone` as string: `{{ $json['Phone Number'] ?? '' }}`
     - `max_results` as number: `5`
   - Connect `People Finder Form` → `Configure Search Defaults`.

4. **Create ScraperCity credentials**
   - In n8n, create credentials of type **HTTP Header Auth**
   - Name them something like `ScraperCity API Key`
   - Add the required header expected by ScraperCity, typically an API key header as documented by ScraperCity.
   - Use the same credential on all ScraperCity HTTP Request nodes.

5. **Add the job submission HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `Submit People Finder Job`
   - Configure:
     - Method: `POST`
     - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Header Auth**
     - Send Body: enabled
     - Specify Body: **JSON**
   - Set the JSON body to:
     ```json
     {
       "name": {{ $json.search_name ? '["' + $json.search_name + '"]' : '[]' }},
       "email": {{ $json.search_email ? '["' + $json.search_email + '"]' : '[]' }},
       "phone_number": {{ $json.search_phone ? '["' + $json.search_phone + '"]' : '[]' }},
       "street_citystatezip": [],
       "max_results": {{ $json.max_results }}
     }
     ```
   - Attach the `ScraperCity API Key` credential.
   - Connect `Configure Search Defaults` → `Submit People Finder Job`.

6. **Add a Set node to store the run ID**
   - Node type: **Set**
   - Name: `Store Run ID`
   - Add one field:
     - `runId` as string: `{{ $json.runId }}`
   - Connect `Submit People Finder Job` → `Store Run ID`.

7. **Add a Split In Batches node for loop control**
   - Node type: **Split In Batches**
   - Name: `Loop Controller`
   - Set:
     - Batch size: `1`
     - Reset option: `false`
   - Connect `Store Run ID` → `Loop Controller`.

8. **Add a Wait node**
   - Node type: **Wait**
   - Name: `Wait 60 Seconds Before Poll`
   - Configure:
     - Unit: `Seconds`
     - Amount: `60`
   - Connect `Loop Controller` → `Wait 60 Seconds Before Poll`.

9. **Add a status-check HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `Check Scrape Job Status`
   - Configure:
     - Method: `GET`
     - URL:
       `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Header Auth**
   - Reuse the `ScraperCity API Key` credential.
   - Connect `Wait 60 Seconds Before Poll` → `Check Scrape Job Status`.

10. **Add an If node to test completion**
    - Node type: **If**
    - Name: `Is Scrape Complete?`
    - Condition:
      - Left value: `{{ $json.status }}`
      - Operator: **equals**
      - Right value: `SUCCEEDED`
    - Use strict comparison behavior if available.
    - Connect `Check Scrape Job Status` → `Is Scrape Complete?`.

11. **Create the loop-back path**
    - Connect the **false** output of `Is Scrape Complete?` back to `Loop Controller`.
    - This creates the repeating poll cycle.

12. **Add the results download HTTP Request node**
    - Node type: **HTTP Request**
    - Name: `Download Enriched Results`
    - Configure:
      - Method: `GET`
      - URL:
        `=https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
      - Authentication: **Generic Credential Type**
      - Generic Auth Type: **HTTP Header Auth**
    - Reuse the `ScraperCity API Key` credential.
    - Connect the **true** output of `Is Scrape Complete?` → `Download Enriched Results`.

13. **Add a Code node to parse the response**
    - Node type: **Code**
    - Name: `Parse and Format Records`
    - Language: JavaScript
    - Paste this logic:
      ```javascript
      const raw = $input.first().json;

      let records = [];
      if (Array.isArray(raw)) {
        records = raw;
      } else if (Array.isArray(raw.data)) {
        records = raw.data;
      } else if (Array.isArray(raw.results)) {
        records = raw.results;
      } else {
        records = [raw];
      }

      return records
        .filter(r => r && (r.name || r.full_name || r.first_name))
        .map(r => ({
          json: {
            full_name: r.full_name ?? [r.first_name, r.last_name].filter(Boolean).join(' ') ?? r.name ?? '',
            email: r.email ?? r.emails?.[0] ?? '',
            phone: r.phone ?? r.phone_number ?? r.phones?.[0] ?? '',
            address: r.address ?? r.street_address ?? '',
            city: r.city ?? '',
            state: r.state ?? '',
            zip: r.zip ?? r.postal_code ?? '',
            age: r.age ?? '',
            source_run_id: $('Store Run ID').item.json.runId
          }
        }));
      ```
   - Connect `Download Enriched Results` → `Parse and Format Records`.

14. **Add a Remove Duplicates node**
    - Node type: **Remove Duplicates**
    - Name: `Remove Duplicate People`
    - Configure:
      - Compare mode: **Selected Fields**
      - Field to compare: `full_name`
    - Connect `Parse and Format Records` → `Remove Duplicate People`.

15. **Prepare Notion**
    - In Notion, create a database with properties matching the node mapping:
      - `Full Name` → Title
      - `Email` → Rich text
      - `Phone` → Rich text
      - `Address` → Rich text
      - `Age` → Rich text
      - `Run ID` → Rich text
    - Create a Notion integration and share the database with it.
    - In n8n, create or connect **Notion API credentials**.

16. **Add the Notion node**
    - Node type: **Notion**
    - Name: `Save Person Record to Notion`
    - Configure:
      - Resource: `Database Page`
      - Operation: `Create`
      - Select the target database ID
      - Title: `{{ $json.full_name }}`
    - Map properties:
      - `Full Name|title` → `{{ $json.full_name }}`
      - `Email|rich_text` → `{{ $json.email }}`
      - `Phone|rich_text` → `{{ $json.phone }}`
      - `Address|rich_text` → `{{ [$json.address, $json.city, $json.state, $json.zip].filter(v => v).join(', ') }}`
      - `Age|rich_text` → `{{ $json.age }}`
      - `Run ID|rich_text` → `{{ $json.source_run_id }}`
   - Attach your Notion credential.
   - Connect `Remove Duplicate People` → `Save Person Record to Notion`.

17. **Validate the full connection order**
   - `People Finder Form`
   - `Configure Search Defaults`
   - `Submit People Finder Job`
   - `Store Run ID`
   - `Loop Controller`
   - `Wait 60 Seconds Before Poll`
   - `Check Scrape Job Status`
   - `Is Scrape Complete?`
     - **true** → `Download Enriched Results`
     - **false** → back to `Loop Controller`
   - `Download Enriched Results`
   - `Parse and Format Records`
   - `Remove Duplicate People`
   - `Save Person Record to Notion`

18. **Activate and test**
   - Activate the workflow so the form is live.
   - Submit a test with at least one populated field.
   - Confirm:
     - ScraperCity returns a valid `runId`
     - The status endpoint eventually returns `SUCCEEDED`
     - Records are downloaded and parsed
     - Notion pages are created correctly

19. **Recommended hardening improvements**
   - Add validation after the form to reject empty submissions.
   - Add a max poll count or timeout guard.
   - Add explicit handling for `FAILED` or `CANCELLED` statuses.
   - Consider deduplicating by multiple fields such as `full_name + email + phone`.
   - If ScraperCity returns file/binary data instead of JSON, configure the download node accordingly before parsing.

**Sub-workflow setup:**  
This workflow does not invoke any sub-workflows and has only one entry point: `People Finder Form`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Find and enrich people records from form submissions into Notion | Workflow title/context |
| A form submission triggers the workflow, capturing name, email, and phone details to search for. | Overall process |
| Default search parameters are configured and a people-finder scrape job is submitted to the ScraperCity API. | Overall process |
| The workflow stores the returned run ID and enters a polling loop, waiting 60 seconds between each status check. | Overall process |
| Once the scrape job is confirmed complete, enriched results are downloaded from the API. | Overall process |
| Results are parsed, formatted, and deduplicated to produce clean person records. | Overall process |
| Each unique person record is saved as a new entry in a Notion database. | Overall process |
| Create a ScraperCity account and obtain an API key for the people-finder endpoint. | Setup requirement |
| Configure the ScraperCity API credentials in n8n for Submit People Finder Job, Check Scrape Job Status, and Download Enriched Results. | Setup requirement |
| Create a Notion integration and share the target database with it, then add Notion credentials in n8n. | Setup requirement |
| Update the Notion node to point to your specific database and map the desired fields. | Setup requirement |
| Adjust the Configure Search Defaults node to set appropriate max_results and other defaults. | Setup requirement |
| Test the form submission end-to-end with a known person to verify records are created correctly in Notion. | Setup requirement |
| Increase or decrease the 60-second wait interval in Wait 60 Seconds Before Poll based on typical job completion times. | Customization note |
| Modify the Parse and Format Records code node to extract additional fields returned by the API. | Customization note |
| Add a filter node after deduplication to only save records meeting a minimum data-quality threshold. | Customization note |