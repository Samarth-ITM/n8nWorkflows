Scrape Apollo.io leads with ScraperCity and sync enriched contacts to Airtable

https://n8nworkflows.xyz/workflows/scrape-apollo-io-leads-with-scrapercity-and-sync-enriched-contacts-to-airtable-14406


# Scrape Apollo.io leads with ScraperCity and sync enriched contacts to Airtable

## 1. Workflow Overview

This workflow manually launches an Apollo.io lead scraping job through ScraperCity, waits for the remote scrape to finish, downloads the resulting leads, normalizes and cleans the contact data, then inserts valid enriched contacts into Airtable.

Typical use cases:
- Build targeted outbound lead lists from Apollo.io filters
- Sync scraped lead data into Airtable for sales or enrichment workflows
- Automate periodic test runs of lead collection with manual supervision

### 1.1 Input Reception and Search Configuration
The workflow begins with a manual trigger and a configuration node where lead-search parameters and Airtable destination details are defined. This block supplies all downstream values.

### 1.2 Scrape Job Creation
The configured search criteria are sent to ScraperCity’s Apollo scraping endpoint. The returned run ID is stored together with Airtable metadata for later polling and saving.

### 1.3 Scrape Status Polling
The workflow waits 60 seconds, then checks whether the ScraperCity scrape job has completed. A conditional branch decides whether to continue to download or enter a retry loop.

### 1.4 Retry Loop for Incomplete Jobs
If the scrape is still pending, the workflow waits another 60 seconds, restores the run ID and Airtable settings into the current item, and polls the status endpoint again.

### 1.5 Result Download and Data Normalization
Once the scrape succeeds, the workflow downloads the lead export and transforms raw contact objects into a flat structure suitable for Airtable.

### 1.6 Data Cleaning and Airtable Sync
Normalized leads are deduplicated by work email, filtered to keep only contacts with a valid email-like value, and inserted into the configured Airtable base and table.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Search Configuration

**Overview:**  
This block starts the workflow manually and defines all search and storage parameters required later. It acts as the single source of truth for lead filters and Airtable destination values.

**Nodes Involved:**  
- When clicking 'Execute workflow'
- Configure Search Parameters

### Node Details

#### When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for interactive execution from the editor.
- **Configuration choices:** No custom parameters; standard manual trigger behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: Configure Search Parameters
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - No runtime issue by itself  
  - Only executes when manually started in the n8n editor or equivalent manual execution context
- **Sub-workflow reference:** None.

#### Configure Search Parameters
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a structured payload containing Apollo search criteria and Airtable destination settings.
- **Configuration choices:** Assigns these static fields:
  - `jobTitles`: `"CEO,CTO,VP of Sales"`
  - `industry`: `"Technology"`
  - `companySize`: `"11-50"`
  - `leadCount`: `100`
  - `airtableBaseId`: `"appYOUR_BASE_ID"`
  - `airtableTableName`: `"Leads"`
- **Key expressions or variables used:** None; values are hardcoded defaults.
- **Input and output connections:**  
  - Input: When clicking 'Execute workflow'  
  - Output: Start Apollo Lead Scrape
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**  
  - Invalid Airtable base ID or table name will only fail downstream at Airtable insertion time  
  - Empty or poorly formatted `jobTitles` can lead to weak or empty scrape results  
  - Invalid `leadCount` values may be rejected by ScraperCity depending on API rules
- **Sub-workflow reference:** None.

---

## 2.2 Scrape Job Creation

**Overview:**  
This block submits the configured Apollo filter search to ScraperCity and stores the returned run ID. It also preserves Airtable metadata so the workflow can continue after waits and retries.

**Nodes Involved:**  
- Start Apollo Lead Scrape
- Store Run ID

### Node Details

#### Start Apollo Lead Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an authenticated POST request to ScraperCity to initiate an Apollo lead scrape job.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/apollo-filters`
  - Authentication: Generic credential type using HTTP Header Auth
  - Body format: JSON
- **Key expressions or variables used:**  
  The JSON body is dynamically built:
  - `personTitles`: derived from `jobTitles.split(',').map(t => t.trim())`
  - `companyIndustry`: `{{ $json.industry }}`
  - `companySize`: `{{ $json.companySize }}`
  - `count`: `{{ $json.leadCount }}`
  - `fileName`: `"Apollo Export {{ $now.toISO() }}"`
- **Input and output connections:**  
  - Input: Configure Search Parameters  
  - Output: Store Run ID
- **Version-specific requirements:** Type version 4.2.
- **Edge cases or potential failure types:**  
  - Missing or invalid ScraperCity API key header  
  - ScraperCity API schema changes  
  - `jobTitles` splitting could produce empty entries if the input string is malformed  
  - Rate limiting, request timeout, or non-200 responses  
  - Body interpolation errors if upstream fields are missing
- **Sub-workflow reference:** None.

#### Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores the scrape `runId` from ScraperCity and reattaches Airtable configuration from the earlier Set node.
- **Configuration choices:** Creates:
  - `runId` from current HTTP response
  - `airtableBaseId` from Configure Search Parameters
  - `airtableTableName` from Configure Search Parameters
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
  - `={{ $('Configure Search Parameters').item.json.airtableBaseId }}`
  - `={{ $('Configure Search Parameters').item.json.airtableTableName }}`
- **Input and output connections:**  
  - Input: Start Apollo Lead Scrape  
  - Output: Wait 60 Seconds Before Poll
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**  
  - If ScraperCity does not return `runId`, all later status/download calls fail  
  - Cross-node expression dependency on `Configure Search Parameters` assumes that node executed successfully in the same run
- **Sub-workflow reference:** None.

---

## 2.3 Scrape Status Polling

**Overview:**  
This block waits before the first status check, calls the status endpoint, and branches depending on whether the remote job has completed. It is the central control point of the workflow.

**Nodes Involved:**  
- Wait 60 Seconds Before Poll
- Check Scrape Status
- Is Scrape Complete?

### Node Details

#### Wait 60 Seconds Before Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before the first status poll to give the scrape time to process.
- **Configuration choices:** Wait duration set to 60 seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: Store Run ID  
  - Output: Check Scrape Status
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or potential failure types:**  
  - n8n resume/webhook handling must function correctly for wait nodes  
  - Very large scrape jobs may require longer than 60 seconds, causing extra polling loops
- **Sub-workflow reference:** None.

#### Check Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an authenticated GET request to ScraperCity to retrieve current scrape status.
- **Configuration choices:**  
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
  - Authentication: HTTP Header Auth
- **Key expressions or variables used:**  
  - URL uses `{{ $json.runId }}`
- **Input and output connections:**  
  - Input: Wait 60 Seconds Before Poll, Preserve Run ID on Retry  
  - Output: Is Scrape Complete?
- **Version-specific requirements:** Type version 4.2.
- **Edge cases or potential failure types:**  
  - Missing `runId` from the incoming item  
  - Invalid or expired ScraperCity credentials  
  - Remote status endpoint downtime or non-JSON responses  
  - Possible unexpected statuses beyond `SUCCEEDED`
- **Sub-workflow reference:** None.

#### Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the current scrape status equals `SUCCEEDED`.
- **Configuration choices:**  
  - Strict string comparison
  - Condition: `$json.status == "SUCCEEDED"`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: Check Scrape Status  
  - True output: Download Lead Results  
  - False output: Wait 60 Seconds Before Retry
- **Version-specific requirements:** Type version 2.2 with conditions version 2.
- **Edge cases or potential failure types:**  
  - Any status other than `SUCCEEDED` is treated as incomplete, including permanent failure states such as `FAILED`, `CANCELLED`, or malformed responses  
  - This means failed jobs may loop forever unless additional conditions are added
- **Sub-workflow reference:** None.

---

## 2.4 Retry Loop for Incomplete Jobs

**Overview:**  
This block handles pending jobs by waiting again, then rebuilding the minimal context needed to continue polling. It creates the loop that repeats until status becomes `SUCCEEDED`.

**Nodes Involved:**  
- Wait 60 Seconds Before Retry
- Preserve Run ID on Retry

### Node Details

#### Wait 60 Seconds Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Introduces a pause between subsequent polling attempts.
- **Configuration choices:** Wait duration set to 60 seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: Is Scrape Complete? (false branch)  
  - Output: Preserve Run ID on Retry
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or potential failure types:**  
  - Same resume considerations as the first wait node  
  - If the external job never succeeds, the workflow continues polling indefinitely
- **Sub-workflow reference:** None.

#### Preserve Run ID on Retry
- **Type and technical role:** `n8n-nodes-base.set`  
  Reconstructs the state required for the next status-check request by preserving the current `runId` and re-reading Airtable settings.
- **Configuration choices:** Assigns:
  - `runId` from current item
  - `airtableBaseId` from Configure Search Parameters
  - `airtableTableName` from Configure Search Parameters
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
  - `={{ $('Configure Search Parameters').item.json.airtableBaseId }}`
  - `={{ $('Configure Search Parameters').item.json.airtableTableName }}`
- **Input and output connections:**  
  - Input: Wait 60 Seconds Before Retry  
  - Output: Check Scrape Status
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**  
  - If the false branch item no longer contains `runId`, polling breaks  
  - Cross-node lookup to Configure Search Parameters must remain valid in the same execution
- **Sub-workflow reference:** None.

---

## 2.5 Result Download and Data Normalization

**Overview:**  
After the job succeeds, this block fetches the completed export from ScraperCity and converts varied raw contact formats into a stable flat schema suitable for downstream storage.

**Nodes Involved:**  
- Download Lead Results
- Parse and Normalize Leads

### Node Details

#### Download Lead Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads lead results for the completed scrape run.
- **Configuration choices:**  
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
  - Authentication: HTTP Header Auth
- **Key expressions or variables used:**  
  - URL uses `{{ $json.runId }}`
- **Input and output connections:**  
  - Input: Is Scrape Complete? (true branch)  
  - Output: Parse and Normalize Leads
- **Version-specific requirements:** Type version 4.2.
- **Edge cases or potential failure types:**  
  - Download may fail if the run is marked succeeded before file availability is fully ready  
  - Response shape may vary: array root, `contacts`, or `data` wrapper  
  - Authentication or permission errors
- **Sub-workflow reference:** None.

#### Parse and Normalize Leads
- **Type and technical role:** `n8n-nodes-base.code`  
  Runs JavaScript to normalize each raw contact into a flat lead object.
- **Configuration choices:**  
  The code:
  - Detects possible payload shapes:
    - root array
    - `json.contacts`
    - `json.data`
  - Maps each contact into:
    - `firstName`
    - `lastName`
    - `fullName`
    - `jobTitle`
    - `company`
    - `workEmail`
    - `personalEmail`
    - `phone`
    - `linkedinUrl`
    - `city`
    - `country`
    - `companyWebsite`
- **Key expressions or variables used:**  
  Code uses `$input.first().json` and conditional property fallback patterns such as:
  - `contact.first_name || contact.firstName || ''`
  - `contact.email || contact.work_email || ''`
  - `contact.account ? (contact.account.website_url || '') : (contact.website_url || '')`
- **Input and output connections:**  
  - Input: Download Lead Results  
  - Output: Remove Duplicate Contacts
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - If the response is empty or in an unexpected format, the node may return zero items  
  - If `$input.first()` is undefined due to empty upstream output, the script can fail  
  - Some contacts may miss name/email/company fields; code defaults those to empty strings
- **Sub-workflow reference:** None.

---

## 2.6 Data Cleaning and Airtable Sync

**Overview:**  
This final block removes duplicate contacts, filters to retain only leads with an email address, and inserts records into Airtable using mapped column names.

**Nodes Involved:**  
- Remove Duplicate Contacts
- Filter Contacts with Email
- Save Leads to Airtable

### Node Details

#### Remove Duplicate Contacts
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Deduplicates normalized leads based on selected field values.
- **Configuration choices:**  
  - Compare mode: selected fields
  - Field used: `workEmail`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: Parse and Normalize Leads  
  - Output: Filter Contacts with Email
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - Empty `workEmail` values may collapse multiple records together before filtering, depending on node behavior  
  - Deduplication only considers `workEmail`, so duplicate people with different emails will remain
- **Sub-workflow reference:** None.

#### Filter Contacts with Email
- **Type and technical role:** `n8n-nodes-base.filter`  
  Keeps only contacts whose `workEmail` is non-empty and contains `@`.
- **Configuration choices:**  
  - Condition 1: `workEmail` is not empty
  - Condition 2: `workEmail` contains `@`
  - Case sensitivity off
  - Loose type validation
- **Key expressions or variables used:**  
  - `={{ $json.workEmail }}`
- **Input and output connections:**  
  - Input: Remove Duplicate Contacts  
  - Output: Save Leads to Airtable
- **Version-specific requirements:** Type version 2.2 with conditions version 2.
- **Edge cases or potential failure types:**  
  - This is only a basic email check; malformed addresses like `x@` still pass  
  - Valid contacts with only personal email are discarded because filtering is based only on `workEmail`
- **Sub-workflow reference:** None.

#### Save Leads to Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Creates new Airtable records from the cleaned lead items.
- **Configuration choices:**  
  - Operation: `create`
  - Base ID taken from `Store Run ID`
  - Table name taken from `Store Run ID`
  - Column mapping defined explicitly:
    - First Name ← `firstName`
    - Last Name ← `lastName`
    - Full Name ← `fullName`
    - Job Title ← `jobTitle`
    - Company ← `company`
    - Work Email ← `workEmail`
    - Personal Email ← `personalEmail`
    - Phone ← `phone`
    - LinkedIn URL ← `linkedinUrl`
    - City ← `city`
    - Country ← `country`
    - Company Website ← `companyWebsite`
- **Key expressions or variables used:**  
  - Base: `={{ $('Store Run ID').item.json.airtableBaseId }}`
  - Table: `={{ $('Store Run ID').item.json.airtableTableName }}`
  - Field mappings use current item values such as `={{ $json.fullName }}`
- **Input and output connections:**  
  - Input: Filter Contacts with Email  
  - Output: none
- **Version-specific requirements:** Type version 2.1.
- **Edge cases or potential failure types:**  
  - Invalid Airtable PAT, insufficient base/table permissions, or wrong base ID  
  - Table fields must exist with compatible names/types  
  - If Airtable contains unique constraints or automations, inserts may fail or behave unexpectedly  
  - Cross-node base/table expression references `Store Run ID`; if later design changes branch structure, this dependency should be reviewed
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow start |  | Configure Search Parameters | ## Workflow trigger and setup\n\nManually triggers the workflow and defines the lead search criteria (job titles, industry, company size, lead count) before any API calls are made. |
| Configure Search Parameters | Set | Defines Apollo search filters and Airtable destination | When clicking 'Execute workflow' | Start Apollo Lead Scrape | ## Workflow trigger and setup\n\nManually triggers the workflow and defines the lead search criteria (job titles, industry, company size, lead count) before any API calls are made. |
| Start Apollo Lead Scrape | HTTP Request | Starts Apollo scrape job via ScraperCity API | Configure Search Parameters | Store Run ID | ## Initiate Apollo scrape job\n\nSends a POST request to ScraperCity to start the Apollo.io lead scrape, then stores the returned run ID alongside Airtable configuration for downstream use. |
| Store Run ID | Set | Preserves run ID and Airtable config | Start Apollo Lead Scrape | Wait 60 Seconds Before Poll | ## Initiate Apollo scrape job\n\nSends a POST request to ScraperCity to start the Apollo.io lead scrape, then stores the returned run ID alongside Airtable configuration for downstream use. |
| Wait 60 Seconds Before Poll | Wait | Delays before first status check | Store Run ID | Check Scrape Status | ## Poll scrape completion status\n\nWaits 60 seconds, then polls ScraperCity to check whether the scrape job is finished. Branches on the result — proceeding to download on success or entering the retry loop on pending. |
| Check Scrape Status | HTTP Request | Retrieves scrape job status | Wait 60 Seconds Before Poll; Preserve Run ID on Retry | Is Scrape Complete? | ## Poll scrape completion status\n\nWaits 60 seconds, then polls ScraperCity to check whether the scrape job is finished. Branches on the result — proceeding to download on success or entering the retry loop on pending. |
| Is Scrape Complete? | If | Branches on SUCCEEDED status | Check Scrape Status | Download Lead Results; Wait 60 Seconds Before Retry | ## Poll scrape completion status\n\nWaits 60 seconds, then polls ScraperCity to check whether the scrape job is finished. Branches on the result — proceeding to download on success or entering the retry loop on pending. |
| Wait 60 Seconds Before Retry | Wait | Delays between retry polls | Is Scrape Complete? | Preserve Run ID on Retry | ## Retry loop for pending scrapes\n\nIf the scrape is not yet complete, waits another 60 seconds, re-attaches the run ID and Airtable metadata, then loops back to re-check the status. |
| Preserve Run ID on Retry | Set | Rebuilds polling context for next status call | Wait 60 Seconds Before Retry | Check Scrape Status | ## Retry loop for pending scrapes\n\nIf the scrape is not yet complete, waits another 60 seconds, re-attaches the run ID and Airtable metadata, then loops back to re-check the status. |
| Download Lead Results | HTTP Request | Downloads completed scrape results | Is Scrape Complete? | Parse and Normalize Leads | ## Download and normalize leads\n\nDownloads the completed lead results from ScraperCity and runs a code node to parse and normalize the raw contact objects into a consistent, usable structure. |
| Parse and Normalize Leads | Code | Flattens and standardizes raw lead objects | Download Lead Results | Remove Duplicate Contacts | ## Download and normalize leads\n\nDownloads the completed lead results from ScraperCity and runs a code node to parse and normalize the raw contact objects into a consistent, usable structure. |
| Remove Duplicate Contacts | Remove Duplicates | Deduplicates leads by work email | Parse and Normalize Leads | Filter Contacts with Email | ## Clean and save to Airtable\n\nRemoves duplicate contacts, filters out any records missing an email address, and writes the final clean lead list to the configured Airtable base and table. |
| Filter Contacts with Email | Filter | Keeps only email-bearing leads | Remove Duplicate Contacts | Save Leads to Airtable | ## Clean and save to Airtable\n\nRemoves duplicate contacts, filters out any records missing an email address, and writes the final clean lead list to the configured Airtable base and table. |
| Save Leads to Airtable | Airtable | Creates Airtable records for final leads | Filter Contacts with Email |  | ## Clean and save to Airtable\n\nRemoves duplicate contacts, filters out any records missing an email address, and writes the final clean lead list to the configured Airtable base and table. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Scrape Apollo.io leads and sync enriched contacts into Airtable via ScraperCity`.

2. **Add a Manual Trigger node**.
   - Node type: `Manual Trigger`
   - Keep default settings
   - This is the workflow entry point

3. **Add a Set node** after the trigger and name it `Configure Search Parameters`.
   - Node type: `Set`
   - Add these fields:
     - `jobTitles` → String → `CEO,CTO,VP of Sales`
     - `industry` → String → `Technology`
     - `companySize` → String → `11-50`
     - `leadCount` → Number → `100`
     - `airtableBaseId` → String → `appYOUR_BASE_ID`
     - `airtableTableName` → String → `Leads`
   - Connect: `Manual Trigger -> Configure Search Parameters`

4. **Create ScraperCity credentials** in n8n.
   - Credential type: `HTTP Header Auth`
   - Add the header required by ScraperCity using your API key
   - The exact header name depends on ScraperCity’s API documentation
   - Save it with a recognizable name such as `ScraperCity API Key`

5. **Add an HTTP Request node** named `Start Apollo Lead Scrape`.
   - Node type: `HTTP Request`
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/apollo-filters`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Header Auth`
   - Select your `ScraperCity API Key` credential
   - Enable body sending
   - Body type: JSON
   - Use this logic in the JSON body:
     - `personTitles`: split the `jobTitles` string by commas and trim spaces
     - `companyIndustry`: use the configured industry
     - `companySize`: use the configured company size
     - `count`: use the configured lead count
     - `fileName`: include a timestamp
   - Equivalent configuration:
     - `personTitles` from `JSON.stringify($json.jobTitles.split(',').map(t => t.trim()))`
     - `companyIndustry` from `$json.industry`
     - `companySize` from `$json.companySize`
     - `count` from `$json.leadCount`
     - `fileName` from `$now.toISO()`
   - Connect: `Configure Search Parameters -> Start Apollo Lead Scrape`

6. **Add a Set node** named `Store Run ID`.
   - Add fields:
     - `runId` → expression from current response: `{{$json.runId}}`
     - `airtableBaseId` → expression from Configure Search Parameters
     - `airtableTableName` → expression from Configure Search Parameters
   - Use expressions equivalent to:
     - `{{$('Configure Search Parameters').item.json.airtableBaseId}}`
     - `{{$('Configure Search Parameters').item.json.airtableTableName}}`
   - Connect: `Start Apollo Lead Scrape -> Store Run ID`

7. **Add a Wait node** named `Wait 60 Seconds Before Poll`.
   - Wait unit: `Seconds`
   - Amount: `60`
   - Connect: `Store Run ID -> Wait 60 Seconds Before Poll`

8. **Add an HTTP Request node** named `Check Scrape Status`.
   - Method: `GET`
   - URL expression: `https://app.scrapercity.com/api/v1/scrape/status/{{$json.runId}}`
   - Authentication: same `ScraperCity API Key`
   - Connect: `Wait 60 Seconds Before Poll -> Check Scrape Status`

9. **Add an If node** named `Is Scrape Complete?`.
   - Condition type: string
   - Left value: `{{$json.status}}`
   - Operation: `equals`
   - Right value: `SUCCEEDED`
   - Use strict comparison if available
   - Connect: `Check Scrape Status -> Is Scrape Complete?`

10. **Add a Wait node** named `Wait 60 Seconds Before Retry`.
    - Wait unit: `Seconds`
    - Amount: `60`
    - Connect the **false** output of `Is Scrape Complete?` to this node

11. **Add a Set node** named `Preserve Run ID on Retry`.
    - Add fields:
      - `runId` → `{{$json.runId}}`
      - `airtableBaseId` → from `Configure Search Parameters`
      - `airtableTableName` → from `Configure Search Parameters`
    - Connect: `Wait 60 Seconds Before Retry -> Preserve Run ID on Retry`

12. **Loop retry polling back into status checking**.
    - Connect: `Preserve Run ID on Retry -> Check Scrape Status`

13. **Add an HTTP Request node** named `Download Lead Results`.
    - Connect the **true** output of `Is Scrape Complete?` to this node
    - Method: `GET`
    - URL expression: `https://app.scrapercity.com/api/downloads/{{$json.runId}}`
    - Authentication: same `ScraperCity API Key`

14. **Add a Code node** named `Parse and Normalize Leads`.
    - Connect: `Download Lead Results -> Parse and Normalize Leads`
    - Paste JavaScript that:
      - reads an array from either:
        - the root response,
        - `contacts`,
        - or `data`
      - outputs one item per contact
      - maps fields into:
        - `firstName`
        - `lastName`
        - `fullName`
        - `jobTitle`
        - `company`
        - `workEmail`
        - `personalEmail`
        - `phone`
        - `linkedinUrl`
        - `city`
        - `country`
        - `companyWebsite`
    - Recreate the logic so it falls back across common alternative field names such as:
      - `first_name` or `firstName`
      - `last_name` or `lastName`
      - `email` or `work_email`
      - nested `account.website_url` or `website_url`

15. **Add a Remove Duplicates node** named `Remove Duplicate Contacts`.
    - Connect: `Parse and Normalize Leads -> Remove Duplicate Contacts`
    - Compare mode: `Selected Fields`
    - Field to compare: `workEmail`

16. **Add a Filter node** named `Filter Contacts with Email`.
    - Connect: `Remove Duplicate Contacts -> Filter Contacts with Email`
    - Create two conditions combined with AND:
      - `workEmail` is not empty
      - `workEmail` contains `@`
    - Case-sensitive: off
    - Loose validation is acceptable

17. **Create Airtable credentials** in n8n.
    - Credential type: `Airtable Personal Access Token`
    - Use a PAT with permissions to create records in the target base/table
    - Save with a clear name such as `Airtable Personal Access Token`

18. **Prepare the Airtable base and table**.
    - Create the base referenced by `airtableBaseId`
    - Create the table referenced by `airtableTableName`
    - Ensure these fields exist:
      - `First Name`
      - `Last Name`
      - `Full Name`
      - `Job Title`
      - `Company`
      - `Work Email`
      - `Personal Email`
      - `Phone`
      - `LinkedIn URL`
      - `City`
      - `Country`
      - `Company Website`

19. **Add an Airtable node** named `Save Leads to Airtable`.
    - Connect: `Filter Contacts with Email -> Save Leads to Airtable`
    - Operation: `Create`
    - Base: expression from `Store Run ID`:
      - `{{$('Store Run ID').item.json.airtableBaseId}}`
    - Table: expression from `Store Run ID`:
      - `{{$('Store Run ID').item.json.airtableTableName}}`
    - Use manual field mapping with these columns:
      - `First Name` ← `{{$json.firstName}}`
      - `Last Name` ← `{{$json.lastName}}`
      - `Full Name` ← `{{$json.fullName}}`
      - `Job Title` ← `{{$json.jobTitle}}`
      - `Company` ← `{{$json.company}}`
      - `Work Email` ← `{{$json.workEmail}}`
      - `Personal Email` ← `{{$json.personalEmail}}`
      - `Phone` ← `{{$json.phone}}`
      - `LinkedIn URL` ← `{{$json.linkedinUrl}}`
      - `City` ← `{{$json.city}}`
      - `Country` ← `{{$json.country}}`
      - `Company Website` ← `{{$json.companyWebsite}}`
    - Select the Airtable PAT credential

20. **Validate connection flow**.
    - Main path:
      - Manual Trigger
      - Configure Search Parameters
      - Start Apollo Lead Scrape
      - Store Run ID
      - Wait 60 Seconds Before Poll
      - Check Scrape Status
      - Is Scrape Complete?
    - Success branch:
      - Download Lead Results
      - Parse and Normalize Leads
      - Remove Duplicate Contacts
      - Filter Contacts with Email
      - Save Leads to Airtable
    - Retry branch:
      - Wait 60 Seconds Before Retry
      - Preserve Run ID on Retry
      - Check Scrape Status

21. **Test the workflow manually**.
    - Click `Execute workflow`
    - Confirm the scrape starts successfully
    - Verify the status loop eventually reaches `SUCCEEDED`
    - Check that records appear in Airtable

22. **Recommended hardening improvements** if rebuilding for production:
    - Add explicit handling for `FAILED`, `CANCELLED`, or timeout states
    - Add a maximum retry counter to avoid infinite loops
    - Improve email validation beyond just checking for `@`
    - Consider batching Airtable writes for large lead volumes
    - Add error notifications on API or Airtable failures

**Sub-workflow setup:**  
This workflow does not invoke any sub-workflows and is not itself represented as a called sub-workflow. It has a single manual entry point.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Scrape Apollo.io leads and sync enriched contacts into Airtable via ScraperCity | Workflow title |
| Create a ScraperCity account and obtain your API key for Apollo.io lead scraping | ScraperCity account setup |
| Set your desired search parameters (`jobTitles`, `industry`, `companySize`, `leadCount`) in the `Configure Search Parameters` node | Configuration guidance |
| Update the `Store Run ID` node with your Airtable Base ID and Table Name | Airtable destination guidance |
| Connect your Airtable account credentials in n8n and verify the `Save Leads to Airtable` node points to the correct base and table | Credential/setup guidance |
| Ensure your Airtable table has columns matching the fields output by the `Parse and Normalize Leads` node | Schema requirement |
| Test the workflow by clicking `Execute workflow` and monitoring the polling loop until leads appear in Airtable | Execution guidance |
| Adjust the two 60-second wait durations to suit expected scrape completion times — longer waits reduce API calls for large scrapes | Performance/customization note |
| Modify the filter in `Filter Contacts with Email` to add additional quality criteria, for example phone number present or a specific job title | Customization note |
| The deduplication key in `Remove Duplicate Contacts` can be changed from email to another unique field if needed | Data-quality customization |