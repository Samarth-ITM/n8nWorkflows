Find local GMB leads with 1-star reviews using Apify and Google Sheets

https://n8nworkflows.xyz/workflows/find-local-gmb-leads-with-1-star-reviews-using-apify-and-google-sheets-14126


# Find local GMB leads with 1-star reviews using Apify and Google Sheets

## 1. Workflow Overview

This workflow collects local Google Business Profile leads for a given business category and city, then isolates businesses that have at least one 1-star review and exports them to Google Sheets. It is designed for lead generation use cases such as reputation management outreach, review recovery services, and local agency prospecting.

At a high level, the workflow does the following:

1. Receives user input from a form.
2. Builds a Google Maps search query.
3. Starts an Apify actor run to scrape Google Maps business and review data.
4. Polls Apify until the scrape finishes.
5. Stops with an error if the Apify run fails.
6. Fetches the dataset results if the run succeeds.
7. Filters to businesses with at least one 1-star review.
8. Creates a new timestamped sheet tab and appends formatted lead rows.

### 1.1 Input Reception and Query Preparation
This block captures the requested business type and location from an n8n form and converts them into a search string for Apify. It also stores the target Google Sheet document ID for downstream nodes.

### 1.2 Apify Scraper Launch and Async Polling
This block starts the Apify Google Maps scraping run, then repeatedly polls the run status until it is no longer `RUNNING`.

### 1.3 Run Outcome Handling
Once polling detects completion, this block distinguishes successful runs from failed or aborted ones. Successful runs proceed to dataset retrieval; unsuccessful runs stop execution with an explicit error.

### 1.4 Dataset Retrieval, Filtering, and Export
This block fetches scraped records, keeps only businesses with 1-star reviews, formats the lead data into a tabular structure, creates a new Google Sheets tab, and writes the results.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Query Preparation

**Overview:**  
This block collects the search parameters from the user and prepares normalized values used by the scraper and export steps. It also contains the manual configuration point for the destination Google Sheet ID.

**Nodes Involved:**  
- GMB Lead Form
- Build Search Query

### Node: GMB Lead Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger that exposes a hosted n8n form and starts the workflow on submission.
- **Configuration choices:**
  - Form title: `GMB Lead Finder`
  - Description asks for a business category and city
  - Two fields:
    - `Business type`
    - `Location`
- **Key expressions or variables used:**
  - Outputs form fields directly as JSON keys, including:
    - `$json["Business type"]`
    - `$json["Location"]`
- **Input and output connections:**
  - No input; this is the workflow trigger
  - Outputs to `Build Search Query`
- **Version-specific requirements:**
  - Uses form trigger type version `2.5`
  - Requires workflow activation to expose the live form URL
- **Edge cases or potential failure types:**
  - Empty or vague user input may produce low-quality search results
  - If the workflow is inactive, the form URL will not function as intended
  - Field names with spaces must be referenced carefully in expressions
- **Sub-workflow reference:** None

### Node: Build Search Query
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates downstream fields needed by Apify and Google Sheets.
- **Configuration choices:**
  - Sets `search_query` to `<Business type> in <Location>`
  - Copies `Location` into `location`
  - Defines `googleSheetId` as a string field, currently blank and intended for manual setup
- **Key expressions or variables used:**
  - `={{ $json["Business type"] + " in " + $json["Location"] }}`
  - `={{ $json.Location }}`
- **Input and output connections:**
  - Input from `GMB Lead Form`
  - Output to `Start Apify Scraper Run`
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - If `googleSheetId` remains blank, both Google Sheets nodes will fail
  - Null or empty form values will create malformed search queries
  - Typographical errors in user input may reduce scraping quality
- **Sub-workflow reference:** None

---

## 2.2 Apify Scraper Launch and Async Polling

**Overview:**  
This block launches the Apify Google Maps scraper actor with review-focused settings, then checks the run status in a loop every 10 seconds until Apify finishes processing.

**Nodes Involved:**  
- Start Apify Scraper Run
- Poll Apify Run Status
- Run Completed?
- Wait Before Retry

### Node: Start Apify Scraper Run
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to Apify to start an actor run.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.apify.com/v2/acts/nwua9Gu5YrADL7ZDj/runs`
  - Auth: Generic credential via HTTP Header Auth
  - JSON body requests:
    - Search query array from the form-built query
    - Location query from the submitted location
    - Up to 10 places per search
    - Up to 200 reviews
    - Reviews sorted by lowest ranking
    - Contact scraping enabled
    - Most enrichment/social/profile/image options disabled
- **Key expressions or variables used:**
  - `={{ $('Build Search Query').item.json.search_query }}`
  - `={{ $('Build Search Query').item.json.location }}`
- **Input and output connections:**
  - Input from `Build Search Query`
  - Output to `Poll Apify Run Status`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.4`
  - Requires a valid HTTP Header Auth credential containing the Apify bearer token
- **Edge cases or potential failure types:**
  - Invalid or missing Apify token causes authentication failure
  - Actor ID may become invalid if the referenced Apify actor is removed or changed
  - Apify may reject malformed request payloads
  - Rate limits or temporary service issues may interrupt execution
- **Sub-workflow reference:** None

### Node: Poll Apify Run Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Polls the Apify actor run endpoint using the run ID returned by the previous node.
- **Configuration choices:**
  - Method defaults to GET
  - URL dynamically includes `{{$json.data.id}}`
  - Retry on fail enabled
  - `maxTries: 5`
  - `waitBetweenTries: 2000 ms`
- **Key expressions or variables used:**
  - `=https://api.apify.com/v2/actor-runs/{{$json.data.id}}`
- **Input and output connections:**
  - Input from `Start Apify Scraper Run`
  - Also receives loop input from `Wait Before Retry`
  - Output to `Run Completed?`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.4`
- **Edge cases or potential failure types:**
  - If the upstream response does not contain `data.id`, the URL expression fails
  - Network issues may still exceed retries
  - Polling may continue for a long-running scrape, increasing total execution time
- **Sub-workflow reference:** None

### Node: Run Completed?
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether the Apify run is still active.
- **Configuration choices:**
  - Condition: `data.status != RUNNING`
  - True branch means the run is finished in some state
  - False branch means keep waiting and retry polling
- **Key expressions or variables used:**
  - `={{ $json.data.status }}`
- **Input and output connections:**
  - Input from `Poll Apify Run Status`
  - True output to `Run Succeeded?`
  - False output to `Wait Before Retry`
- **Version-specific requirements:**
  - IF node version `2.3`
- **Edge cases or potential failure types:**
  - Statuses such as `READY`, `TIMING-OUT`, `ABORTING`, or other non-`RUNNING` values will be treated as “completed” and passed onward; actual success is checked in the next node
  - Missing `data.status` can break or misroute logic depending on runtime payload
- **Sub-workflow reference:** None

### Node: Wait Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before checking Apify again.
- **Configuration choices:**
  - Wait amount: `10`
  - No unit is explicitly shown in the JSON extract, but in this context it is intended as a short delay loop and behaves as a timed wait node
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from the false branch of `Run Completed?`
  - Output back to `Poll Apify Run Status`
- **Version-specific requirements:**
  - Wait node version `1.1`
  - Wait nodes may resume using n8n’s internal execution resumption mechanism
- **Edge cases or potential failure types:**
  - Long-running polling loops may consume execution history and time
  - If n8n restarts or wait resumption is misconfigured, delayed executions may fail to resume
- **Sub-workflow reference:** None

---

## 2.3 Run Outcome Handling

**Overview:**  
This block distinguishes a successful Apify completion from failure states. It guarantees that dataset fetching only happens when the run status is explicitly `SUCCEEDED`.

**Nodes Involved:**  
- Run Succeeded?
- Stop and Error

### Node: Run Succeeded?
- **Type and technical role:** `n8n-nodes-base.if`  
  Validates that the completed Apify run ended successfully.
- **Configuration choices:**
  - Condition: `data.status == SUCCEEDED`
  - True branch proceeds to dataset retrieval
  - False branch routes to explicit failure handling
- **Key expressions or variables used:**
  - `={{ $json.data.status }}`
- **Input and output connections:**
  - Input from true branch of `Run Completed?`
  - True output to `Fetch Scraped Dataset`
  - False output to `Stop and Error`
- **Version-specific requirements:**
  - IF node version `2.3`
- **Edge cases or potential failure types:**
  - Any non-`SUCCEEDED` completion, including `FAILED`, `ABORTED`, or timeout-related states, is treated as an error
  - Missing status field leads to incorrect routing or expression issues
- **Sub-workflow reference:** None

### Node: Stop and Error
- **Type and technical role:** `n8n-nodes-base.stopAndError`  
  Explicitly terminates execution with a custom error message.
- **Configuration choices:**
  - Error message includes returned Apify status
- **Key expressions or variables used:**
  - `=Apify run failed with status: {{ $json.data.status }}`
- **Input and output connections:**
  - Input from false branch of `Run Succeeded?`
  - No downstream output
- **Version-specific requirements:**
  - Stop and Error node version `1`
- **Edge cases or potential failure types:**
  - If `data.status` is absent, the message may be incomplete
  - This node intentionally fails the run, so it will surface as an execution error in n8n
- **Sub-workflow reference:** None

---

## 2.4 Dataset Retrieval, Filtering, and Export

**Overview:**  
This block downloads the scraped Apify dataset, filters businesses to only those with 1-star reviews, reformats the records into a flat lead schema, creates a new tab in Google Sheets, and appends the final rows.

**Nodes Involved:**  
- Fetch Scraped Dataset
- Has 1-Star Reviews?
- Create Tab for This Run
- Transform & Format GMB Data
- Write Results to Sheet

### Node: Fetch Scraped Dataset
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves the Apify dataset items for the successful run.
- **Configuration choices:**
  - GET request to dataset items endpoint
  - Uses `defaultDatasetId` from the actor run response
  - Query params:
    - `clean=true`
    - `format=json`
  - Retry on fail enabled
  - `maxTries: 5`
  - `waitBetweenTries: 2000 ms`
- **Key expressions or variables used:**
  - `=https://api.apify.com/v2/datasets/{{$json.data.defaultDatasetId}}/items?clean=true&format=json`
- **Input and output connections:**
  - Input from true branch of `Run Succeeded?`
  - Output to `Has 1-Star Reviews?`
- **Version-specific requirements:**
  - HTTP Request node version `4.4`
- **Edge cases or potential failure types:**
  - If `defaultDatasetId` is missing, request URL construction fails
  - Empty datasets will simply produce no usable rows downstream
  - Authentication and network issues can still break retrieval
- **Sub-workflow reference:** None

### Node: Has 1-Star Reviews?
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters each scraped business record to keep only businesses with at least one 1-star review.
- **Configuration choices:**
  - Numeric condition: `reviewsDistribution.oneStar > 0`
  - Only the true branch is used in this workflow
- **Key expressions or variables used:**
  - `={{ $json.reviewsDistribution.oneStar }}`
- **Input and output connections:**
  - Input from `Fetch Scraped Dataset`
  - True output to `Create Tab for This Run`
  - False output unused
- **Version-specific requirements:**
  - IF node version `2.3`
- **Edge cases or potential failure types:**
  - If `reviewsDistribution` is missing, the numeric comparison may fail or behave unexpectedly
  - If no businesses satisfy the condition, downstream nodes may receive no items
- **Sub-workflow reference:** None

### Node: Create Tab for This Run
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Creates a new worksheet tab in an existing Google Sheets document to hold this run’s results.
- **Configuration choices:**
  - Operation: `create`
  - Spreadsheet document ID comes from `Build Search Query.googleSheetId`
  - New tab title is based on the search query plus current timestamp
  - `executeOnce: true` ensures only one tab is created even if multiple items arrive
- **Key expressions or variables used:**
  - `={{ $('Build Search Query').item.json.googleSheetId }}`
  - `={{ $('Build Search Query').item.json.search_query + ' - ' + $now.format('yyyy-MM-dd HH:mm') }}`
- **Input and output connections:**
  - Input from true branch of `Has 1-Star Reviews?`
  - Output to `Transform & Format GMB Data`
- **Version-specific requirements:**
  - Google Sheets node version `4.7`
  - Requires valid Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Blank or invalid spreadsheet ID will cause API failure
  - If the generated tab name exceeds Google Sheets limits or collides with an existing tab name, creation may fail
  - If no items pass the filter, this node may not execute at all
- **Sub-workflow reference:** None

### Node: Transform & Format GMB Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates all filtered records and converts them into a flat export schema for Sheets.
- **Configuration choices:**
  - Pulls all items from `Has 1-Star Reviews?` using `.all()`
  - For each business:
    - Reads reviews array
    - Reads one-star count
    - Searches first up to 10 reviews for one containing review images
    - Builds a new JSON object with selected columns
- **Key expressions or variables used:**
  - `$('Has 1-Star Reviews?').all()`
  - Optional chaining such as:
    - `gmb.json.reviewsDistribution?.oneStar || 0`
    - `reviews[0]?.reviewUrl || ''`
  - Output fields:
    - `Business Name`
    - `GMB URL`
    - `City[Location]`
    - `Phone Number`
    - `Alternative Number`
    - `Business Email`
    - `Negative review URLs`
    - `Negative review URL With Image`
    - `One Star Review Count`
- **Input and output connections:**
  - Triggered after `Create Tab for This Run`
  - Outputs to `Write Results to Sheet`
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - If `Has 1-Star Reviews?` outputs zero items, the code returns an empty array; write step may append nothing
  - If business structures differ from expected Apify output, some fields may be blank
  - Phone formatting wraps the phone value in parentheses, which may not always be desired
  - Only the first business email is captured
  - Only the first review URL and first matching review-with-image URL are exported
- **Sub-workflow reference:** None

### Node: Write Results to Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the formatted lead rows to the newly created worksheet tab.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet document ID from `Build Search Query.googleSheetId`
  - Sheet name from `Create Tab for This Run.title`
  - Columns auto-map to incoming fields
  - Expected columns are explicitly defined in schema
- **Key expressions or variables used:**
  - `={{ $('Create Tab for This Run').item.json.title }}`
  - `={{ $('Build Search Query').item.json.googleSheetId }}`
- **Input and output connections:**
  - Input from `Transform & Format GMB Data`
  - No downstream node
- **Version-specific requirements:**
  - Google Sheets node version `4.7`
  - Requires valid Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - If the sheet tab was not created successfully, append will fail
  - If the incoming data is empty, no rows will be written
  - Schema auto-mapping depends on exact field names from the Code node
- **Sub-workflow reference:** None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GMB Lead Form | formTrigger | Receives business type and location from the user through a hosted form |  | Build Search Query | ## Step 1 Input & Query Builder<br>User fills the form with a business type and location. The Set node combines them into a search string (e.g. "plumber in Miami, FL") and stores the Google Sheet ID for use downstream. |
| Build Search Query | set | Builds the Apify search query, passes location, and stores the Google Sheet ID | GMB Lead Form | Start Apify Scraper Run | ## Step 1 Input & Query Builder<br>User fills the form with a business type and location. The Set node combines them into a search string (e.g. "plumber in Miami, FL") and stores the Google Sheet ID for use downstream. |
| Start Apify Scraper Run | httpRequest | Starts the Apify Google Maps scraping actor run | Build Search Query | Poll Apify Run Status | ## Step 2 Apify Scraper & Polling Loop<br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Poll Apify Run Status | httpRequest | Polls Apify for actor run status | Start Apify Scraper Run; Wait Before Retry | Run Completed? | ## Step 2 Apify Scraper & Polling Loop<br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Run Completed? | if | Checks whether Apify is still running or has reached a terminal status | Poll Apify Run Status | Run Succeeded?; Wait Before Retry | ## Step 2 Apify Scraper & Polling Loop<br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Wait Before Retry | wait | Delays the next polling attempt by 10 seconds | Run Completed? | Poll Apify Run Status | ## Step 2 Apify Scraper & Polling Loop<br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Run Succeeded? | if | Confirms Apify run status is SUCCEEDED before dataset retrieval | Run Completed? | Fetch Scraped Dataset; Stop and Error | ## Step 2 Apify Scraper & Polling Loop<br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Stop and Error | stopAndError | Halts the workflow if Apify completed unsuccessfully | Run Succeeded? |  | ## Step 2 Apify Scraper & Polling Loop<br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Fetch Scraped Dataset | httpRequest | Downloads scraped Google Maps dataset items from Apify | Run Succeeded? | Has 1-Star Reviews? | ## Step 3 Filter, Transform & Export<br>Only businesses with at least one 1-star review pass through. Key fields name, phone, email, GMB URL, review links are extracted and written to a new timestamped tab in Google Sheets. |
| Has 1-Star Reviews? | if | Filters businesses to those with at least one 1-star review | Fetch Scraped Dataset | Create Tab for This Run | ## Step 3 Filter, Transform & Export<br>Only businesses with at least one 1-star review pass through. Key fields name, phone, email, GMB URL, review links are extracted and written to a new timestamped tab in Google Sheets. |
| Create Tab for This Run | googleSheets | Creates a new timestamped worksheet tab for the current run | Has 1-Star Reviews? | Transform & Format GMB Data | ## Step 3 Filter, Transform & Export<br>Only businesses with at least one 1-star review pass through. Key fields name, phone, email, GMB URL, review links are extracted and written to a new timestamped tab in Google Sheets. |
| Transform & Format GMB Data | code | Reshapes filtered Apify business records into flat spreadsheet rows | Create Tab for This Run | Write Results to Sheet | ## Step 3 Filter, Transform & Export<br>Only businesses with at least one 1-star review pass through. Key fields name, phone, email, GMB URL, review links are extracted and written to a new timestamped tab in Google Sheets. |
| Write Results to Sheet | googleSheets | Appends formatted leads into the created worksheet tab | Transform & Format GMB Data |  | ## Step 3 Filter, Transform & Export<br>Only businesses with at least one 1-star review pass through. Key fields name, phone, email, GMB URL, review links are extracted and written to a new timestamped tab in Google Sheets. |
| Sticky Note – Workflow Overview | stickyNote | Workspace documentation note describing workflow purpose and setup |  |  | ## GMB Lead Finder<br><br>Finds local businesses with 1-star Google reviews — ideal for reputation management outreach.<br><br>### How it works<br>1. User submits a business type (e.g. "dentist") and city via the web form.<br>2. Workflow builds a search query and triggers an Apify Google Maps scraper.<br>3. Apify runs async — polls every 10 s until the job finishes.<br>4. Only businesses with at least one 1-star review are kept.<br>5. A new tab is created in Google Sheets and the leads are written there.<br><br>### Setup steps<br>- [ ] In Build Search Query, paste your Google Sheet ID into the googleSheetId field.<br>- [ ] Add your Apify API token as HTTP Header Auth (header: Authorization, value: Bearer YOUR_TOKEN).<br>- [ ] Connect your Google Sheets OAuth2 credential to the two Sheets nodes.<br>- [ ] Activate the workflow to get the form URL.<br><br>### Customization<br>- Increase maxCrawledPlacesPerSearch to scrape more businesses.<br>- Increase maxReviews to collect more review data. |
| Sticky Note – Apify Loop | stickyNote | Workspace documentation note for the input/query section |  |  | ## Step 1 Input & Query Builder<br><br>User fills the form with a business type and location. The Set node combines them into a search string (e.g. "plumber in Miami, FL") and stores the Google Sheet ID for use downstream. |
| Sticky Note1 | stickyNote | Workspace documentation note for the Apify polling loop |  |  | ## Step 2 Apify Scraper & Polling Loop<br><br>Apify runs asynchronously. The workflow starts the scrape job, then polls every 10 s until the status is no longer RUNNING.<br>- ✅ SUCCEEDED → fetch the dataset and continue<br>- ❌ FAILED / ABORTED → Stop and Error halts the execution<br><br>Both HTTP nodes retry up to 5× on network failure. |
| Sticky Note2 | stickyNote | Workspace documentation note for filter/transform/export |  |  | ## Step 3 Filter, Transform & Export<br><br>Only businesses with at least one 1-star review pass through. Key fields name, phone, email, GMB URL, review links are extracted and written to a new timestamped tab in Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a descriptive name such as:  
   `GMB Lead Finder — Scrape Google Business Reviews by Type & Location`.

2. **Add a Form Trigger node** named `GMB Lead Form`.
   - Node type: `Form Trigger`
   - Set form title to `GMB Lead Finder`
   - Set form description to:  
     `Enter a business category and city to find local businesses with 1-star Google reviews.`
   - Add two fields:
     1. `Business type` with placeholder `e.g. plumber, roofer, dentist`
     2. `Location` with placeholder `e.g. Miami, FL`

3. **Add a Set node** named `Build Search Query`.
   - Connect `GMB Lead Form -> Build Search Query`
   - Add these fields:
     - `search_query` as string:  
       `={{ $json["Business type"] + " in " + $json["Location"] }}`
     - `location` as string:  
       `={{ $json.Location }}`
     - `googleSheetId` as string:  
       set this manually to your Google Sheets file ID
   - Important: the Google Sheet ID must be the spreadsheet document ID, not a tab name

4. **Create an HTTP Header Auth credential for Apify**.
   - Credential type: `HTTP Header Auth`
   - Header name: `Authorization`
   - Header value: `Bearer YOUR_APIFY_TOKEN`

5. **Add an HTTP Request node** named `Start Apify Scraper Run`.
   - Connect `Build Search Query -> Start Apify Scraper Run`
   - Method: `POST`
   - URL:  
     `https://api.apify.com/v2/acts/nwua9Gu5YrADL7ZDj/runs`
   - Authentication: Generic Credential Type
   - Generic Auth Type: HTTP Header Auth
   - Select the Apify credential created above
   - Send Body: enabled
   - Specify Body: `JSON`
   - JSON body:
     - `searchStringsArray`: array containing the built query
     - `includeWebResults`: `false`
     - `language`: `en`
     - `locationQuery`: location from the Set node
     - `maxCrawledPlacesPerSearch`: `10`
     - `scrapeContacts`: `true`
     - `maxImages`: `0`
     - `maxReviews`: `200`
     - `maximumLeadsEnrichmentRecords`: `0`
     - `reviewsOrigin`: `google`
     - `reviewsSort`: `lowestRanking`
     - `scrapeDirectories`: `false`
     - `scrapeImageAuthors`: `false`
     - `scrapePlaceDetailPage`: `false`
     - `scrapeReviewsPersonalData`: `true`
     - Social media profile scraping options all `false`
     - `scrapeTableReservationProvider`: `false`
     - `skipClosedPlaces`: `false`
   - Use these dynamic expressions where needed:
     - Search query: `={{ $('Build Search Query').item.json.search_query }}`
     - Location: `={{ $('Build Search Query').item.json.location }}`

6. **Add an HTTP Request node** named `Poll Apify Run Status`.
   - Connect `Start Apify Scraper Run -> Poll Apify Run Status`
   - Method: `GET`
   - URL:  
     `=https://api.apify.com/v2/actor-runs/{{$json.data.id}}`
   - Authentication: same Apify HTTP Header Auth credential
   - Enable retry on fail
   - Set max tries to `5`
   - Set wait between tries to `2000 ms`

7. **Add an IF node** named `Run Completed?`.
   - Connect `Poll Apify Run Status -> Run Completed?`
   - Condition:
     - Left value: `={{ $json.data.status }}`
     - Operator: `notEquals`
     - Right value: `RUNNING`
   - True output = run has finished in some status
   - False output = keep polling

8. **Add a Wait node** named `Wait Before Retry`.
   - Connect `Run Completed?` false output -> `Wait Before Retry`
   - Set wait amount to `10` seconds
   - Then connect `Wait Before Retry -> Poll Apify Run Status`
   - This creates the polling loop

9. **Add another IF node** named `Run Succeeded?`.
   - Connect `Run Completed?` true output -> `Run Succeeded?`
   - Condition:
     - Left value: `={{ $json.data.status }}`
     - Operator: `equals`
     - Right value: `SUCCEEDED`

10. **Add a Stop and Error node** named `Stop and Error`.
    - Connect `Run Succeeded?` false output -> `Stop and Error`
    - Error message:
      `=Apify run failed with status: {{ $json.data.status }}`

11. **Add an HTTP Request node** named `Fetch Scraped Dataset`.
    - Connect `Run Succeeded?` true output -> `Fetch Scraped Dataset`
    - Method: `GET`
    - URL:  
      `=https://api.apify.com/v2/datasets/{{$json.data.defaultDatasetId}}/items?clean=true&format=json`
    - Authentication: same Apify credential
    - Enable retry on fail
    - Set max tries to `5`
    - Set wait between tries to `2000 ms`

12. **Add an IF node** named `Has 1-Star Reviews?`.
    - Connect `Fetch Scraped Dataset -> Has 1-Star Reviews?`
    - Condition:
      - Left value: `={{ $json.reviewsDistribution.oneStar }}`
      - Operator: `greater than`
      - Right value: `0`

13. **Create Google Sheets OAuth2 credentials**.
    - Credential type: `Google Sheets OAuth2 API`
    - Authorize access to the Google account that owns or can edit the target spreadsheet

14. **Add a Google Sheets node** named `Create Tab for This Run`.
    - Connect `Has 1-Star Reviews?` true output -> `Create Tab for This Run`
    - Operation: `Create`
    - Document ID:  
      `={{ $('Build Search Query').item.json.googleSheetId }}`
    - Title:
      `={{ $('Build Search Query').item.json.search_query + ' - ' + $now.format('yyyy-MM-dd HH:mm') }}`
    - Enable `Execute Once`
    - Attach the Google Sheets OAuth2 credential

15. **Add a Code node** named `Transform & Format GMB Data`.
    - Connect `Create Tab for This Run -> Transform & Format GMB Data`
    - Paste this logic conceptually:
      - Read all items from `Has 1-Star Reviews?`
      - For each item:
        - Read reviews array and one-star count
        - Find the first review among the first 10 reviews that contains images
        - Build a flat object with:
          - Business Name
          - GMB URL
          - City[Location]
          - Phone Number
          - Alternative Number
          - Business Email
          - Negative review URLs
          - Negative review URL With Image
          - One Star Review Count
    - Expected output per item:
      - `Business Name` from business title
      - `GMB URL` from business URL
      - `City[Location]` from address
      - `Phone Number` from `phone`
      - `Alternative Number` from `phoneUnformatted`
      - `Business Email` from first email in `emails`
      - `Negative review URLs` from first review URL
      - `Negative review URL With Image` from first review-with-image URL
      - `One Star Review Count` from review distribution

16. **Add a Google Sheets node** named `Write Results to Sheet`.
    - Connect `Transform & Format GMB Data -> Write Results to Sheet`
    - Operation: `Append`
    - Document ID:  
      `={{ $('Build Search Query').item.json.googleSheetId }}`
    - Sheet Name:
      `={{ $('Create Tab for This Run').item.json.title }}`
    - Use auto-mapping for input data
    - Define or allow these columns:
      - Business Name
      - GMB URL
      - City[Location]
      - Phone Number
      - Alternative Number
      - Business Email
      - Negative review URLs
      - Negative review URL With Image
      - One Star Review Count
    - Attach the same Google Sheets OAuth2 credential

17. **Optionally add sticky notes** to document the canvas.
   Suggested note topics:
   - Workflow overview and setup reminders
   - Input/query block
   - Apify polling loop
   - Filter/transform/export block

18. **Activate the workflow**.
   - This is necessary to use the live form trigger URL

19. **Test the workflow**.
   - Submit values such as:
     - Business type: `dentist`
     - Location: `Miami, FL`
   - Confirm:
     - Apify run starts successfully
     - Polling loops until completion
     - Businesses with 1-star reviews are returned
     - A new Google Sheets tab is created
     - Rows are appended correctly

20. **Validate production constraints**.
   - Confirm the Apify actor remains available
   - Confirm the spreadsheet ID is fixed and valid
   - Confirm tab titles are acceptable to Google Sheets
   - Confirm your Google OAuth credential has edit access to the sheet

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require any child workflow configuration.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| In `Build Search Query`, you must paste your destination Google Sheet ID into the `googleSheetId` field before the export nodes can work. | Workflow setup |
| The Apify credential must be configured as HTTP Header Auth with header `Authorization` and value `Bearer YOUR_TOKEN`. | Workflow setup |
| Both Google Sheets nodes must use a valid Google Sheets OAuth2 credential with write access to the destination spreadsheet. | Workflow setup |
| The workflow must be activated to obtain and use the live form URL from the Form Trigger node. | n8n form operation |
| Main customization levers are `maxCrawledPlacesPerSearch` and `maxReviews` in `Start Apify Scraper Run`. | Scraping depth / performance |
| The workflow is intended for finding local businesses with poor Google reviews for outreach scenarios such as reputation management. | Functional purpose |