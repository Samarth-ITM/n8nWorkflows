Generate LinkedIn leads using Google Sheets and Serper API

https://n8nworkflows.xyz/workflows/generate-linkedin-leads-using-google-sheets-and-serper-api-14682


# Generate LinkedIn leads using Google Sheets and Serper API

# 1. Workflow Overview

This workflow generates LinkedIn lead candidates by reading search criteria from Google Sheets, querying Serper’s Google Search API for LinkedIn profile results, extracting profile information, deduplicating against both the current run and an existing target sheet, and appending only new profiles back into Google Sheets.

Typical use cases:
- Build prospect lists from structured search criteria
- Enrich a lead sheet with fresh LinkedIn profile URLs
- Run periodic lead discovery while avoiding duplicates already stored

The workflow is organized into the following logical blocks.

## 1.1 Manual Trigger and Criteria Intake
The workflow starts manually, then reads rows from a Google Sheets tab containing lead search criteria.

## 1.2 Criteria Filtering and Query Construction
Rows are filtered to keep only active/eligible search rows, then converted into Google-style LinkedIn search queries compatible with Serper.

## 1.3 Batch Execution and Search API Calls
Queries are processed in batches through a Split In Batches node, then sent to the Serper API using an HTTP POST request.

## 1.4 Rate Limiting and API Error Handling
Responses are delayed through a Wait node to reduce request pressure. HTTP request failures are routed to an explicit Stop and Error node.

## 1.5 Profile Extraction and In-Run Deduplication
The workflow parses organic search results, keeps only LinkedIn profile URLs matching Greece-related signals, converts them into normalized lead objects, and deduplicates them by profile URL.

## 1.6 Existing-Sheet Comparison
The workflow reads the target Google Sheet containing previously collected profiles and removes any profile already present by URL or full name.

## 1.7 Save New Profiles and Batch Loop Control
If new profiles exist, they are appended to the target sheet. The workflow then loops back to continue processing the next batch. If no new profiles exist, it still loops to continue batch processing.

---

# 2. Block-by-Block Analysis

## 2.1 Manual Trigger and Criteria Intake

### Overview
This block starts the workflow manually and loads search criteria from a Google Sheets document. It is the entry point for the entire lead-generation process.

### Nodes Involved
- Manual Start
- Get Search Criteria

### Node Details

#### Manual Start
- **Type / role:** `n8n-nodes-base.manualTrigger`; manual execution entry point
- **Configuration choices:** No parameters configured; this node simply starts the workflow when launched from the editor
- **Key expressions / variables:** None
- **Input / output connections:**  
  - Input: none
  - Output: `Get Search Criteria`
- **Version-specific requirements:** Type version 1
- **Edge cases / failures:** No runtime logic; only unavailable in unattended scheduled execution unless replaced with another trigger
- **Sub-workflow reference:** None

#### Get Search Criteria
- **Type / role:** `n8n-nodes-base.googleSheets`; reads search criteria rows from Google Sheets
- **Configuration choices:**
  - Reads from spreadsheet `LinkedIn Targets`
  - Sheet/tab: `Search Criteria`
  - Document ID placeholder: `YOUR_SPREADSHEET_ID`
  - No extra options configured
- **Key expressions / variables:** None in parameters
- **Input / output connections:**  
  - Input: `Manual Start`
  - Output: `Filter Search Results`
- **Version-specific requirements:** Google Sheets node version 4
- **Edge cases / failures:**
  - Missing or invalid Google Sheets credentials
  - Spreadsheet or tab not found
  - Access denied due to sharing/permissions
  - Empty sheet produces no downstream items
- **Sub-workflow reference:** None

---

## 2.2 Criteria Filtering and Query Construction

### Overview
This block removes rows that should not be processed, then builds search queries from the remaining spreadsheet data. It converts structured criteria into request-ready search inputs.

### Nodes Involved
- Filter Search Results
- Create Paginated Search Queries

### Node Details

#### Filter Search Results
- **Type / role:** `n8n-nodes-base.filter`; keeps only rows whose `status` field is empty
- **Configuration choices:**
  - Condition checks whether `{{$json.status}}` is empty
  - Logical combinator: `and`
  - Strict type validation enabled by node defaults/options shown
- **Key expressions / variables:**
  - `={{ $json.status }}`
- **Input / output connections:**  
  - Input: `Get Search Criteria`
  - Output: `Create Paginated Search Queries`
- **Version-specific requirements:** Filter node version 2.3
- **Edge cases / failures:**
  - If `status` is absent, it may be treated as empty and pass
  - If users populate `status` inconsistently, filtering behavior may become ambiguous
  - Rows with whitespace-only values may not behave as intended depending on n8n type coercion
- **Sub-workflow reference:** None

#### Create Paginated Search Queries
- **Type / role:** `n8n-nodes-base.code`; constructs search queries and limits result count
- **Configuration choices:**
  - Reads fields from each row:
    - `job_title`
    - `company`
    - `location`
    - `industry`
    - `num_results`
  - Skips rows where both `job_title` and `company` are empty
  - Builds a query beginning with `linkedin`
  - Appends quoted values for title, company, and location
  - Appends industry as an unquoted term if present
  - Sets `num` to `Math.min(parseInt(item.json.num_results) || 10, 10)`
- **Key expressions / variables:**
  - Internal code variables: `jobTitle`, `company`, `location`, `industry`
  - Output fields:
    - `query`
    - `num`
- **Input / output connections:**  
  - Input: `Filter Search Results`
  - Output: `Batch Process 50 Pages`
- **Version-specific requirements:** Code node version 2
- **Edge cases / failures:**
  - Non-numeric `num_results` falls back to 10
  - Values above 10 are capped at 10
  - Rows missing both `job_title` and `company` are silently skipped
  - No true pagination is implemented despite the node name; each criteria row produces one query item
- **Sub-workflow reference:** None

---

## 2.3 Batch Execution and Search API Calls

### Overview
This block controls throughput and sends search requests to Serper. It batches query items and posts them to the external API.

### Nodes Involved
- Batch Process 50 Pages
- Post to LinkedIn Search API

### Node Details

#### Batch Process 50 Pages
- **Type / role:** `n8n-nodes-base.splitInBatches`; iterates over query items in groups
- **Configuration choices:**
  - Batch size set to `50`
  - Used as the main loop controller for the workflow
- **Key expressions / variables:** None directly, but downstream nodes reference this node’s current item
- **Input / output connections:**  
  - Input: `Create Paginated Search Queries`
  - Output 0: not used
  - Output 1: `Post to LinkedIn Search API`
  - Also receives loopback input from:
    - `Check for New Profiles`
    - `Append New Profiles to Sheets`
- **Version-specific requirements:** Split In Batches version 3
- **Edge cases / failures:**
  - If no input items exist, nothing is processed
  - Loop logic depends on correct return path; accidental rewiring can stop iteration
  - The node name suggests “50 Pages,” but actual items are search-query rows, not pages
- **Sub-workflow reference:** None

#### Post to LinkedIn Search API
- **Type / role:** `n8n-nodes-base.httpRequest`; sends POST requests to Serper’s Google Search API
- **Configuration choices:**
  - URL: `https://google.serper.dev/search`
  - Method: `POST`
  - Sends headers and JSON body
  - Headers:
    - `X-API-KEY: YOUR_API_KEY`
    - `Content-Type: application/json`
  - Body:
    - `q = {{ $('Batch Process 50 Pages').last().json.query }}`
    - `num = {{ parseInt($('Batch Process 50 Pages').last().json.num) }}`
  - HTTP node batching option enabled with internal batch size `5`
  - `onError` is set to continue to error output
- **Key expressions / variables:**
  - `={{ $('Batch Process 50 Pages').last().json.query }}`
  - `={{ parseInt($('Batch Process 50 Pages').last().json.num) }}`
- **Input / output connections:**  
  - Input: `Batch Process 50 Pages`
  - Success output: `Wait for Rate Limit`
  - Error output: `Terminate on Error`
- **Version-specific requirements:** HTTP Request version 4.2
- **Edge cases / failures:**
  - Invalid or missing API key
  - Serper quota exhaustion / 429 rate limit
  - Network failure or timeout
  - Unexpected response structure causing downstream parsing gaps
  - Use of `.last()` may be risky if batch semantics are misunderstood; it assumes the current batch’s effective item can be referenced this way
- **Sub-workflow reference:** None

---

## 2.4 Rate Limiting and API Error Handling

### Overview
This block introduces a wait stage after API calls and routes API failures to a hard stop. It is intended to reduce pressure on the external API and make failures explicit.

### Nodes Involved
- Wait for Rate Limit
- Terminate on Error

### Node Details

#### Wait for Rate Limit
- **Type / role:** `n8n-nodes-base.wait`; pauses processing between API request and parsing
- **Configuration choices:**
  - Wait unit is set to `seconds`
  - No explicit duration value is visible in the exported parameters, so behavior depends on n8n defaults or omitted configuration state
  - Webhook ID present: `rate-limit-pause`
- **Key expressions / variables:** None
- **Input / output connections:**  
  - Input: `Post to LinkedIn Search API` success output
  - Output: `Extract Profile Data`
- **Version-specific requirements:** Wait node version 1
- **Edge cases / failures:**
  - If no duration is configured, pause behavior may not match expectations
  - Wait nodes can resume asynchronously depending on execution mode and deployment setup
  - In some hosting setups, wait functionality requires proper database and queue configuration
- **Sub-workflow reference:** None

#### Terminate on Error
- **Type / role:** `n8n-nodes-base.stopAndError`; terminates execution when API request fails
- **Configuration choices:**
  - Error message fixed to `Error`
- **Key expressions / variables:** None
- **Input / output connections:**  
  - Input: `Post to LinkedIn Search API` error output
  - Output: none
- **Version-specific requirements:** Type version 1
- **Edge cases / failures:**
  - The error message is generic and does not preserve original HTTP diagnostics unless inspected in execution logs
  - Any transient API error stops the workflow instead of retrying
- **Sub-workflow reference:** None

---

## 2.5 Profile Extraction and In-Run Deduplication

### Overview
This block parses Serper responses, extracts candidate LinkedIn profiles, applies Greece-related heuristics, normalizes fields, and removes duplicate profile URLs found during the same run.

### Nodes Involved
- Extract Profile Data
- Deduplicate Profiles by URL

### Node Details

#### Extract Profile Data
- **Type / role:** `n8n-nodes-base.code`; transforms API responses into lead objects
- **Configuration choices:**
  - Reads all input items
  - Uses `item.json.organic || []` as source result list
  - Reads page from `item.json.searchParameters?.page || 1`
  - Keeps only results whose URL contains `linkedin.com/in/`
  - Applies a Greece filter:
    - accept if URL contains `gr.linkedin.com`
    - or snippet/title contains one of several Greek-related keywords:
      `greece`, `greek`, `ελλάδα`, `αθήνα`, `athens`, `thessaloniki`, `θεσσαλονίκη`, `piraeus`, `, gr`
  - Attempts to parse:
    - name
    - job title
    - company
    - location
  - Splits full name into `firstName` and `lastName`
  - Creates a guessed `domain` from company by lowercasing, stripping non-alphanumeric characters, and appending `.com`
  - Sets empty `email` and `phone`
  - Sets `scraped_date` to current ISO date
  - Sets `search_page`
  - Performs an additional in-node deduplication by `profile_url`
- **Key expressions / variables:**
  - Input structures:
    - `item.json.organic`
    - `item.json.searchParameters?.page`
  - Output fields:
    - `firstName`
    - `lastName`
    - `fullName`
    - `title`
    - `company`
    - `domain`
    - `location`
    - `profile_url`
    - `email`
    - `phone`
    - `scraped_date`
    - `search_page`
- **Input / output connections:**  
  - Input: `Wait for Rate Limit`
  - Output: `Deduplicate Profiles by URL`
- **Version-specific requirements:** Code node version 2
- **Edge cases / failures:**
  - If Serper response format changes, parsing may miss data
  - Greece filtering may exclude relevant profiles or include false positives
  - Title parsing assumes common Google result formatting and may break on unusual titles
  - Domain generation is heuristic and often inaccurate for real companies
  - Non-Latin names or unconventional title separators may reduce parsing quality
  - Node has `alwaysOutputData: true`, so it can emit an empty set without causing hard failure
- **Sub-workflow reference:** None

#### Deduplicate Profiles by URL
- **Type / role:** `n8n-nodes-base.code`; removes duplicate profiles by `profile_url`
- **Configuration choices:**
  - Reads all incoming items
  - Uses a JavaScript `Set` keyed by `item.json.profile_url`
  - Returns only unique profile objects
- **Key expressions / variables:**
  - `item.json.profile_url`
- **Input / output connections:**  
  - Input: `Extract Profile Data`
  - Output: `Read Existing Profiles from Sheets`
- **Version-specific requirements:** Code node version 2
- **Edge cases / failures:**
  - Profiles without `profile_url` are ignored from uniqueness tracking logic and effectively dropped unless handled elsewhere
  - This duplicates some functionality already present in `Extract Profile Data`
- **Sub-workflow reference:** None

---

## 2.6 Existing-Sheet Comparison

### Overview
This block loads already-saved profiles from the target sheet and excludes any newly found profile already present by URL or by full name.

### Nodes Involved
- Read Existing Profiles from Sheets
- Filter New Profiles

### Node Details

#### Read Existing Profiles from Sheets
- **Type / role:** `n8n-nodes-base.googleSheets`; loads the current target list
- **Configuration choices:**
  - Spreadsheet: `LinkedIn Targets`
  - Sheet/tab: `Target List`
  - Document ID placeholder: `YOUR_SPREADSHEET_ID`
  - No special options configured
- **Key expressions / variables:** None
- **Input / output connections:**  
  - Input: `Deduplicate Profiles by URL`
  - Output: `Filter New Profiles`
- **Version-specific requirements:** Google Sheets version 4
- **Edge cases / failures:**
  - Credential errors or permission denial
  - Large sheets may slow execution
  - Missing expected columns (`profile_url`, `fullName`) weakens deduplication
- **Sub-workflow reference:** None

#### Filter New Profiles
- **Type / role:** `n8n-nodes-base.code`; compares current-run profiles to existing sheet rows
- **Configuration choices:**
  - Pulls new profiles using:
    - `$('Deduplicate Profiles by URL').all().map(i => i.json)`
  - Pulls sheet rows using:
    - `$('Read Existing Profiles from Sheets').all()`
  - Builds:
    - `existingUrls` from `row.profile_url.trim()`
    - `existingNames` from `row.fullName.toLowerCase().trim()`
  - Filters out profiles whose URL or full name already exists
  - Returns only net-new profiles
  - `alwaysOutputData` enabled
- **Key expressions / variables:**
  - `$('Deduplicate Profiles by URL').all()`
  - `$('Read Existing Profiles from Sheets').all()`
  - `profile.profile_url`
  - `profile.fullName`
- **Input / output connections:**  
  - Input: `Read Existing Profiles from Sheets`
  - Output: `Check for New Profiles`
- **Version-specific requirements:** Code node version 2
- **Edge cases / failures:**
  - Name-based deduplication can cause false positives for people with the same name
  - URL normalization is limited; small URL variations may bypass matching
  - If sheet data lacks `fullName` or `profile_url`, duplicates may pass through
  - Since it references upstream nodes directly, rewiring or renaming nodes can break expressions
  - `alwaysOutputData: true` means downstream nodes still execute even when no profiles remain
- **Sub-workflow reference:** None

---

## 2.7 Save New Profiles and Batch Loop Control

### Overview
This block decides whether there is anything to append to the target sheet, saves new profiles if present, and loops back to continue iterating through batches.

### Nodes Involved
- Check for New Profiles
- Append New Profiles to Sheets

### Node Details

#### Check for New Profiles
- **Type / role:** `n8n-nodes-base.if`; tests whether current item data is non-empty
- **Configuration choices:**
  - Condition uses `={{ $json }}`
  - Operator: object `notEmpty`
  - True output goes to append node
  - False output loops back to batch controller
- **Key expressions / variables:**
  - `={{ $json }}`
- **Input / output connections:**  
  - Input: `Filter New Profiles`
  - True output: `Append New Profiles to Sheets`
  - False output: `Batch Process 50 Pages`
- **Version-specific requirements:** IF node version 2
- **Edge cases / failures:**
  - Object `notEmpty` on a per-item basis means behavior depends on whether `Filter New Profiles` returns zero items or at least one item
  - If `Filter New Profiles` returns no items, branch behavior may depend on n8n execution semantics rather than this condition alone
- **Sub-workflow reference:** None

#### Append New Profiles to Sheets
- **Type / role:** `n8n-nodes-base.googleSheets`; appends net-new leads into the target list
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `LinkedIn Targets`
  - Sheet/tab: `Target List`
  - Explicit field mapping:
    - `email`
    - `phone`
    - `title`
    - `domain`
    - `company`
    - `fullName`
    - `lastName`
    - `location`
    - `firstName`
    - `profile_url`
    - `scraped_date`
  - Mapping mode: define below
  - No matching columns used
  - No type conversion forced
- **Key expressions / variables:**
  - Mapped values use `{{$json.fieldName}}` for each lead field
- **Input / output connections:**  
  - Input: `Check for New Profiles` true branch
  - Output: `Batch Process 50 Pages`
- **Version-specific requirements:** Google Sheets version 4
- **Edge cases / failures:**
  - Append duplicates can still happen if concurrent executions write between read and append
  - Missing columns in the sheet can cause mapping issues
  - Permission or quota errors from Google Sheets
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Canvas documentation for full workflow |  |  | ## Untitled workflow / ### How it works / 1. The workflow starts with manually triggering the process. / 2. It reads search criteria and filters the items based on certain conditions. / 3. It builds paginated search queries for further LinkedIn profile searches. / 4. The workflow processes the searches in batches, handling rate limits and errors. / 5. It parses and deduplicates profile data before saving new profiles to a target list. / ### Setup steps / - [ ] Set up Google Sheets API credentials for accessing the spreadsheet. / - [ ] Configure the HTTP request node to interact with the LinkedIn search API. / - [ ] Ensure rate limiting configurations align with API constraints. / ### Customization / The number of profiles processed in parallel and the rate-limit delay can be adjusted based on API usage limits. |
| Sticky Note1 | stickyNote | Canvas documentation for manual trigger and sheet read |  |  | ## Manual trigger and criteria / Start the process and read search criteria from Google Sheets. |
| Sticky Note2 | stickyNote | Canvas documentation for filtering and query construction |  |  | ## Filter and search setup / Filter search criteria and build paginated search inputs. |
| Sticky Note3 | stickyNote | Canvas documentation for batch and API search area |  |  | ## Manage batches and searches / Process search batches and send requests to the LinkedIn search API. |
| Sticky Note4 | stickyNote | Canvas documentation for rate limiting and error handling |  |  | ## Rate limiting and error handling / Handle rate limits and manage errors during API requests. |
| Sticky Note5 | stickyNote | Canvas documentation for parsing and deduplication |  |  | ## Profile parsing and deduplication / Parse API responses and remove duplicate profiles. |
| Sticky Note6 | stickyNote | Canvas documentation for existing profile filtering |  |  | ## Existing profile filtering / Read existing profiles and filter out duplicates. |
| Sticky Note7 | stickyNote | Canvas documentation for sheet append block |  |  | ## Save new profiles / Check for new profiles and save them to a target list. |
| Manual Start | manualTrigger | Manual entry point |  | Get Search Criteria | ## Manual trigger and criteria / Start the process and read search criteria from Google Sheets. |
| Get Search Criteria | googleSheets | Read search criteria rows from source sheet | Manual Start | Filter Search Results | ## Manual trigger and criteria / Start the process and read search criteria from Google Sheets. |
| Filter Search Results | filter | Keep criteria rows whose status is empty | Get Search Criteria | Create Paginated Search Queries | ## Filter and search setup / Filter search criteria and build paginated search inputs. |
| Create Paginated Search Queries | code | Build Serper-ready LinkedIn search queries | Filter Search Results | Batch Process 50 Pages | ## Filter and search setup / Filter search criteria and build paginated search inputs. |
| Batch Process 50 Pages | splitInBatches | Iterate through query items in batches and control loop | Create Paginated Search Queries; Check for New Profiles; Append New Profiles to Sheets | Post to LinkedIn Search API | ## Manage batches and searches / Process search batches and send requests to the LinkedIn search API. |
| Post to LinkedIn Search API | httpRequest | Submit search queries to Serper API | Batch Process 50 Pages | Wait for Rate Limit; Terminate on Error | ## Manage batches and searches / Process search batches and send requests to the LinkedIn search API. |
| Wait for Rate Limit | wait | Delay processing after API request | Post to LinkedIn Search API | Extract Profile Data | ## Rate limiting and error handling / Handle rate limits and manage errors during API requests. |
| Terminate on Error | stopAndError | Hard-stop execution on API error | Post to LinkedIn Search API |  | ## Rate limiting and error handling / Handle rate limits and manage errors during API requests. |
| Extract Profile Data | code | Parse Serper results into normalized profile leads | Wait for Rate Limit | Deduplicate Profiles by URL | ## Profile parsing and deduplication / Parse API responses and remove duplicate profiles. |
| Deduplicate Profiles by URL | code | Remove duplicate profile URLs in current run | Extract Profile Data | Read Existing Profiles from Sheets | ## Profile parsing and deduplication / Parse API responses and remove duplicate profiles. |
| Read Existing Profiles from Sheets | googleSheets | Load already-saved target profiles | Deduplicate Profiles by URL | Filter New Profiles | ## Existing profile filtering / Read existing profiles and filter out duplicates. |
| Filter New Profiles | code | Remove profiles already present in target sheet | Read Existing Profiles from Sheets | Check for New Profiles | ## Existing profile filtering / Read existing profiles and filter out duplicates. |
| Check for New Profiles | if | Decide whether to append current profiles | Filter New Profiles | Append New Profiles to Sheets; Batch Process 50 Pages | ## Save new profiles / Check for new profiles and save them to a target list. |
| Append New Profiles to Sheets | googleSheets | Append net-new leads to target sheet | Check for New Profiles | Batch Process 50 Pages | ## Save new profiles / Check for new profiles and save them to a target list. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `Generate LinkedIn leads using Google Sheets and Serper API`.

2. **Add a Manual Trigger node.**
   - Node type: `Manual Trigger`
   - Name: `Manual Start`

3. **Prepare Google Sheets credentials.**
   - Create Google Sheets credentials in n8n.
   - Ensure the connected Google account has access to the spreadsheet you will use.
   - You need one spreadsheet with at least two tabs:
     - `Search Criteria`
     - `Target List`

4. **Create the Search Criteria sheet structure.**
   - Add columns similar to:
     - `job_title`
     - `company`
     - `location`
     - `industry`
     - `num_results`
     - `status`
   - Intended behavior:
     - rows with empty `status` are processed
     - rows with any non-empty `status` are skipped

5. **Add a Google Sheets node to read criteria.**
   - Node type: `Google Sheets`
   - Name: `Get Search Criteria`
   - Operation: read rows
   - Select your spreadsheet
   - Select sheet/tab: `Search Criteria`
   - Connect: `Manual Start -> Get Search Criteria`

6. **Add a Filter node to keep eligible rows.**
   - Node type: `Filter`
   - Name: `Filter Search Results`
   - Condition:
     - Left value: `{{ $json.status }}`
     - Operator: `is empty`
   - Connect: `Get Search Criteria -> Filter Search Results`

7. **Add a Code node to build search queries.**
   - Node type: `Code`
   - Name: `Create Paginated Search Queries`
   - Paste logic equivalent to:
     - read `job_title`, `company`, `location`, `industry`, `num_results`
     - skip row if both `job_title` and `company` are empty
     - build query string starting with `linkedin`
     - quote title/company/location
     - cap `num_results` at 10
     - output `{ query, num }`
   - Connect: `Filter Search Results -> Create Paginated Search Queries`

8. **Use this logic in the Code node.**
   - Input fields expected from sheet rows:
     - `job_title`
     - `company`
     - `location`
     - `industry`
     - `num_results`
   - Output fields:
     - `query`
     - `num`

9. **Add a Split In Batches node.**
   - Node type: `Split In Batches`
   - Name: `Batch Process 50 Pages`
   - Batch size: `50`
   - Connect: `Create Paginated Search Queries -> Batch Process 50 Pages`

10. **Prepare Serper API access.**
    - Get an API key from Serper.
    - You can store it directly in the node or preferably in credentials/environment variables.

11. **Add an HTTP Request node for Serper.**
    - Node type: `HTTP Request`
    - Name: `Post to LinkedIn Search API`
    - Method: `POST`
    - URL: `https://google.serper.dev/search`
    - Enable sending headers
    - Enable sending body
    - Headers:
      - `X-API-KEY`: your Serper API key
      - `Content-Type`: `application/json`
    - Body parameters:
      - `q`: `{{ $('Batch Process 50 Pages').last().json.query }}`
      - `num`: `{{ parseInt($('Batch Process 50 Pages').last().json.num) }}`
    - Enable node-level batching if you want to mirror the original:
      - batch size `5`
    - Set error handling so the node continues to an error output
    - Connect from the second/main iterative output of `Batch Process 50 Pages` to `Post to LinkedIn Search API`

12. **Add a Wait node for throttling.**
    - Node type: `Wait`
    - Name: `Wait for Rate Limit`
    - Unit: `seconds`
    - Set an explicit delay value appropriate for your Serper quota, for example `1` or more
    - Connect success output:
      - `Post to LinkedIn Search API -> Wait for Rate Limit`

13. **Add a Stop and Error node.**
    - Node type: `Stop and Error`
    - Name: `Terminate on Error`
    - Error message: `Error`
    - Connect error output:
      - `Post to LinkedIn Search API (error output) -> Terminate on Error`

14. **Add a Code node to parse search results.**
    - Node type: `Code`
    - Name: `Extract Profile Data`
    - Enable `Always Output Data`
    - Paste logic equivalent to:
      - read `organic` results from each Serper response
      - keep only URLs containing `linkedin.com/in/`
      - keep only results that appear Greece-related by URL or snippet/title keywords
      - parse title/snippet into:
        - `firstName`
        - `lastName`
        - `fullName`
        - `title`
        - `company`
        - `domain`
        - `location`
        - `profile_url`
        - `email`
        - `phone`
        - `scraped_date`
        - `search_page`
      - deduplicate by `profile_url`
    - Connect: `Wait for Rate Limit -> Extract Profile Data`

15. **Implement the Greece filter if you want the same behavior.**
    - Keep profile if:
      - URL contains `gr.linkedin.com`
      - or title/snippet includes terms such as:
        - `greece`
        - `greek`
        - `ελλάδα`
        - `αθήνα`
        - `athens`
        - `thessaloniki`
        - `θεσσαλονίκη`
        - `piraeus`
        - `, gr`

16. **Add another Code node for URL deduplication.**
    - Node type: `Code`
    - Name: `Deduplicate Profiles by URL`
    - Logic:
      - read all items
      - keep only the first occurrence of each `profile_url`
      - return unique profile objects
    - Connect: `Extract Profile Data -> Deduplicate Profiles by URL`

17. **Create the Target List sheet structure.**
    - In the same spreadsheet, create tab `Target List`
    - Add columns:
      - `firstName`
      - `lastName`
      - `fullName`
      - `title`
      - `company`
      - `domain`
      - `location`
      - `profile_url`
      - `scraped_date`
      - `email`
      - `phone`

18. **Add a Google Sheets node to read existing leads.**
    - Node type: `Google Sheets`
    - Name: `Read Existing Profiles from Sheets`
    - Operation: read rows
    - Select the same spreadsheet
    - Select tab: `Target List`
    - Connect: `Deduplicate Profiles by URL -> Read Existing Profiles from Sheets`

19. **Add a Code node to filter existing leads.**
    - Node type: `Code`
    - Name: `Filter New Profiles`
    - Enable `Always Output Data`
    - Logic:
      - get current-run profiles from `Deduplicate Profiles by URL`
      - get existing sheet rows from `Read Existing Profiles from Sheets`
      - build sets from:
        - `profile_url`
        - lowercase trimmed `fullName`
      - exclude profiles already present by URL or name
      - return only new profiles
    - Connect: `Read Existing Profiles from Sheets -> Filter New Profiles`

20. **Add an IF node to check whether there are profiles to save.**
    - Node type: `IF`
    - Name: `Check for New Profiles`
    - Condition:
      - Left value: `{{ $json }}`
      - Operator: `not empty`
    - Connect: `Filter New Profiles -> Check for New Profiles`

21. **Add a Google Sheets append node.**
    - Node type: `Google Sheets`
    - Name: `Append New Profiles to Sheets`
    - Operation: `Append`
    - Spreadsheet: same spreadsheet
    - Sheet/tab: `Target List`
    - Use explicit field mapping:
      - `firstName -> {{ $json.firstName }}`
      - `lastName -> {{ $json.lastName }}`
      - `fullName -> {{ $json.fullName }}`
      - `title -> {{ $json.title }}`
      - `company -> {{ $json.company }}`
      - `domain -> {{ $json.domain }}`
      - `location -> {{ $json.location }}`
      - `profile_url -> {{ $json.profile_url }}`
      - `scraped_date -> {{ $json.scraped_date }}`
      - `email -> {{ $json.email }}`
      - `phone -> {{ $json.phone }}`
    - Connect true branch:
      - `Check for New Profiles -> Append New Profiles to Sheets`

22. **Create the loop back to the batch node.**
    - Connect false branch:
      - `Check for New Profiles (false) -> Batch Process 50 Pages`
    - Connect append completion:
      - `Append New Profiles to Sheets -> Batch Process 50 Pages`

23. **Verify the batch loop behavior.**
    - The Split In Batches node should:
      - receive initial items from query creation
      - process a batch
      - continue when loopback input returns from either:
        - no new profiles found
        - profiles appended successfully

24. **Optionally add sticky notes for documentation.**
    - Add notes similar to:
      - Manual trigger and criteria
      - Filter and search setup
      - Manage batches and searches
      - Rate limiting and error handling
      - Profile parsing and deduplication
      - Existing profile filtering
      - Save new profiles

25. **Test with a small dataset first.**
    - Use a few rows in `Search Criteria`
    - Set `num_results` low, such as `3` to `5`
    - Run manually and inspect:
      - Serper response structure
      - parsed profile fields
      - duplicate filtering
      - sheet append output

26. **Replace placeholders before production use.**
    - Replace `YOUR_SPREADSHEET_ID`
    - Replace `YOUR_API_KEY`
    - Confirm tab names match exactly

27. **Recommended hardening improvements.**
    - Add retry logic for transient Serper failures
    - Normalize LinkedIn URLs before deduplication
    - Use a more robust location/company parser
    - Add explicit wait duration
    - Add logging or success counters
    - Consider moving API key to credentials or env vars instead of plain text headers

### Expected input/output behavior

#### Search Criteria input
Each row should ideally contain:
- `job_title`
- `company`
- `location`
- `industry`
- `num_results`
- `status`

#### Parsed profile output
Each accepted result becomes a row with:
- `firstName`
- `lastName`
- `fullName`
- `title`
- `company`
- `domain`
- `location`
- `profile_url`
- `email`
- `phone`
- `scraped_date`
- `search_page`

### Credential configuration summary
- **Google Sheets credentials:** required for both read and append operations
- **Serper API key:** required in HTTP header `X-API-KEY`

### Sub-workflow setup
This workflow does **not** use sub-workflows and does **not** invoke any child workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow reads search criteria from a Google Sheet and appends only net-new profiles to a target sheet. | Overall workflow behavior |
| Rows are processed only when the `status` field is empty. | Search criteria filtering logic |
| The current implementation uses Serper’s Google Search endpoint rather than LinkedIn’s own API. | External integration design |
| The workflow includes a Greece-focused filter based on LinkedIn URL patterns and snippet/title keywords. | Profile extraction logic |
| `num_results` is capped to 10 in the query-building code. | Query construction constraint |
| The Wait node should be reviewed to ensure an explicit duration is configured for your environment. | Rate-limit management |
| The batch node name suggests page-based processing, but the actual unit processed is query items generated from sheet rows. | Naming clarification |
| API failures terminate the workflow through a Stop and Error node with a generic message. | Error handling limitation |
| The sheet-based deduplication checks both `profile_url` and `fullName`; same-name collisions are possible. | Deduplication caveat |