Scrape Shopify store leads with ScraperCity and push verified contacts to HubSpot CRM

https://n8nworkflows.xyz/workflows/scrape-shopify-store-leads-with-scrapercity-and-push-verified-contacts-to-hubspot-crm-14032


# Scrape Shopify store leads with ScraperCity and push verified contacts to HubSpot CRM

# 1. Workflow Overview

This workflow manually launches a lead-generation pipeline that targets **Shopify stores**, enriches the results with **email validation from ScraperCity**, and then **upserts verified contacts into HubSpot CRM**.

Its main use case is outbound prospecting or CRM enrichment for teams that want to:
- scrape Shopify store leads by geography,
- keep only leads with email addresses,
- validate those emails in bulk,
- and push only valid contacts into HubSpot.

The workflow is organized into the following logical blocks:

## 1.1 Trigger and Search Configuration
A manual trigger starts the workflow. A Set node defines the search criteria used for the Shopify lead scrape, such as country, number of leads, and whether emails and phones should be included.

## 1.2 Scrape Job Submission
The workflow sends a POST request to ScraperCity to start a Shopify store lead scrape and stores the returned `runId` for later polling and download.

## 1.3 Scrape Status Polling Loop
A wait-and-retry loop checks the scrape job status until ScraperCity reports the scrape as `SUCCEEDED`.

## 1.4 Download and Parse Scraped Leads
Once the scrape is finished, the workflow downloads the results and parses them into structured lead items, supporting multiple possible response shapes.

## 1.5 Lead Filtering and Deduplication
The lead list is reduced to records that contain an email address, then deduplicated on the email field.

## 1.6 Email Validation Job Submission
All deduplicated email addresses are collected into one array and sent in bulk to ScraperCity’s email validation endpoint. The returned validation `runId` is stored.

## 1.7 Validation Status Polling Loop
A second wait-and-retry loop checks the validation job status until it completes successfully.

## 1.8 Download and Merge Validation Results
The validation results are downloaded, normalized, and merged back into the original lead records using email as the join key.

## 1.9 Valid Lead Filtering and HubSpot Upsert
Only leads with accepted validation statuses are kept, and each one is created or updated as a HubSpot contact.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Search Configuration

**Overview:**  
This block starts the workflow manually and defines the core scraping parameters. These values drive the initial ScraperCity request and determine the market and scope of the lead search.

**Nodes Involved:**  
- When clicking Execute workflow
- Configure Search Parameters

### Node: When clicking Execute workflow
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for the workflow.
- **Configuration choices:**  
  No parameters are configured; execution begins when the user manually runs the workflow.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Configure Search Parameters`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  No external failure mode. Only usable through manual execution; not suitable as-is for scheduled automation.
- **Sub-workflow reference:**  
  None.

### Node: Configure Search Parameters
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a structured configuration payload for the scrape request.
- **Configuration choices:**  
  Assigns:
  - `countryCode = "US"`
  - `totalLeads = 200`
  - `includeEmails = true`
  - `includePhones = true`
- **Key expressions or variables used:**  
  Static assignments only.
- **Input and output connections:**  
  - Input: `When clicking Execute workflow`
  - Output: `Start Shopify Store Scrape`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  Low risk. Main issue is invalid business assumptions rather than technical failure, such as unsupported country codes or excessively high lead volume causing slower jobs or API-side rejection.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Scrape Job Submission

**Overview:**  
This block sends the Shopify lead scrape request to ScraperCity and extracts the returned run identifier. That run ID is required by the polling and download stages.

**Nodes Involved:**  
- Start Shopify Store Scrape
- Store Scrape Run ID

### Node: Start Shopify Store Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an authenticated POST request to ScraperCity to launch a scrape job.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/store-leads`
  - Body format: JSON
  - Authentication: generic credential type using HTTP header auth
  - JSON body contains:
    - `platform: "shopify"`
    - `countryCode` from workflow config
    - `emails` from `includeEmails`
    - `phones` from `includePhones`
    - `totalLeads` from config
- **Key expressions or variables used:**  
  - `{{ $json.countryCode }}`
  - `{{ $json.includeEmails }}`
  - `{{ $json.includePhones }}`
  - `{{ $json.totalLeads }}`
- **Input and output connections:**  
  - Input: `Configure Search Parameters`
  - Output: `Store Scrape Run ID`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing or invalid ScraperCity API credential
  - 401/403 authentication failures
  - 4xx validation errors if body fields are not accepted
  - 5xx API instability
  - Timeout for slow upstream API
  - If the response format changes and no `runId` is returned, downstream logic will break
- **Sub-workflow reference:**  
  None.

### Node: Store Scrape Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts the scrape job identifier into a stable field.
- **Configuration choices:**  
  Assigns `runId = {{$json.runId}}`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Start Shopify Store Scrape`
  - Output: `Wait Before First Poll`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  If `runId` is absent, empty, or differently named, later polling/download URLs will be malformed.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Scrape Status Polling Loop

**Overview:**  
This block repeatedly checks whether the scrape job has finished. It pauses between attempts and only moves forward when the status equals `SUCCEEDED`.

**Nodes Involved:**  
- Wait Before First Poll
- Poll Loop Controller
- Check Scrape Status
- Is Scrape Complete?
- Wait 60 Seconds Before Retry

### Node: Wait Before First Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays the first status check to give the scrape job time to begin.
- **Configuration choices:**  
  Wait amount: `60`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Store Scrape Run ID`
  - Output: `Poll Loop Controller`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  Too-short delay may cause unnecessary polling; too-long delay slows execution.
- **Sub-workflow reference:**  
  None.

### Node: Poll Loop Controller
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Used here as a loop controller to repeatedly route execution through the scrape status check path.
- **Configuration choices:**  
  `reset: false`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Wait Before First Poll`, `Wait 60 Seconds Before Retry`
  - Output: `Check Scrape Status`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  This looping pattern works, but if the incoming item state becomes inconsistent or the job never reaches `SUCCEEDED`, execution can continue indefinitely.
- **Sub-workflow reference:**  
  None.

### Node: Check Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries ScraperCity for the current status of the scrape job.
- **Configuration choices:**  
  - Method defaults to `GET`
  - URL references the stored scrape run ID:
    `https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Scrape Run ID').item.json.runId }}`
  - Authentication: generic credential type with HTTP header auth
- **Key expressions or variables used:**  
  - `{{ $('Store Scrape Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input: `Poll Loop Controller`
  - Output: `Is Scrape Complete?`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid/missing run ID
  - API auth issues
  - API returns statuses other than `SUCCEEDED` such as failed/cancelled states that are not explicitly handled
  - Network or timeout failures
- **Sub-workflow reference:**  
  None.

### Node: Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the status returned by ScraperCity is exactly `SUCCEEDED`.
- **Configuration choices:**  
  String equality comparison:
  - Left: `{{$json.status}}`
  - Right: `SUCCEEDED`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Check Scrape Status`
  - True output: `Download Scraped Store Leads`
  - False output: `Wait 60 Seconds Before Retry`
- **Version-specific requirements:**  
  Type version `2.2`, conditions version `2`, strict type validation.
- **Edge cases or potential failure types:**  
  - Any non-`SUCCEEDED` status loops forever, including terminal failure states if the API uses them
  - Case sensitivity means `succeeded` would not match
- **Sub-workflow reference:**  
  None.

### Node: Wait 60 Seconds Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses before sending another scrape status request.
- **Configuration choices:**  
  Wait amount: `60`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Is Scrape Complete?` (false branch)
  - Output: `Poll Loop Controller`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  Same operational tradeoff as the initial wait: too frequent increases API pressure; too slow increases latency.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Download and Parse Scraped Leads

**Overview:**  
After the scrape completes, this block downloads the output and converts it into individual lead items. The parser is defensive and can handle array-like JSON responses as well as CSV-like text payloads.

**Nodes Involved:**  
- Download Scraped Store Leads
- Parse CSV Lead Data

### Node: Download Scraped Store Leads
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed scrape output from ScraperCity.
- **Configuration choices:**  
  - URL: `https://app.scrapercity.com/api/downloads/{{ $('Store Scrape Run ID').item.json.runId }}`
  - Authentication: generic credential type with HTTP header auth
- **Key expressions or variables used:**  
  - `{{ $('Store Scrape Run ID').item.json.runId }}`
- **Input and output connections:**  
  - Input: `Is Scrape Complete?` (true branch)
  - Output: `Parse CSV Lead Data`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Download endpoint unavailable
  - Empty file/content
  - Unexpected MIME type or body shape
  - Invalid run ID or expired download
- **Sub-workflow reference:**  
  None.

### Node: Parse CSV Lead Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes the downloaded content into one n8n item per lead.
- **Configuration choices:**  
  The JavaScript logic:
  1. Reads the first input item via `$input.first().json`
  2. If the response is already an array, emits each element
  3. If `raw.data` is an array, emits each element
  4. Otherwise tries to parse CSV text from:
     - raw string itself,
     - `raw.body`,
     - `raw.csv`
  5. If no parseable content is found, returns a single error item
  6. Uses a simple quoted-field-aware CSV parser
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - Input: `Download Scraped Store Leads`
  - Output: `Filter Records with Emails`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - CSV parser is basic and may break on multiline quoted values
  - Unexpected header names will affect downstream mappings
  - If parsing fails, an error item is emitted instead of stopping execution; downstream nodes may process it unless filtered out
  - If the HTTP Request node returns binary data rather than JSON/body text, this code may not parse it correctly without additional configuration
- **Sub-workflow reference:**  
  None.

---

## 2.5 Lead Filtering and Deduplication

**Overview:**  
This block removes unusable records and reduces duplicate entries before email validation. It ensures only unique email-bearing leads proceed to the validation stage.

**Nodes Involved:**  
- Filter Records with Emails
- Remove Duplicate Leads

### Node: Filter Records with Emails
- **Type and technical role:** `n8n-nodes-base.filter`  
  Keeps only items where the `email` field is not empty.
- **Configuration choices:**  
  Condition:
  - `email` is not empty
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
- **Input and output connections:**  
  - Input: `Parse CSV Lead Data`
  - Output: `Remove Duplicate Leads`
- **Version-specific requirements:**  
  Type version `2.2`, loose validation, case insensitive.
- **Edge cases or potential failure types:**  
  - If scraped output uses a different field name than `email`, all records will be filtered out
  - Invalid email format is not checked here; only non-empty presence
- **Sub-workflow reference:**  
  None.

### Node: Remove Duplicate Leads
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Deduplicates items based on selected field comparison.
- **Configuration choices:**  
  - Compare mode: selected fields
  - Field compared: `email`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Filter Records with Emails`
  - Output: `Collect Emails for Validation`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Email case sensitivity/format normalization depends on node behavior; `Test@x.com` and `test@x.com` may or may not collapse depending on implementation
  - If email contains whitespace variations, duplicates may survive unless normalized beforehand
- **Sub-workflow reference:**  
  None.

---

## 2.6 Email Validation Job Submission

**Overview:**  
This block converts the filtered lead stream into a bulk validation request. It also preserves the original lead records so they can be rejoined with the validation output later.

**Nodes Involved:**  
- Collect Emails for Validation
- Start Email Validation
- Store Validation Run ID

### Node: Collect Emails for Validation
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates all deduplicated leads into a single item containing:
  - an `emails` array for bulk validation,
  - a `leads` array for later merging.
- **Configuration choices:**  
  JavaScript collects all input items:
  - `emails = $input.all().map(item => item.json.email).filter(Boolean)`
  - `leads = $input.all().map(item => item.json)`
- **Key expressions or variables used:**  
  - `$input.all()`
- **Input and output connections:**  
  - Input: `Remove Duplicate Leads`
  - Output: `Start Email Validation`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Large lead volumes may produce a large in-memory array
  - Empty input creates an empty email list, which may cause API rejection or pointless validation job creation
- **Sub-workflow reference:**  
  None.

### Node: Start Email Validation
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Starts a bulk email validation job in ScraperCity.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/email-validator`
  - JSON body with `emails`
  - Authentication: generic credential type with HTTP header auth
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.emails) }}`
- **Input and output connections:**  
  - Input: `Collect Emails for Validation`
  - Output: `Store Validation Run ID`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Empty email array
  - Auth and quota errors
  - API body format mismatch if the endpoint expects a raw array instead of a stringified array embedded in JSON
  - Timeout for larger payloads
- **Sub-workflow reference:**  
  None.

### Node: Store Validation Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores the validation job identifier under `validationRunId`.
- **Configuration choices:**  
  Assigns `validationRunId = {{$json.runId}}`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Start Email Validation`
  - Output: `Wait Before Validation Poll`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  Missing `runId` breaks the validation polling and download path.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Validation Status Polling Loop

**Overview:**  
This block mirrors the scrape polling loop but for the email validation job. It retries every 30 seconds until ScraperCity reports completion.

**Nodes Involved:**  
- Wait Before Validation Poll
- Validation Poll Loop Controller
- Check Validation Status
- Is Validation Complete?
- Wait 30 Seconds Before Validation Retry

### Node: Wait Before Validation Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays the first validation status check.
- **Configuration choices:**  
  Wait amount: `30`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Store Validation Run ID`
  - Output: `Validation Poll Loop Controller`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  Similar scheduling tradeoff as the scrape wait.
- **Sub-workflow reference:**  
  None.

### Node: Validation Poll Loop Controller
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Used as the loop controller for repeated validation status checks.
- **Configuration choices:**  
  `reset: false`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Wait Before Validation Poll`, `Wait 30 Seconds Before Validation Retry`
  - Output: `Check Validation Status`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  Same as the first polling controller: if the job never reaches the expected status, the loop can continue indefinitely.
- **Sub-workflow reference:**  
  None.

### Node: Check Validation Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Checks the current status of the email validation job.
- **Configuration choices:**  
  URL:
  `https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Validation Run ID').item.json.validationRunId }}`
  with HTTP header auth.
- **Key expressions or variables used:**  
  - `{{ $('Store Validation Run ID').item.json.validationRunId }}`
- **Input and output connections:**  
  - Input: `Validation Poll Loop Controller`
  - Output: `Is Validation Complete?`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid run ID
  - Auth failure
  - Unhandled failed/cancelled terminal statuses
  - API instability/timeouts
- **Sub-workflow reference:**  
  None.

### Node: Is Validation Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches when validation status is exactly `SUCCEEDED`.
- **Configuration choices:**  
  Strict string equality:
  - `{{$json.status}} == "SUCCEEDED"`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Check Validation Status`
  - True output: `Download Validation Results`
  - False output: `Wait 30 Seconds Before Validation Retry`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  Terminal failure states are not separately handled and therefore fall into retry behavior.
- **Sub-workflow reference:**  
  None.

### Node: Wait 30 Seconds Before Validation Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays repeated validation polling attempts.
- **Configuration choices:**  
  Wait amount: `30`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Is Validation Complete?` (false branch)
  - Output: `Validation Poll Loop Controller`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  No special technical issue beyond pacing and runtime duration.
- **Sub-workflow reference:**  
  None.

---

## 2.8 Download and Merge Validation Results

**Overview:**  
This block downloads the validator output, normalizes its structure, and merges the validation metadata back into the original lead records using email matching.

**Nodes Involved:**  
- Download Validation Results
- Merge Leads with Validation Results

### Node: Download Validation Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed validation output.
- **Configuration choices:**  
  URL:
  `https://app.scrapercity.com/api/downloads/{{ $('Store Validation Run ID').item.json.validationRunId }}`
  with HTTP header auth.
- **Key expressions or variables used:**  
  - `{{ $('Store Validation Run ID').item.json.validationRunId }}`
- **Input and output connections:**  
  - Input: `Is Validation Complete?` (true branch)
  - Output: `Merge Leads with Validation Results`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Empty download
  - Unexpected schema
  - Invalid run ID or stale link
- **Sub-workflow reference:**  
  None.

### Node: Merge Leads with Validation Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Reconstructs final lead objects by joining original lead data with validation statuses.
- **Configuration choices:**  
  The JavaScript logic:
  1. Reads the validation response from `$input.first().json`
  2. Accepts validation arrays in one of these forms:
     - raw array
     - `data` array
     - `results` array
  3. Builds a lookup map keyed by normalized email
  4. Retrieves original leads from `Collect Emails for Validation`
  5. If no original leads are available, emits validation results directly
  6. Merges validation fields into each lead:
     - `emailStatus`
     - `isCatchAll`
     - `mxFound`
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `$('Collect Emails for Validation').first()`
- **Input and output connections:**  
  - Input: `Download Validation Results`
  - Output: `Filter Valid Emails Only`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If validation response schema changes, the parser may produce an empty map
  - Email matching depends on consistent field naming and normalization
  - If original lead retrieval fails, the node degrades to outputting validation results directly, which may not contain fields expected by HubSpot
  - If multiple validation entries exist for the same email, later entries overwrite earlier ones
- **Sub-workflow reference:**  
  None.

---

## 2.9 Valid Lead Filtering and HubSpot Upsert

**Overview:**  
This final block keeps only leads whose email validation result is acceptable and then sends them to HubSpot as contacts. Existing contacts are updated based on the email address.

**Nodes Involved:**  
- Filter Valid Emails Only
- Create or Update HubSpot Contact

### Node: Filter Valid Emails Only
- **Type and technical role:** `n8n-nodes-base.filter`  
  Retains only records whose merged email validation status is considered acceptable.
- **Configuration choices:**  
  OR conditions:
  - `emailStatus == "valid"`
  - `emailStatus == "catch_all"`
  - `emailStatus == "VALID"`
- **Key expressions or variables used:**  
  - `={{ $json.emailStatus }}`
- **Input and output connections:**  
  - Input: `Merge Leads with Validation Results`
  - Output: `Create or Update HubSpot Contact`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Other potentially acceptable statuses are excluded
  - Mixed capitalization is only partially handled
  - `catch_all` can include riskier addresses, depending on business policy
- **Sub-workflow reference:**  
  None.

### Node: Create or Update HubSpot Contact
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Upserts contacts in HubSpot CRM using email as the key.
- **Configuration choices:**  
  - Authentication: `appToken`
  - Main email field: `{{$json.email}}`
  - Additional mapped fields:
    - `city`
    - `country`
    - `message`
    - `firstName`
    - `websiteUrl`
    - `companyName`
  - The `message` field is built from validation metadata and social/store attributes.
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `={{ $json.city || '' }}`
  - `={{ $json.country || $json.countryCode || '' }}`
  - `={{ $json.first_name || $json.firstName || '' }}`
  - `={{ $json.website || $json.domain || $json.url || '' }}`
  - `={{ $json.store_name || $json.company || $json.businessName || '' }}`
  - Multiline message expression containing:
    - `{{ $json.emailStatus }}`
    - `{{ $json.isCatchAll }}`
    - `{{ $json.mxFound }}`
    - `{{ $json.instagram || '' }}`
    - `{{ $json.facebook || '' }}`
    - `{{ $json.twitter || '' }}`
- **Input and output connections:**  
  - Input: `Filter Valid Emails Only`
  - Output: none
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Authentication misconfiguration in HubSpot
  - Property mismatch if HubSpot account does not support one of the mapped fields
  - Validation errors for field format, especially website URL or custom message/property expectations
  - Rate limiting if many contacts are processed
  - Contact upsert behavior depends on actual node operation defaults and available credentials
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking Execute workflow | Manual Trigger | Manual workflow entry point |  | Configure Search Parameters | ## Workflow trigger and config\n\nManually triggers the workflow and sets the core search parameters — country code, lead count, and contact type flags — that drive the scrape job. |
| Configure Search Parameters | Set | Defines scrape parameters | When clicking Execute workflow | Start Shopify Store Scrape | ## Workflow trigger and config\n\nManually triggers the workflow and sets the core search parameters — country code, lead count, and contact type flags — that drive the scrape job. |
| Start Shopify Store Scrape | HTTP Request | Starts Shopify lead scrape job in ScraperCity | Configure Search Parameters | Store Scrape Run ID | ## Initiate scrape job\n\nSubmits the scrape request to ScraperCity and stores the returned run ID for subsequent polling. |
| Store Scrape Run ID | Set | Stores scrape run ID | Start Shopify Store Scrape | Wait Before First Poll | ## Initiate scrape job\n\nSubmits the scrape request to ScraperCity and stores the returned run ID for subsequent polling. |
| Wait Before First Poll | Wait | Delays first scrape status poll | Store Scrape Run ID | Poll Loop Controller | ## Scrape job polling loop\n\nWaits before the first status check, then polls the scrape job status in a loop — retrying every 60 seconds — until the scrape is reported as complete. |
| Poll Loop Controller | Split In Batches | Controls scrape polling loop | Wait Before First Poll; Wait 60 Seconds Before Retry | Check Scrape Status | ## Scrape job polling loop\n\nWaits before the first status check, then polls the scrape job status in a loop — retrying every 60 seconds — until the scrape is reported as complete. |
| Check Scrape Status | HTTP Request | Retrieves scrape job status | Poll Loop Controller | Is Scrape Complete? | ## Scrape job polling loop\n\nWaits before the first status check, then polls the scrape job status in a loop — retrying every 60 seconds — until the scrape is reported as complete. |
| Is Scrape Complete? | If | Branches on scrape completion status | Check Scrape Status | Download Scraped Store Leads; Wait 60 Seconds Before Retry | ## Scrape job polling loop\n\nWaits before the first status check, then polls the scrape job status in a loop — retrying every 60 seconds — until the scrape is reported as complete. |
| Wait 60 Seconds Before Retry | Wait | Delays next scrape poll attempt | Is Scrape Complete? | Poll Loop Controller | ## Scrape job polling loop\n\nWaits before the first status check, then polls the scrape job status in a loop — retrying every 60 seconds — until the scrape is reported as complete. |
| Download Scraped Store Leads | HTTP Request | Downloads completed scraped leads | Is Scrape Complete? | Parse CSV Lead Data | ## Download and parse leads\n\nDownloads the completed scrape results as a CSV file and parses it into structured lead records ready for filtering. |
| Parse CSV Lead Data | Code | Parses downloaded lead data into items | Download Scraped Store Leads | Filter Records with Emails | ## Download and parse leads\n\nDownloads the completed scrape results as a CSV file and parses it into structured lead records ready for filtering. |
| Filter Records with Emails | Filter | Keeps only records with an email | Parse CSV Lead Data | Remove Duplicate Leads | ## Filter and deduplicate leads\n\nRemoves records without email addresses and eliminates duplicate leads to produce a clean, unique list for validation. |
| Remove Duplicate Leads | Remove Duplicates | Deduplicates leads by email | Filter Records with Emails | Collect Emails for Validation | ## Filter and deduplicate leads\n\nRemoves records without email addresses and eliminates duplicate leads to produce a clean, unique list for validation. |
| Collect Emails for Validation | Code | Aggregates emails and preserves original leads | Remove Duplicate Leads | Start Email Validation | ## Initiate email validation\n\nAggregates all lead emails into a single bulk array and submits them to ScraperCity's email validation API, storing the returned validation run ID. |
| Start Email Validation | HTTP Request | Starts bulk email validation job | Collect Emails for Validation | Store Validation Run ID | ## Initiate email validation\n\nAggregates all lead emails into a single bulk array and submits them to ScraperCity's email validation API, storing the returned validation run ID. |
| Store Validation Run ID | Set | Stores validation run ID | Start Email Validation | Wait Before Validation Poll | ## Initiate email validation\n\nAggregates all lead emails into a single bulk array and submits them to ScraperCity's email validation API, storing the returned validation run ID. |
| Wait Before Validation Poll | Wait | Delays first validation status poll | Store Validation Run ID | Validation Poll Loop Controller | ## Validation job polling loop\n\nWaits before the first validation status check, then polls in a loop — retrying every 30 seconds — until the email validation job is complete. |
| Validation Poll Loop Controller | Split In Batches | Controls validation polling loop | Wait Before Validation Poll; Wait 30 Seconds Before Validation Retry | Check Validation Status | ## Validation job polling loop\n\nWaits before the first validation status check, then polls in a loop — retrying every 30 seconds — until the email validation job is complete. |
| Check Validation Status | HTTP Request | Retrieves validation job status | Validation Poll Loop Controller | Is Validation Complete? | ## Validation job polling loop\n\nWaits before the first validation status check, then polls in a loop — retrying every 30 seconds — until the email validation job is complete. |
| Is Validation Complete? | If | Branches on validation completion status | Check Validation Status | Download Validation Results; Wait 30 Seconds Before Validation Retry | ## Validation job polling loop\n\nWaits before the first validation status check, then polls in a loop — retrying every 30 seconds — until the email validation job is complete. |
| Wait 30 Seconds Before Validation Retry | Wait | Delays next validation poll attempt | Is Validation Complete? | Validation Poll Loop Controller | ## Validation job polling loop\n\nWaits before the first validation status check, then polls in a loop — retrying every 30 seconds — until the email validation job is complete. |
| Download Validation Results | HTTP Request | Downloads email validation results | Is Validation Complete? | Merge Leads with Validation Results | ## Download and merge validation results\n\nDownloads the validation results, merges them back with the original lead records by email, and filters out any leads with invalid or unverified emails. |
| Merge Leads with Validation Results | Code | Joins leads with validation metadata | Download Validation Results | Filter Valid Emails Only | ## Download and merge validation results\n\nDownloads the validation results, merges them back with the original lead records by email, and filters out any leads with invalid or unverified emails. |
| Filter Valid Emails Only | Filter | Keeps only validated/accepted email statuses | Merge Leads with Validation Results | Create or Update HubSpot Contact | ## Download and merge validation results\n\nDownloads the validation results, merges them back with the original lead records by email, and filters out any leads with invalid or unverified emails. |
| Create or Update HubSpot Contact | HubSpot | Upserts verified contacts into HubSpot | Filter Valid Emails Only |  | ## Push contacts to HubSpot\n\nUpserts each verified lead as a contact in HubSpot CRM, creating new records or updating existing ones based on email address. |
| Sticky Note | Sticky Note | Documentation/comment node |  |  | ## Scrape Shopify store leads and push verified contacts into HubSpot CRM\n\n### How it works\n\n1. The workflow is triggered manually and configures search parameters (country, lead count, contact types) before initiating a Shopify store scrape via the ScraperCity API.\n2. It polls the scrape job in a loop, waiting 60 seconds between retries, until the job completes and the scraped lead data can be downloaded.\n3. The downloaded CSV is parsed, filtered to records containing emails, and deduplicated to produce a clean lead list.\n4. All collected emails are batched and submitted to ScraperCity's email validation API, with a separate polling loop checking for completion.\n5. Once validation results are downloaded, they are merged back with the original lead records and filtered to retain only verified, valid emails.\n6. Each verified lead is upserted as a contact in HubSpot CRM using the HubSpot node.\n\n### Setup steps\n\n- [ ] Create a ScraperCity account and obtain your API key for the scrape and validation endpoints\n- [ ] Add your ScraperCity API key as an n8n credential and reference it in the HTTP Request nodes\n- [ ] Connect your HubSpot account in n8n via the HubSpot credential (OAuth2 or API key)\n- [ ] Update the 'Configure Search Parameters' node with your desired countryCode, totalLeads, includeEmails, and includePhones values\n- [ ] Review the polling wait times in 'Wait Before First Poll' and 'Wait 60 Seconds Before Retry' to match your expected scrape duration\n- [ ] Map the correct HubSpot contact fields in the 'Create or Update HubSpot Contact' node to match your lead data structure\n\n### Customization\n\nAdjust the countryCode and totalLeads parameters to target different markets or volumes. The polling intervals (60s for scraping, 30s for validation) can be tuned based on typical job completion times. The email filter and deduplication steps can be extended to also filter by phone availability or store category. Additional HubSpot properties (e.g., lead source, country) can be mapped in the final upsert node. |
| Sticky Note1 | Sticky Note | Documentation/comment node |  |  | ## Workflow trigger and config\n\nManually triggers the workflow and sets the core search parameters — country code, lead count, and contact type flags — that drive the scrape job. |
| Sticky Note2 | Sticky Note | Documentation/comment node |  |  | ## Initiate scrape job\n\nSubmits the scrape request to ScraperCity and stores the returned run ID for subsequent polling. |
| Sticky Note3 | Sticky Note | Documentation/comment node |  |  | ## Scrape job polling loop\n\nWaits before the first status check, then polls the scrape job status in a loop — retrying every 60 seconds — until the scrape is reported as complete. |
| Sticky Note4 | Sticky Note | Documentation/comment node |  |  | ## Download and parse leads\n\nDownloads the completed scrape results as a CSV file and parses it into structured lead records ready for filtering. |
| Sticky Note5 | Sticky Note | Documentation/comment node |  |  | ## Filter and deduplicate leads\n\nRemoves records without email addresses and eliminates duplicate leads to produce a clean, unique list for validation. |
| Sticky Note6 | Sticky Note | Documentation/comment node |  |  | ## Initiate email validation\n\nAggregates all lead emails into a single bulk array and submits them to ScraperCity's email validation API, storing the returned validation run ID. |
| Sticky Note7 | Sticky Note | Documentation/comment node |  |  | ## Validation job polling loop\n\nWaits before the first validation status check, then polls in a loop — retrying every 30 seconds — until the email validation job is complete. |
| Sticky Note8 | Sticky Note | Documentation/comment node |  |  | ## Download and merge validation results\n\nDownloads the validation results, merges them back with the original lead records by email, and filters out any leads with invalid or unverified emails. |
| Sticky Note9 | Sticky Note | Documentation/comment node |  |  | ## Push contacts to HubSpot\n\nUpserts each verified lead as a contact in HubSpot CRM, creating new records or updating existing ones based on email address. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named something like:  
   `Scrape Shopify store leads and push verified contacts into HubSpot CRM`.

2. **Add a Manual Trigger node**.
   - Node type: **Manual Trigger**
   - Leave default configuration.
   - Rename it to: `When clicking Execute workflow`

3. **Add a Set node** after the trigger.
   - Node type: **Set**
   - Rename to: `Configure Search Parameters`
   - Create these fields:
     - `countryCode` → String → `US`
     - `totalLeads` → Number → `200`
     - `includeEmails` → Boolean → `true`
     - `includePhones` → Boolean → `true`

4. **Connect the Manual Trigger to the Set node**.

5. **Create a ScraperCity credential** for HTTP header authentication.
   - In n8n credentials, create a credential usable by HTTP Request nodes.
   - Use a header-based auth approach, typically an API key header as required by ScraperCity.
   - Apply this credential to all ScraperCity HTTP Request nodes in the workflow.

6. **Add an HTTP Request node** to start the scrape.
   - Node type: **HTTP Request**
   - Rename to: `Start Shopify Store Scrape`
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/store-leads`
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **HTTP Header Auth**
   - Enable body sending
   - Body content type: **JSON**
   - Use this JSON body logic:
     - `platform` = `shopify`
     - `countryCode` from the Set node
     - `emails` from `includeEmails`
     - `phones` from `includePhones`
     - `totalLeads` from `totalLeads`
   - Equivalent expression structure:
     - `countryCode: {{$json.countryCode}}`
     - `emails: {{$json.includeEmails}}`
     - `phones: {{$json.includePhones}}`
     - `totalLeads: {{$json.totalLeads}}`

7. **Connect `Configure Search Parameters` to `Start Shopify Store Scrape`**.

8. **Add a Set node** to capture the scrape run ID.
   - Node type: **Set**
   - Rename to: `Store Scrape Run ID`
   - Add field:
     - `runId` → String → `={{ $json.runId }}`

9. **Connect `Start Shopify Store Scrape` to `Store Scrape Run ID`**.

10. **Add a Wait node** for initial scrape polling delay.
    - Node type: **Wait**
    - Rename to: `Wait Before First Poll`
    - Wait amount: `60` seconds

11. **Connect `Store Scrape Run ID` to `Wait Before First Poll`**.

12. **Add a Split In Batches node** to act as a loop controller.
    - Node type: **Split In Batches**
    - Rename to: `Poll Loop Controller`
    - Keep reset option `false`

13. **Connect `Wait Before First Poll` to `Poll Loop Controller`**.

14. **Add an HTTP Request node** to check scrape status.
    - Rename to: `Check Scrape Status`
    - Method: `GET`
    - URL:
      `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Scrape Run ID').item.json.runId }}`
    - Authentication: same ScraperCity header auth credential

15. **Connect `Poll Loop Controller` to `Check Scrape Status`**.

16. **Add an If node** for scrape completion.
    - Rename to: `Is Scrape Complete?`
    - Condition:
      - Value 1: `={{ $json.status }}`
      - Operator: equals
      - Value 2: `SUCCEEDED`
    - Use strict/case-sensitive matching

17. **Connect `Check Scrape Status` to `Is Scrape Complete?`**.

18. **Add a Wait node** for scrape retry.
    - Rename to: `Wait 60 Seconds Before Retry`
    - Wait amount: `60` seconds

19. **Connect the false output of `Is Scrape Complete?` to `Wait 60 Seconds Before Retry`**.

20. **Connect `Wait 60 Seconds Before Retry` back to `Poll Loop Controller`** to form the polling loop.

21. **Add an HTTP Request node** to download scrape results.
    - Rename to: `Download Scraped Store Leads`
    - Method: `GET`
    - URL:
      `=https://app.scrapercity.com/api/downloads/{{ $('Store Scrape Run ID').item.json.runId }}`
    - Authentication: same ScraperCity credential

22. **Connect the true output of `Is Scrape Complete?` to `Download Scraped Store Leads`**.

23. **Add a Code node** to parse returned lead data.
    - Rename to: `Parse CSV Lead Data`
    - Paste the JavaScript logic from the workflow behavior:
      - Accept already-parsed arrays
      - Accept `data` arrays
      - Otherwise parse CSV from raw string, `body`, or `csv`
      - Emit one item per lead
      - Return an error item if nothing parseable exists

24. **Connect `Download Scraped Store Leads` to `Parse CSV Lead Data`**.

25. **Add a Filter node** to keep only rows with an email.
    - Rename to: `Filter Records with Emails`
    - Condition:
      - `={{ $json.email }}`
      - Operator: not empty

26. **Connect `Parse CSV Lead Data` to `Filter Records with Emails`**.

27. **Add a Remove Duplicates node**.
    - Rename to: `Remove Duplicate Leads`
    - Compare mode: **Selected Fields**
    - Field to compare: `email`

28. **Connect `Filter Records with Emails` to `Remove Duplicate Leads`**.

29. **Add a Code node** to prepare validation payload.
    - Rename to: `Collect Emails for Validation`
    - Use code that:
      - collects all input emails into an `emails` array
      - collects all original lead objects into a `leads` array
      - returns one item containing both arrays

30. **Connect `Remove Duplicate Leads` to `Collect Emails for Validation`**.

31. **Add an HTTP Request node** to start email validation.
    - Rename to: `Start Email Validation`
    - Method: `POST`
    - URL: `https://app.scrapercity.com/api/v1/scrape/email-validator`
    - Authentication: same ScraperCity credential
    - JSON body should contain the email array from the previous node

32. **Connect `Collect Emails for Validation` to `Start Email Validation`**.

33. **Add a Set node** to capture the validation run ID.
    - Rename to: `Store Validation Run ID`
    - Field:
      - `validationRunId` → String → `={{ $json.runId }}`

34. **Connect `Start Email Validation` to `Store Validation Run ID`**.

35. **Add a Wait node** for the first validation status poll.
    - Rename to: `Wait Before Validation Poll`
    - Wait amount: `30` seconds

36. **Connect `Store Validation Run ID` to `Wait Before Validation Poll`**.

37. **Add a Split In Batches node** for the validation loop.
    - Rename to: `Validation Poll Loop Controller`
    - Keep reset option `false`

38. **Connect `Wait Before Validation Poll` to `Validation Poll Loop Controller`**.

39. **Add an HTTP Request node** to check validation job status.
    - Rename to: `Check Validation Status`
    - URL:
      `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Validation Run ID').item.json.validationRunId }}`
    - Authentication: same ScraperCity credential

40. **Connect `Validation Poll Loop Controller` to `Check Validation Status`**.

41. **Add an If node** for validation completion.
    - Rename to: `Is Validation Complete?`
    - Condition:
      - `={{ $json.status }}`
      - equals `SUCCEEDED`

42. **Connect `Check Validation Status` to `Is Validation Complete?`**.

43. **Add a Wait node** for validation retry.
    - Rename to: `Wait 30 Seconds Before Validation Retry`
    - Wait amount: `30` seconds

44. **Connect the false output of `Is Validation Complete?` to `Wait 30 Seconds Before Validation Retry`**.

45. **Connect `Wait 30 Seconds Before Validation Retry` back to `Validation Poll Loop Controller`**.

46. **Add an HTTP Request node** to download validation results.
    - Rename to: `Download Validation Results`
    - URL:
      `=https://app.scrapercity.com/api/downloads/{{ $('Store Validation Run ID').item.json.validationRunId }}`
    - Authentication: same ScraperCity credential

47. **Connect the true output of `Is Validation Complete?` to `Download Validation Results`**.

48. **Add a Code node** to merge leads with validation results.
    - Rename to: `Merge Leads with Validation Results`
    - Logic should:
      - read validation response
      - normalize possible schemas (`array`, `data`, `results`)
      - build a lookup by normalized email
      - retrieve original leads from `Collect Emails for Validation`
      - merge `emailStatus`, `isCatchAll`, and `mxFound` into each lead

49. **Connect `Download Validation Results` to `Merge Leads with Validation Results`**.

50. **Add a Filter node** to keep only accepted validation statuses.
    - Rename to: `Filter Valid Emails Only`
    - OR conditions:
      - `emailStatus = valid`
      - `emailStatus = catch_all`
      - `emailStatus = VALID`

51. **Connect `Merge Leads with Validation Results` to `Filter Valid Emails Only`**.

52. **Create a HubSpot credential** in n8n.
    - Use the HubSpot auth method supported in your environment.
    - The workflow uses `appToken` authentication in the node configuration.
    - Ensure the credential has permission to create/update contacts.

53. **Add a HubSpot node**.
    - Rename to: `Create or Update HubSpot Contact`
    - Configure it to create/update contacts using email.
    - Main email expression:
      - `={{ $json.email }}`
    - Map additional fields:
      - `city` → `={{ $json.city || '' }}`
      - `country` → `={{ $json.country || $json.countryCode || '' }}`
      - `firstName` → `={{ $json.first_name || $json.firstName || '' }}`
      - `websiteUrl` → `={{ $json.website || $json.domain || $json.url || '' }}`
      - `companyName` → `={{ $json.store_name || $json.company || $json.businessName || '' }}`
      - `message` → multiline expression including platform, email status, catch-all flag, MX flag, and social links

54. **Connect `Filter Valid Emails Only` to `Create or Update HubSpot Contact`**.

55. **Optionally add sticky notes** matching the original structure:
    - Workflow trigger and config
    - Initiate scrape job
    - Scrape job polling loop
    - Download and parse leads
    - Filter and deduplicate leads
    - Initiate email validation
    - Validation job polling loop
    - Download and merge validation results
    - Push contacts to HubSpot

56. **Test the workflow end-to-end** with a small lead count first, such as 10–20 leads.
    - Verify ScraperCity returns `runId`
    - Confirm downloaded scrape data includes an `email` field
    - Confirm validation data includes status fields
    - Validate HubSpot property mappings

57. **Recommended hardening before production use:**
    - Add explicit handling for `FAILED`, `CANCELLED`, or timeout states in both polling loops
    - Add max retry counters to avoid infinite polling
    - Normalize emails to lowercase before deduplication
    - Add guards for empty email arrays before starting validation
    - Confirm whether ScraperCity download responses should be parsed as text or JSON in your n8n version

**Sub-workflow setup:**  
This workflow does **not** use any sub-workflow or Execute Workflow node. No child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Scrape Shopify store leads and push verified contacts into HubSpot CRM | Workflow title |
| Create a ScraperCity account and obtain your API key for the scrape and validation endpoints | ScraperCity setup |
| Add your ScraperCity API key as an n8n credential and reference it in the HTTP Request nodes | Credential setup |
| Connect your HubSpot account in n8n via the HubSpot credential (OAuth2 or API key) | HubSpot setup |
| Update the `Configure Search Parameters` node with your desired countryCode, totalLeads, includeEmails, and includePhones values | Customization |
| Review the polling wait times in `Wait Before First Poll` and `Wait 60 Seconds Before Retry` to match your expected scrape duration | Runtime tuning |
| Map the correct HubSpot contact fields in the `Create or Update HubSpot Contact` node to match your lead data structure | HubSpot field mapping |
| Adjust the countryCode and totalLeads parameters to target different markets or volumes | Search customization |
| The polling intervals (60s for scraping, 30s for validation) can be tuned based on typical job completion times | Performance tuning |
| The email filter and deduplication steps can be extended to also filter by phone availability or store category | Pipeline extension |
| Additional HubSpot properties (e.g., lead source, country) can be mapped in the final upsert node | CRM enrichment |