Scrape Google Maps businesses with ScraperCity and send Gmail outreach

https://n8nworkflows.xyz/workflows/scrape-google-maps-businesses-with-scrapercity-and-send-gmail-outreach-14448


# Scrape Google Maps businesses with ScraperCity and send Gmail outreach

# 1. Workflow Overview

This workflow automates local-business lead generation and outreach. It starts from a manually defined search configuration, launches a Google Maps scraping job through ScraperCity, waits and polls until the scrape is finished, extracts business records, filters and deduplicates them, submits those domains to ScraperCity’s email-finder service in batches, polls again until email discovery is complete, merges discovered emails back into the original lead records, and finally sends personalized outreach emails through Gmail.

Typical use cases include:
- Building outbound lead lists for local service businesses
- Prospecting businesses in a specific city or region
- Sending semi-personalized cold outreach at small to moderate scale
- Creating a reusable enrichment-and-emailing pipeline that can later be extended with CRM or spreadsheet logging

## 1.1 Entry and Search Configuration
The workflow is manually triggered and defines core parameters such as business type, target location, max result count, sender identity, and subject line.

## 1.2 Google Maps Scrape Submission
The configured search data is sent to ScraperCity’s Maps scraping endpoint, and the returned `runId` is stored with the configuration values needed later.

## 1.3 Maps Job Polling Loop
The workflow waits before the first poll, checks scrape status, and loops every 60 seconds until the Maps job reports `SUCCEEDED`.

## 1.4 Maps Result Parsing and Lead Normalization
Completed scrape results are downloaded and transformed into normalized business records with consistent fields such as `businessName`, `website`, `domain`, `phone`, `address`, and name components.

## 1.5 Lead Filtering and Deduplication
Businesses without usable domains are removed, and duplicate domains are eliminated so downstream enrichment only processes unique contact opportunities.

## 1.6 Email-Finder Batch Preparation and Submission
The leads are split into batches, converted into the contact structure expected by ScraperCity’s email-finder API, and submitted as a second asynchronous scrape job.

## 1.7 Email-Finder Polling Loop
As with the Maps step, the workflow waits, polls job status, and retries every 60 seconds until the email-finder job succeeds.

## 1.8 Email Result Merge and Validation
The email-finder result set is downloaded, matched back to the original business items by domain, and filtered so only records with valid-looking email addresses move forward.

## 1.9 Gmail Outreach and Batch Continuation
Each valid lead receives a personalized text email via Gmail. After sending, execution loops back to the batch node so the next group of businesses can be processed.

---

# 2. Block-by-Block Analysis

## 2.1 Entry and Search Configuration

**Overview:**  
This block starts the workflow and sets all top-level parameters used downstream. It centralizes variables so the rest of the workflow can reference them consistently.

**Nodes Involved:**  
- Start Workflow
- Configure Search Parameters

### Node: Start Workflow
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for interactive execution.
- **Configuration choices:** No parameters are required. The workflow starts only when launched manually in the editor or via a manual execution action.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Configure Search Parameters`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** No runtime failure expected; this is a simple manual entry node.
- **Sub-workflow reference:** None.

### Node: Configure Search Parameters
- **Type and technical role:** `n8n-nodes-base.set`; initializes workflow variables.
- **Configuration choices:** Creates these fields:
  - `businessType` = `"plumbers"`
  - `locationQuery` = `"Austin, TX"`
  - `maxResults` = `50`
  - `senderName` = `"Your Name"`
  - `senderEmail` = `"user@example.com"`
  - `emailSubject` = `"Quick question about your business"`
- **Key expressions or variables used:** Static values in this template, later referenced by many nodes.
- **Input and output connections:**  
  - Input: `Start Workflow`  
  - Output: `Start Google Maps Scrape`
- **Version-specific requirements:** Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Poor-quality parameters may lead to weak or empty scrape results.
  - Very high `maxResults` may increase cost, runtime, or API quota usage.
- **Sub-workflow reference:** None.

---

## 2.2 Google Maps Scrape Submission

**Overview:**  
This block submits the Maps scraping job to ScraperCity and preserves the returned run identifier together with configuration context. Those stored values are reused throughout polling, download, and email composition.

**Nodes Involved:**  
- Start Google Maps Scrape
- Store Maps Run ID

### Node: Start Google Maps Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a POST request to ScraperCity’s Maps scrape API.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/maps`
  - Authentication: generic credential type using HTTP header auth
  - JSON body includes:
    - `searchStringsArray`: array containing `businessType`
    - `locationQuery`
    - `maxCrawledPlacesPerSearch`: `maxResults`
- **Key expressions or variables used:**  
  - `{{ $json.businessType }}`
  - `{{ $json.locationQuery }}`
  - `{{ $json.maxResults }}`
- **Input and output connections:**  
  - Input: `Configure Search Parameters`
  - Output: `Store Maps Run ID`
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing or invalid ScraperCity API key
  - 4xx/5xx API responses
  - Request body rejected due to API schema mismatch
  - Network timeout or DNS/TLS errors
  - Search returning little or no data despite successful request
- **Sub-workflow reference:** None.

### Node: Store Maps Run ID
- **Type and technical role:** `n8n-nodes-base.set`; stores the returned Maps `runId` and duplicates configuration values into the current item.
- **Configuration choices:** Assigns:
  - `mapsRunId` from the API response `runId`
  - `businessType`, `locationQuery`, `senderName`, `senderEmail`, `emailSubject` pulled from `Configure Search Parameters`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
  - `={{ $('Configure Search Parameters').item.json.businessType }}`
  - `={{ $('Configure Search Parameters').item.json.locationQuery }}`
  - `={{ $('Configure Search Parameters').item.json.senderName }}`
  - `={{ $('Configure Search Parameters').item.json.senderEmail }}`
  - `={{ $('Configure Search Parameters').item.json.emailSubject }}`
- **Input and output connections:**  
  - Input: `Start Google Maps Scrape`
  - Output: `Wait Before First Maps Poll`
- **Version-specific requirements:** Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - If `runId` is absent from ScraperCity’s response, all downstream polling will fail.
  - Cross-node expression references depend on `Configure Search Parameters` having executed successfully.
- **Sub-workflow reference:** None.

---

## 2.3 Maps Job Polling Loop

**Overview:**  
This block implements asynchronous job tracking for the Maps scrape. It waits 30 seconds before the first status check, then repeats polling every 60 seconds until the job status equals `SUCCEEDED`.

**Nodes Involved:**  
- Wait Before First Maps Poll
- Poll Maps Scrape Status
- Is Maps Scrape Complete?
- Wait 60s Before Retry Maps Poll

### Node: Wait Before First Maps Poll
- **Type and technical role:** `n8n-nodes-base.wait`; delays first poll to give ScraperCity time to start processing.
- **Configuration choices:** Wait amount `30` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Store Maps Run ID`
  - Output: `Poll Maps Scrape Status`
- **Version-specific requirements:** Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - Long-running workflows may be affected by execution retention settings.
  - If the n8n instance restarts, resume behavior depends on environment and execution mode.
- **Sub-workflow reference:** None.

### Node: Poll Maps Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; checks the status of the submitted Maps scrape.
- **Configuration choices:**  
  - Method: `GET`
  - URL built from `mapsRunId`
  - Authentication via HTTP header auth credential
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Maps Run ID').item.json.mapsRunId }}`
- **Input and output connections:**  
  - Inputs: `Wait Before First Maps Poll`, `Wait 60s Before Retry Maps Poll`
  - Output: `Is Maps Scrape Complete?`
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid or expired API credentials
  - Invalid `mapsRunId`
  - API status endpoint returning temporary errors
  - Rate limiting if polling is too aggressive
- **Sub-workflow reference:** None.

### Node: Is Maps Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`; routes execution based on job completion state.
- **Configuration choices:** Checks whether `status === "SUCCEEDED"` using strict validation and case-sensitive matching.
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Poll Maps Scrape Status`
  - True output: `Download Maps Results`
  - False output: `Wait 60s Before Retry Maps Poll`
- **Version-specific requirements:** If node version `2.2`.
- **Edge cases or potential failure types:**  
  - Jobs with statuses like `FAILED`, `CANCELLED`, or unexpected values are treated the same as “not complete,” causing indefinite retry behavior.
  - Missing `status` field will also route to retry.
- **Sub-workflow reference:** None.

### Node: Wait 60s Before Retry Maps Poll
- **Type and technical role:** `n8n-nodes-base.wait`; throttles repeated polling.
- **Configuration choices:** Wait amount `60` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Is Maps Scrape Complete?` false branch
  - Output: `Poll Maps Scrape Status`
- **Version-specific requirements:** Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - Infinite loop risk if ScraperCity never returns `SUCCEEDED`.
  - No maximum retry count or failure branch is defined.
- **Sub-workflow reference:** None.

---

## 2.4 Maps Result Parsing and Lead Normalization

**Overview:**  
After the Maps scrape completes, this block downloads the result payload and normalizes it into clean business lead records. It also extracts domains and derives approximate first/last name fields for downstream email finding.

**Nodes Involved:**  
- Download Maps Results
- Parse and Clean Business Records

### Node: Download Maps Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads final scrape output from ScraperCity.
- **Configuration choices:**  
  - Method: `GET`
  - URL built from `mapsRunId`
  - Authentication via HTTP header auth
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/downloads/{{ $('Store Maps Run ID').item.json.mapsRunId }}`
- **Input and output connections:**  
  - Input: `Is Maps Scrape Complete?` true branch
  - Output: `Parse and Clean Business Records`
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Download endpoint may return JSON, nested JSON, or text payloads
  - Missing file, expired run, or unsupported format
  - API auth failures and timeout issues
- **Sub-workflow reference:** None.

### Node: Parse and Clean Business Records
- **Type and technical role:** `n8n-nodes-base.code`; transforms raw results into normalized business items.
- **Configuration choices:**  
  - Accepts multiple possible payload shapes:
    - direct array
    - object with `data` array
    - raw string treated as CSV
  - Extracts and normalizes:
    - `businessName`
    - `website`
    - `domain`
    - `phone`
    - `address`
    - `city`
    - `firstName`
    - `lastName`
  - Skips records lacking name or website
  - Deduplicates domains internally using a `Set`
- **Key expressions or variables used:** Uses `$input.first().json`.
- **Input and output connections:**  
  - Input: `Download Maps Results`
  - Output: `Filter Businesses With a Website`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**  
  - CSV parsing is simplistic and can break on quoted commas or complex CSV formatting.
  - Invalid URLs may cause domain parsing fallback behavior.
  - Name splitting assumes business name can be reused as a contact-like name; this is often inaccurate.
  - If the HTTP node returns binary instead of JSON/text, this code would not handle it.
- **Sub-workflow reference:** None.

---

## 2.5 Lead Filtering and Deduplication

**Overview:**  
This block applies a final domain presence check and removes duplicate domains before sending lead batches to email enrichment. It helps avoid wasted API calls and duplicate outreach attempts.

**Nodes Involved:**  
- Filter Businesses With a Website
- Remove Duplicate Domains

### Node: Filter Businesses With a Website
- **Type and technical role:** `n8n-nodes-base.filter`; passes only records with a non-empty `domain`.
- **Configuration choices:**  
  - Condition: `domain` is not empty
  - Case-insensitive, loose validation
- **Key expressions or variables used:**  
  - `={{ $json.domain }}`
- **Input and output connections:**  
  - Input: `Parse and Clean Business Records`
  - Output: `Remove Duplicate Domains`
- **Version-specific requirements:** Filter node version `2.2`.
- **Edge cases or potential failure types:**  
  - Since the previous Code node already skips missing websites/domains, this is mainly a defensive filter.
  - Invalid domains can still pass if non-empty.
- **Sub-workflow reference:** None.

### Node: Remove Duplicate Domains
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`; eliminates repeated domains.
- **Configuration choices:**  
  - Compare mode: selected fields
  - Comparison field: `domain`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Filter Businesses With a Website`
  - Output: `Batch Businesses for Email Finding`
- **Version-specific requirements:** Remove Duplicates version `2`.
- **Edge cases or potential failure types:**  
  - Case variations or subdomain variations may still be treated as distinct unless normalized upstream.
  - This duplicates logic already partly handled in the Code node.
- **Sub-workflow reference:** None.

---

## 2.6 Email-Finder Batch Preparation and Submission

**Overview:**  
This block chunks the lead list into batches of 10 and transforms each batch into ScraperCity email-finder format. The original business records are preserved so discovered emails can later be merged back.

**Nodes Involved:**  
- Batch Businesses for Email Finding
- Prepare Email Finder Payload
- Start Email Finder Scrape
- Store Email Finder Run ID

### Node: Batch Businesses for Email Finding
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through lead records in groups.
- **Configuration choices:** Batch size `10`.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Inputs: `Remove Duplicate Domains`, `Send Outreach Email` (loop continuation)
  - Output: `Prepare Email Finder Payload`
- **Version-specific requirements:** Split In Batches version `3`.
- **Edge cases or potential failure types:**  
  - Batch size must align with ScraperCity plan or request size limits.
  - Loop behavior depends on downstream node successfully returning control after each batch.
- **Sub-workflow reference:** None.

### Node: Prepare Email Finder Payload
- **Type and technical role:** `n8n-nodes-base.code`; constructs API payload and keeps original records.
- **Configuration choices:**  
  - Builds `contacts` array:
    - `first_name`
    - `last_name`
    - `domain`
  - Stores `originalItems` as the current batch’s raw business records
  - Returns one item containing both structures
- **Key expressions or variables used:** Uses `$input.all()`.
- **Input and output connections:**  
  - Input: `Batch Businesses for Email Finding`
  - Output: `Start Email Finder Scrape`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**  
  - If the batch is empty, output structure may not be useful.
  - Name data may be low quality because it was inferred from business names.
- **Sub-workflow reference:** None.

### Node: Start Email Finder Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`; submits a batch to ScraperCity’s email-finder endpoint.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/email-finder`
  - Authentication: HTTP header auth
  - JSON body contains `contacts`
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.contacts) }}`
- **Input and output connections:**  
  - Input: `Prepare Email Finder Payload`
  - Output: `Store Email Finder Run ID`
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - API key problems
  - Payload too large or malformed
  - Contacts rejected because domains or names do not meet API expectations
- **Sub-workflow reference:** None.

### Node: Store Email Finder Run ID
- **Type and technical role:** `n8n-nodes-base.set`; stores the email-finder `runId` and serializes original batch data.
- **Configuration choices:** Assigns:
  - `emailRunId` from response `runId`
  - `originalItems` as a JSON string from `Prepare Email Finder Payload`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
  - `={{ JSON.stringify($('Prepare Email Finder Payload').item.json.originalItems) }}`
- **Input and output connections:**  
  - Input: `Start Email Finder Scrape`
  - Output: `Wait Before First Email Poll`
- **Version-specific requirements:** Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Large batches increase size of serialized `originalItems`
  - Missing `runId` breaks the polling phase
  - If original item structure changes, later JSON parsing may fail
- **Sub-workflow reference:** None.

---

## 2.7 Email-Finder Polling Loop

**Overview:**  
This block waits for email-finder processing and polls ScraperCity until the batch job is completed. Its logic mirrors the Maps polling phase.

**Nodes Involved:**  
- Wait Before First Email Poll
- Poll Email Finder Status
- Is Email Finder Complete?
- Wait 60s Before Retry Email Poll

### Node: Wait Before First Email Poll
- **Type and technical role:** `n8n-nodes-base.wait`; delays first status check for the email-finder job.
- **Configuration choices:** Wait amount `30` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Store Email Finder Run ID`
  - Output: `Poll Email Finder Status`
- **Version-specific requirements:** Wait node version `1.1`.
- **Edge cases or potential failure types:** Same operational caveats as other Wait nodes.
- **Sub-workflow reference:** None.

### Node: Poll Email Finder Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; queries email-finder job status.
- **Configuration choices:**  
  - Method: `GET`
  - URL built from `emailRunId`
  - Authentication via HTTP header auth
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Email Finder Run ID').item.json.emailRunId }}`
- **Input and output connections:**  
  - Inputs: `Wait Before First Email Poll`, `Wait 60s Before Retry Email Poll`
  - Output: `Is Email Finder Complete?`
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid run ID
  - Rate limiting or temporary API errors
  - Credential problems
- **Sub-workflow reference:** None.

### Node: Is Email Finder Complete?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the email-finder job has finished successfully.
- **Configuration choices:** Condition `status === "SUCCEEDED"` with strict validation.
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Poll Email Finder Status`
  - True output: `Download Email Finder Results`
  - False output: `Wait 60s Before Retry Email Poll`
- **Version-specific requirements:** If node version `2.2`.
- **Edge cases or potential failure types:**  
  - Non-success terminal states are not handled separately and will retry indefinitely.
  - Missing status also triggers retry.
- **Sub-workflow reference:** None.

### Node: Wait 60s Before Retry Email Poll
- **Type and technical role:** `n8n-nodes-base.wait`; delays repeated polling.
- **Configuration choices:** Wait amount `60` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Is Email Finder Complete?` false branch
  - Output: `Poll Email Finder Status`
- **Version-specific requirements:** Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - Infinite loop risk if the job never reaches `SUCCEEDED`.
- **Sub-workflow reference:** None.

---

## 2.8 Email Result Merge and Validation

**Overview:**  
This block downloads discovered email results, maps them to the original business records by domain, and removes any records without usable email addresses. It prepares the final outreach-ready lead set.

**Nodes Involved:**  
- Download Email Finder Results
- Merge Emails With Business Data
- Filter Records With Valid Email

### Node: Download Email Finder Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the completed email-finder output.
- **Configuration choices:**  
  - Method: `GET`
  - URL built from `emailRunId`
  - Authentication via HTTP header auth
- **Key expressions or variables used:**  
  - `=https://app.scrapercity.com/api/downloads/{{ $('Store Email Finder Run ID').item.json.emailRunId }}`
- **Input and output connections:**  
  - Input: `Is Email Finder Complete?` true branch
  - Output: `Merge Emails With Business Data`
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**  
  - Empty results even for successful jobs
  - Response shape differences
  - API auth or download errors
- **Sub-workflow reference:** None.

### Node: Merge Emails With Business Data
- **Type and technical role:** `n8n-nodes-base.code`; joins discovered email records with original business items.
- **Configuration choices:**  
  - Reads email result payload in several possible shapes:
    - direct array
    - `data` array
    - `contacts` array
  - Parses `originalItems` from the earlier Set node
  - Builds a domain-to-email lookup
  - Outputs merged items only for businesses with found emails
- **Key expressions or variables used:**  
  - `$('Store Email Finder Run ID').item.json.originalItems`
  - `JSON.parse(...)`
- **Input and output connections:**  
  - Input: `Download Email Finder Results`
  - Output: `Filter Records With Valid Email`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**  
  - JSON parse failure if `originalItems` is malformed
  - Domain mismatches between source and enrichment results
  - Only one email per domain is stored; multiple contacts are not preserved
  - Records with no found email are silently discarded
- **Sub-workflow reference:** None.

### Node: Filter Records With Valid Email
- **Type and technical role:** `n8n-nodes-base.filter`; keeps only records with an `@` in the email field.
- **Configuration choices:**  
  - Condition: `email` contains `@`
  - Loose validation, case-insensitive
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
- **Input and output connections:**  
  - Input: `Merge Emails With Business Data`
  - Output: `Send Outreach Email`
- **Version-specific requirements:** Filter node version `2.2`.
- **Edge cases or potential failure types:**  
  - This is only a minimal validity check; malformed addresses like `a@b` still pass.
  - Role-based or risky emails are not filtered out.
- **Sub-workflow reference:** None.

---

## 2.9 Gmail Outreach and Batch Continuation

**Overview:**  
This final block sends a personalized plain-text email to each qualified lead. Once complete, the workflow loops back to the batching node to process the next batch of businesses.

**Nodes Involved:**  
- Send Outreach Email

### Node: Send Outreach Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends outbound email using Gmail OAuth2.
- **Configuration choices:**  
  - Recipient: `{{$json.email}}`
  - Subject: taken from configured `emailSubject`
  - Email type: plain text
  - Attribution disabled
  - Message body inserts:
    - `firstName`
    - `businessName`
    - `businessType` fallback
    - `city` fallback to configured location
    - configured sender name and sender email
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `={{ $('Configure Search Parameters').item.json.emailSubject }}`
  - `{{ $json.firstName }}`
  - `{{ $json.businessName }}`
  - `{{ $json.businessType || 'businesses' }}`
  - `{{ $json.city || $('Configure Search Parameters').item.json.locationQuery }}`
  - `{{ $('Configure Search Parameters').item.json.senderName }}`
  - `{{ $('Configure Search Parameters').item.json.senderEmail }}`
- **Input and output connections:**  
  - Input: `Filter Records With Valid Email`
  - Output: `Batch Businesses for Email Finding`
- **Version-specific requirements:** Gmail node version `2.1`; requires Gmail OAuth2 credential.
- **Edge cases or potential failure types:**  
  - Gmail OAuth2 not configured or expired
  - Gmail daily sending limits or anti-spam enforcement
  - Sending from an address inconsistent with `senderEmail` text signature
  - Missing `firstName` or fallback quality issues
  - No send logging, dedupe, or compliance checks built in
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Workflow | Manual Trigger | Manual workflow entry point |  | Configure Search Parameters | ## Workflow entry and config\n\nManual trigger that kicks off the workflow and a Set node that defines all search parameters used downstream. |
| Configure Search Parameters | Set | Defines search and sender parameters | Start Workflow | Start Google Maps Scrape | ## Workflow entry and config\n\nManual trigger that kicks off the workflow and a Set node that defines all search parameters used downstream. |
| Start Google Maps Scrape | HTTP Request | Submits Google Maps scrape job to ScraperCity | Configure Search Parameters | Store Maps Run ID | ## Submit Maps scrape job\n\nSends the Google Maps scrape request to ScraperCity and stores the returned run ID along with context fields for use in polling. |
| Store Maps Run ID | Set | Stores Maps run ID and workflow context | Start Google Maps Scrape | Wait Before First Maps Poll | ## Submit Maps scrape job\n\nSends the Google Maps scrape request to ScraperCity and stores the returned run ID along with context fields for use in polling. |
| Wait Before First Maps Poll | Wait | Initial delay before first Maps status check | Store Maps Run ID | Poll Maps Scrape Status | ## Poll Maps scrape status\n\nWaits before the first poll, checks whether the Maps scrape job is finished, and retries every 60 seconds if it is not yet complete. |
| Poll Maps Scrape Status | HTTP Request | Polls ScraperCity for Maps scrape status | Wait Before First Maps Poll; Wait 60s Before Retry Maps Poll | Is Maps Scrape Complete? | ## Poll Maps scrape status\n\nWaits before the first poll, checks whether the Maps scrape job is finished, and retries every 60 seconds if it is not yet complete. |
| Is Maps Scrape Complete? | If | Checks if Maps scrape succeeded | Poll Maps Scrape Status | Download Maps Results; Wait 60s Before Retry Maps Poll | ## Poll Maps scrape status\n\nWaits before the first poll, checks whether the Maps scrape job is finished, and retries every 60 seconds if it is not yet complete. |
| Wait 60s Before Retry Maps Poll | Wait | Retry delay for Maps polling loop | Is Maps Scrape Complete? | Poll Maps Scrape Status | ## Poll Maps scrape status\n\nWaits before the first poll, checks whether the Maps scrape job is finished, and retries every 60 seconds if it is not yet complete. |
| Download Maps Results | HTTP Request | Downloads completed Maps scrape output | Is Maps Scrape Complete? | Parse and Clean Business Records | ## Download and parse Maps results\n\nDownloads the completed Maps scrape file and parses or cleans the CSV/JSON payload into structured business records. |
| Parse and Clean Business Records | Code | Normalizes and cleans business lead records | Download Maps Results | Filter Businesses With a Website | ## Download and parse Maps results\n\nDownloads the completed Maps scrape file and parses or cleans the CSV/JSON payload into structured business records. |
| Filter Businesses With a Website | Filter | Keeps only leads with a non-empty domain | Parse and Clean Business Records | Remove Duplicate Domains | ## Filter and deduplicate leads\n\nRemoves businesses without a website and eliminates duplicate domains so only unique, contactable leads proceed. |
| Remove Duplicate Domains | Remove Duplicates | Deduplicates leads by domain | Filter Businesses With a Website | Batch Businesses for Email Finding | ## Filter and deduplicate leads\n\nRemoves businesses without a website and eliminates duplicate domains so only unique, contactable leads proceed. |
| Batch Businesses for Email Finding | Split In Batches | Iterates through leads in batches | Remove Duplicate Domains; Send Outreach Email | Prepare Email Finder Payload | ## Batch and prepare email finder\n\nSplits leads into manageable batches and builds the contacts payload required by the ScraperCity email-finder API. |
| Prepare Email Finder Payload | Code | Builds email-finder contacts payload and preserves originals | Batch Businesses for Email Finding | Start Email Finder Scrape | ## Batch and prepare email finder\n\nSplits leads into manageable batches and builds the contacts payload required by the ScraperCity email-finder API. |
| Start Email Finder Scrape | HTTP Request | Submits email-finder batch job to ScraperCity | Prepare Email Finder Payload | Store Email Finder Run ID | ## Submit email finder job\n\nPosts the batch to the ScraperCity email-finder endpoint and stores the returned run ID for subsequent polling. |
| Store Email Finder Run ID | Set | Stores email-finder run ID and serialized original batch | Start Email Finder Scrape | Wait Before First Email Poll | ## Submit email finder job\n\nPosts the batch to the ScraperCity email-finder endpoint and stores the returned run ID for subsequent polling. |
| Wait Before First Email Poll | Wait | Initial delay before first email-finder status check | Store Email Finder Run ID | Poll Email Finder Status | ## Poll email finder status\n\nWaits before the first poll, checks whether the email-finder job is finished, and retries every 60 seconds if still pending. |
| Poll Email Finder Status | HTTP Request | Polls ScraperCity for email-finder job status | Wait Before First Email Poll; Wait 60s Before Retry Email Poll | Is Email Finder Complete? | ## Poll email finder status\n\nWaits before the first poll, checks whether the email-finder job is finished, and retries every 60 seconds if still pending. |
| Is Email Finder Complete? | If | Checks if email-finder job succeeded | Poll Email Finder Status | Download Email Finder Results; Wait 60s Before Retry Email Poll | ## Poll email finder status\n\nWaits before the first poll, checks whether the email-finder job is finished, and retries every 60 seconds if still pending. |
| Wait 60s Before Retry Email Poll | Wait | Retry delay for email-finder polling loop | Is Email Finder Complete? | Poll Email Finder Status | ## Poll email finder status\n\nWaits before the first poll, checks whether the email-finder job is finished, and retries every 60 seconds if still pending. |
| Download Email Finder Results | HTTP Request | Downloads completed email-finder output | Is Email Finder Complete? | Merge Emails With Business Data | ## Download and merge email results\n\nDownloads the completed email-finder results, merges them with the original business records, and filters out any records lacking a valid email address. |
| Merge Emails With Business Data | Code | Merges found emails into original lead records | Download Email Finder Results | Filter Records With Valid Email | ## Download and merge email results\n\nDownloads the completed email-finder results, merges them with the original business records, and filters out any records lacking a valid email address. |
| Filter Records With Valid Email | Filter | Keeps only records containing a basic valid email pattern | Merge Emails With Business Data | Send Outreach Email | ## Download and merge email results\n\nDownloads the completed email-finder results, merges them with the original business records, and filters out any records lacking a valid email address. |
| Send Outreach Email | Gmail | Sends personalized Gmail outreach and continues batch loop | Filter Records With Valid Email | Batch Businesses for Email Finding | ## Send personalised outreach email\n\nSends a personalised Gmail outreach message to each qualifying business and loops back to the batch node to process the next batch. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Scrape Google Maps businesses and send personalized outreach via Gmail`.

2. **Add a Manual Trigger node** named **Start Workflow**.  
   - Type: `Manual Trigger`
   - No special configuration required.

3. **Add a Set node** named **Configure Search Parameters** and connect it after **Start Workflow**.  
   Configure these fields:
   - `businessType` as String, example: `plumbers`
   - `locationQuery` as String, example: `Austin, TX`
   - `maxResults` as Number, example: `50`
   - `senderName` as String, example: `Your Name`
   - `senderEmail` as String, example: `user@example.com`
   - `emailSubject` as String, example: `Quick question about your business`

4. **Create ScraperCity authentication** in n8n.  
   - Credential type: **HTTP Header Auth**
   - Add the header expected by ScraperCity for API access using your API key.
   - Reuse this same credential on all ScraperCity HTTP Request nodes.

5. **Add an HTTP Request node** named **Start Google Maps Scrape** and connect it after **Configure Search Parameters**.  
   Configure:
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/maps`
   - Authentication: Generic Credential Type → HTTP Header Auth
   - Send Body: enabled
   - Specify Body: JSON
   - JSON body:
     - `searchStringsArray` should contain the configured business type
     - `locationQuery` should use the configured location
     - `maxCrawledPlacesPerSearch` should use `maxResults`
   Expressions:
   - `businessType` from current item
   - `locationQuery` from current item
   - `maxResults` from current item

6. **Add a Set node** named **Store Maps Run ID** and connect it after **Start Google Maps Scrape**.  
   Add:
   - `mapsRunId` = response `runId`
   - `businessType` = value from **Configure Search Parameters**
   - `locationQuery` = value from **Configure Search Parameters**
   - `senderName` = value from **Configure Search Parameters**
   - `senderEmail` = value from **Configure Search Parameters**
   - `emailSubject` = value from **Configure Search Parameters**

7. **Add a Wait node** named **Wait Before First Maps Poll** and connect it after **Store Maps Run ID**.  
   - Wait amount: `30` seconds

8. **Add an HTTP Request node** named **Poll Maps Scrape Status** and connect it after **Wait Before First Maps Poll**.  
   Configure:
   - Method: `GET`
   - URL expression using `mapsRunId`:  
     `https://app.scrapercity.com/api/v1/scrape/status/<mapsRunId>`
   - Authentication: same ScraperCity credential

9. **Add an If node** named **Is Maps Scrape Complete?** and connect it after **Poll Maps Scrape Status**.  
   Condition:
   - Left value: response field `status`
   - Operator: equals
   - Right value: `SUCCEEDED`

10. **Add a Wait node** named **Wait 60s Before Retry Maps Poll** for the false branch of the Maps status check.  
    - Wait amount: `60` seconds  
    Connect:
    - False output of **Is Maps Scrape Complete?** → **Wait 60s Before Retry Maps Poll**
    - **Wait 60s Before Retry Maps Poll** → **Poll Maps Scrape Status**

11. **Add an HTTP Request node** named **Download Maps Results** for the true branch of the Maps status check.  
    Configure:
    - Method: `GET`
    - URL expression:  
      `https://app.scrapercity.com/api/downloads/<mapsRunId>`
    - Authentication: same ScraperCity credential

12. **Add a Code node** named **Parse and Clean Business Records** and connect it after **Download Maps Results**.  
    Implement logic that:
    - Accepts direct arrays, nested `data` arrays, or a string payload
    - If string, attempts CSV parsing
    - Extracts normalized business fields:
      - `businessName`
      - `website`
      - `domain`
      - `phone`
      - `address`
      - `city`
      - `firstName`
      - `lastName`
    - Skips records with no business name or no website
    - Extracts `domain` from the website URL
    - Deduplicates domains using a set
    - Returns one output item per business

13. **Add a Filter node** named **Filter Businesses With a Website** and connect it after the Code node.  
    Condition:
    - `domain` is not empty

14. **Add a Remove Duplicates node** named **Remove Duplicate Domains** and connect it after the filter.  
    Configure:
    - Compare selected fields
    - Field to compare: `domain`

15. **Add a Split In Batches node** named **Batch Businesses for Email Finding** and connect it after **Remove Duplicate Domains**.  
    Configure:
    - Batch size: `10`  
    Adjust this number if your ScraperCity plan or API constraints require smaller or larger batches.

16. **Add a Code node** named **Prepare Email Finder Payload** and connect it after **Batch Businesses for Email Finding**.  
    Logic should:
    - Read all current batch items
    - Build a `contacts` array with:
      - `first_name`
      - `last_name`
      - `domain`
    - Also preserve the original batch items under `originalItems`
    - Return a single item containing both `contacts` and `originalItems`

17. **Add an HTTP Request node** named **Start Email Finder Scrape** and connect it after **Prepare Email Finder Payload**.  
    Configure:
    - Method: `POST`
    - URL: `https://app.scrapercity.com/api/v1/scrape/email-finder`
    - Authentication: ScraperCity credential
    - Body type: JSON
    - Body should send the `contacts` array

18. **Add a Set node** named **Store Email Finder Run ID** and connect it after **Start Email Finder Scrape**.  
    Add:
    - `emailRunId` = response `runId`
    - `originalItems` = JSON string of `originalItems` from **Prepare Email Finder Payload**

19. **Add a Wait node** named **Wait Before First Email Poll** and connect it after **Store Email Finder Run ID**.  
    - Wait amount: `30` seconds

20. **Add an HTTP Request node** named **Poll Email Finder Status** and connect it after **Wait Before First Email Poll**.  
    Configure:
    - Method: `GET`
    - URL expression using `emailRunId`:  
      `https://app.scrapercity.com/api/v1/scrape/status/<emailRunId>`
    - Authentication: ScraperCity credential

21. **Add an If node** named **Is Email Finder Complete?** and connect it after **Poll Email Finder Status**.  
    Condition:
    - `status` equals `SUCCEEDED`

22. **Add a Wait node** named **Wait 60s Before Retry Email Poll** for the false branch.  
    - Wait amount: `60` seconds  
    Connect:
    - False output of **Is Email Finder Complete?** → **Wait 60s Before Retry Email Poll**
    - **Wait 60s Before Retry Email Poll** → **Poll Email Finder Status**

23. **Add an HTTP Request node** named **Download Email Finder Results** for the true branch.  
    Configure:
    - Method: `GET`
    - URL expression:  
      `https://app.scrapercity.com/api/downloads/<emailRunId>`
    - Authentication: ScraperCity credential

24. **Add a Code node** named **Merge Emails With Business Data** and connect it after **Download Email Finder Results**.  
    Logic should:
    - Read the downloaded result payload
    - Support array, `data`, or `contacts` response shapes
    - Parse `originalItems` from **Store Email Finder Run ID**
    - Build a domain-to-email lookup
    - Merge emails into original business records
    - Output only records where an email was found

25. **Add a Filter node** named **Filter Records With Valid Email** and connect it after the merge node.  
    Condition:
    - `email` contains `@`

26. **Create Gmail OAuth2 credentials** in n8n.  
    - Credential type: **Gmail OAuth2**
    - Authorize the Gmail account that will send the outreach emails
    - Ensure the Gmail account is allowed for the intended sending volume and complies with Google sending limits

27. **Add a Gmail node** named **Send Outreach Email** and connect it after **Filter Records With Valid Email**.  
    Configure:
    - Operation: send email
    - To: `{{$json.email}}`
    - Subject: from **Configure Search Parameters** `emailSubject`
    - Email type: plain text
    - Disable attribution append if desired
    - Body should reference:
      - recipient first name
      - business name
      - business type or fallback text
      - city or configured location
      - sender name
      - sender email

28. **Loop back for batch continuation** by connecting **Send Outreach Email** back to **Batch Businesses for Email Finding**.  
    This allows n8n to continue with the next batch after processing the current one.

29. **Optionally add sticky notes** to document each section:
    - Entry and config
    - Maps scrape submit
    - Maps polling
    - Maps download/parse
    - Lead filtering
    - Email-finder batching
    - Email-finder submit
    - Email-finder polling
    - Email merge
    - Gmail sending

30. **Test incrementally**:
    - First test only through the Maps submit and poll flow
    - Then test parsing and domain extraction
    - Then test one email-finder batch
    - Finally test Gmail sending with a small self-controlled list

31. **Recommended hardening before production use**:
    - Add explicit failure handling for ScraperCity statuses like `FAILED`
    - Add retry limits to both polling loops
    - Add lead logging to Google Sheets, Airtable, or a database
    - Add suppression logic to avoid emailing the same domain twice
    - Add stronger email validation and compliance checks
    - Add throttling to Gmail sends if volume increases

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Scrape Google Maps businesses and send personalized outreach via Gmail | Workflow title |
| The workflow is triggered manually and configured with search parameters (business type, location, max results, sender name). | Overall workflow behavior |
| A Google Maps scrape job is submitted to ScraperCity and polled until complete, with a 60-second retry loop if still in progress. | Overall workflow behavior |
| Results are downloaded, parsed, filtered for businesses with websites, and deduplicated by domain. | Overall workflow behavior |
| Businesses are batched and submitted to ScraperCity's email-finder API, which is polled in a similar retry loop until complete. | Overall workflow behavior |
| Email finder results are downloaded and merged back with the original business records, then filtered for valid emails. | Overall workflow behavior |
| A personalised outreach email is sent via Gmail for each qualifying business, and the batch loop continues until all businesses are processed. | Overall workflow behavior |
| Create a ScraperCity account and obtain your API key at app.scrapercity.com | ScraperCity setup |
| Add your ScraperCity API key to the HTTP Request nodes (Start Google Maps Scrape, Poll Maps Scrape Status, Download Maps Results, Start Email Finder Scrape, Poll Email Finder Status, Download Email Finder Results) | Credential setup |
| Connect your Gmail account to n8n via OAuth2 credential for the Send Outreach Email node | Credential setup |
| Update the Configure Search Parameters node with your desired businessType, locationQuery, maxResults, and senderName values | Configuration guidance |
| Adjust the batch size in the Batch Businesses for Email Finding node to match your ScraperCity plan limits | Capacity planning |
| Customise the email body in the Send Outreach Email node with your personalised outreach message template | Outreach customization |
| Adjust wait durations in the Wait nodes to match ScraperCity's typical job completion times for your plan. | Performance tuning |
| Add a Google Sheets or database node after Send Outreach Email to log contacted businesses. | Suggested extension |
| Modify the filter conditions in Filter Businesses With a Website and Filter Records With Valid Email to tighten or relax lead quality criteria. | Suggested extension |