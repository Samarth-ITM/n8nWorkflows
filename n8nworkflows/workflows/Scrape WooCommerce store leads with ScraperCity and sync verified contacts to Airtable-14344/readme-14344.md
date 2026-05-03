Scrape WooCommerce store leads with ScraperCity and sync verified contacts to Airtable

https://n8nworkflows.xyz/workflows/scrape-woocommerce-store-leads-with-scrapercity-and-sync-verified-contacts-to-airtable-14344


# Scrape WooCommerce store leads with ScraperCity and sync verified contacts to Airtable

## 1. Workflow Overview

This workflow manually launches a lead-generation process for WooCommerce stores using ScraperCity, waits for the scraping job to finish, downloads the resulting CSV, filters and deduplicates contacts, and then upserts the verified leads into Airtable.

Typical use cases:
- Building prospect lists of WooCommerce stores in a target country
- Collecting contactable leads for outreach
- Syncing scraped data into Airtable as a working lead database
- Running repeated lead collection jobs with simple parameter changes

### 1.1 Trigger and Scrape Setup
This block starts the workflow manually, defines scrape settings such as platform, country, and lead count, then submits a scrape request to ScraperCity and stores the returned `runId`.

### 1.2 Scrape Status Polling Loop
This block waits before the first status check, polls ScraperCity for job status, and loops with a 60-second retry delay until the scrape is marked as successful.

### 1.3 Results Download and Parsing
Once the scrape is complete, this block downloads the CSV output from ScraperCity and parses it into structured n8n items using a custom Code node.

### 1.4 Contact Filtering and Deduplication
This block keeps only rows containing at least an email or phone number, then removes duplicate email records before syncing.

### 1.5 Airtable Lead Sync
This final block upserts cleaned lead records into Airtable using the email address as the match key.

---

## 2. Block-by-Block Analysis

## 2.1 Trigger and Scrape Setup

**Overview:**  
This block initializes the workflow and defines all runtime parameters required for the scrape and Airtable destination. It then starts the scrape job with ScraperCity and extracts the returned run identifier needed for polling.

**Nodes Involved:**  
- When clicking 'Execute workflow'
- Configure Scrape Parameters
- Start WooCommerce Lead Scrape
- Store Run ID

### Node: When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for the workflow.
- **Configuration choices:**  
  No parameters configured; it simply starts execution when triggered in the editor.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Configure Scrape Parameters`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - No automated execution; this must be launched manually unless replaced with another trigger.
- **Sub-workflow reference:**  
  None.

### Node: Configure Scrape Parameters
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines the scrape configuration and Airtable destination settings as workflow data.
- **Configuration choices:**  
  Assigns these fields:
  - `platform` = `woocommerce`
  - `countryCode` = `US`
  - `totalLeads` = `500`
  - `collectEmails` = `true`
  - `collectPhones` = `true`
  - `airtableBaseId` = `appYOUR_BASE_ID`
  - `airtableTableName` = `WooCommerce Leads`
- **Key expressions or variables used:**  
  Static values are assigned here and reused downstream.
- **Input and output connections:**  
  - Input: `When clicking 'Execute workflow'`
  - Output: `Start WooCommerce Lead Scrape`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Invalid Airtable base ID or table name will break the Airtable sync later.
  - Setting unrealistic `totalLeads` may increase scrape time significantly.
  - Wrong `countryCode` or unsupported platform may produce no useful results.
- **Sub-workflow reference:**  
  None.

### Node: Start WooCommerce Lead Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the scrape creation request to the ScraperCity API.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/store-leads`
  - Authentication: Generic credential type using HTTP Header Auth
  - Body format: JSON
  - JSON body dynamically built from prior Set node values:
    - `platform`
    - `countryCode`
    - `emails`
    - `phones`
    - `totalLeads`
- **Key expressions or variables used:**  
  - `{{ $json.platform }}`
  - `{{ $json.countryCode }}`
  - `{{ $json.collectEmails }}`
  - `{{ $json.collectPhones }}`
  - `{{ $json.totalLeads }}`
- **Input and output connections:**  
  - Input: `Configure Scrape Parameters`
  - Output: `Store Run ID`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing or invalid ScraperCity API credential
  - 4xx/5xx API response
  - Malformed body if upstream values are changed incorrectly
  - Timeout or network issues
  - API schema changes on ScraperCity side
- **Sub-workflow reference:**  
  None.

### Node: Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores only the returned `runId` for simpler polling logic.
- **Configuration choices:**  
  Sets:
  - `runId` = `{{ $json.runId }}`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Start WooCommerce Lead Scrape`
  - Output: `Wait Before First Poll`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If ScraperCity does not return `runId`, downstream polling will fail.
  - If response structure changes, the expression will evaluate empty.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Scrape Status Polling Loop

**Overview:**  
This block manages asynchronous job completion. It delays the first poll, checks status, and either proceeds to download on success or loops back after another 60-second wait.

**Nodes Involved:**  
- Wait Before First Poll
- Poll Scrape Status
- Is Scrape Complete?
- Wait 60 Seconds Before Retry
- Preserve Run ID for Retry

### Node: Wait Before First Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Introduces an initial delay before the first status check.
- **Configuration choices:**  
  - Wait amount: `60`
  - No explicit unit is shown in the JSON snippet; in practice this should be verified in n8n UI after import.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Store Run ID`
  - Output: `Poll Scrape Status`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - If the wait unit is not what the builder expects, actual delay may differ.
  - Wait nodes depend on n8n’s execution persistence/resume behavior.
- **Sub-workflow reference:**  
  None.

### Node: Poll Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries ScraperCity for the status of the scraping job.
- **Configuration choices:**  
  - Method: `GET`
  - URL dynamically built from `runId`:
    `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
  - Authentication: Generic credential type using HTTP Header Auth
- **Key expressions or variables used:**  
  - `={{ "https://app.scrapercity.com/api/v1/scrape/status/" + $json.runId }}` conceptually; actual expression is inline in URL
  - `{{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Wait Before First Poll`, `Preserve Run ID for Retry`
  - Output: `Is Scrape Complete?`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing `runId`
  - Invalid ScraperCity credential
  - Job not found
  - Network/API timeout
  - Unexpected response structure or status values
- **Sub-workflow reference:**  
  None.

### Node: Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes execution depending on whether the scrape has completed successfully.
- **Configuration choices:**  
  Condition checks:
  - `status` equals `SUCCEEDED`
  - Strict validation enabled
  - Case-sensitive comparison
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Poll Scrape Status`
  - True output: `Download Scraped Results`
  - False output: `Wait 60 Seconds Before Retry`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Statuses such as `FAILED`, `CANCELLED`, `PENDING`, or any unexpected string all go to retry.
  - This means a permanently failed scrape may loop indefinitely unless the workflow is modified.
- **Sub-workflow reference:**  
  None.

### Node: Wait 60 Seconds Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays before retrying another status poll.
- **Configuration choices:**  
  - Wait amount: `60`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Is Scrape Complete?` false branch
  - Output: `Preserve Run ID for Retry`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Same wait-node persistence concerns as the first wait node.
  - Infinite polling risk if status never becomes `SUCCEEDED`.
- **Sub-workflow reference:**  
  None.

### Node: Preserve Run ID for Retry
- **Type and technical role:** `n8n-nodes-base.set`  
  Re-exposes `runId` after the wait step so the next polling request has the required identifier.
- **Configuration choices:**  
  Sets:
  - `runId` = `{{ $json.runId }}`
- **Key expressions or variables used:**  
  - `={{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Wait 60 Seconds Before Retry`
  - Output: `Poll Scrape Status`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If `runId` is lost or undefined after resuming the wait, subsequent polling will fail.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Results Download and Parsing

**Overview:**  
After the scrape succeeds, the workflow downloads the raw CSV text and converts it into individual structured items. A custom parser is used rather than n8n’s dedicated CSV node.

**Nodes Involved:**  
- Download Scraped Results
- Parse and Clean CSV Results

### Node: Download Scraped Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed scrape output from ScraperCity.
- **Configuration choices:**  
  - Method: `GET`
  - URL: `https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
  - Response format explicitly set to `text`
  - Authentication: Generic credential type using HTTP Header Auth
- **Key expressions or variables used:**  
  - `{{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Is Scrape Complete?` true branch
  - Output: `Parse and Clean CSV Results`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing `runId`
  - Empty download
  - Non-text response if API behavior changes
  - Unauthorized request
  - Download endpoint returns an error page instead of CSV
- **Sub-workflow reference:**  
  None.

### Node: Parse and Clean CSV Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses CSV text into records, normalizes headers, and performs an initial deduplication pass.
- **Configuration choices:**  
  The code:
  - Reads CSV text from `items[0].json.data` or `items[0].json.body`
  - Returns no items if content is empty
  - Splits into lines
  - Uses a custom `parseCSVLine()` function with basic quote handling
  - Lowercases headers and converts spaces to underscores
  - Builds one JSON item per row
  - Normalizes email to lowercase for dedupe logic
  - Deduplicates by email
  - Allows blank emails by assigning a synthetic unique key like `no-email-i`
- **Key expressions or variables used:**  
  Internal JavaScript variables:
  - `csvText`
  - `lines`
  - `parseCSVLine`
  - `headers`
  - `seen`
  - `output`
- **Input and output connections:**  
  - Input: `Download Scraped Results`
  - Output: `Filter Contacts with Email or Phone`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Custom CSV parsing is simplistic and may break on complex CSV cases:
    - embedded escaped quotes
    - multiline quoted fields
    - commas inside fields with unusual formatting
  - If the API returns JSON instead of text, parsing will fail silently or produce empty output.
  - If headers differ from expected names, downstream field mappings may become empty.
  - Initial deduplication happens here, and another dedupe happens later; behavior should be understood before modifying.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Contact Filtering and Deduplication

**Overview:**  
This block narrows the result set to contactable leads and removes duplicate email records before syncing to Airtable.

**Nodes Involved:**  
- Filter Contacts with Email or Phone
- Remove Duplicate Emails

### Node: Filter Contacts with Email or Phone
- **Type and technical role:** `n8n-nodes-base.filter`  
  Keeps only items that have at least one contact method.
- **Configuration choices:**  
  Logical `OR` across two conditions:
  - `email` is not empty
  - `phone` is not empty
  Case-sensitive comparison disabled; loose type validation enabled.
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `={{ $json.phone }}`
- **Input and output connections:**  
  - Input: `Parse and Clean CSV Results`
  - Output: `Remove Duplicate Emails`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Whitespace-only values may still count as non-empty depending on source cleanliness.
  - Invalid emails are not validated; only presence is checked.
  - Invalid or placeholder phone numbers are not validated.
- **Sub-workflow reference:**  
  None.

### Node: Remove Duplicate Emails
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Removes duplicate items based on the `email` field.
- **Configuration choices:**  
  - Compare field: `email`
- **Key expressions or variables used:**  
  None beyond configured field name.
- **Input and output connections:**  
  - Input: `Filter Contacts with Email or Phone`
  - Output: `Upsert Lead into Airtable`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - This duplicates logic already partially handled in the Code node.
  - Records without email may be treated unexpectedly depending on node behavior for empty comparison fields.
  - If multiple leads share a general inbox email, legitimate distinct businesses may be collapsed.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Airtable Lead Sync

**Overview:**  
This block writes each verified lead into Airtable using an upsert operation. Existing records are matched on email and updated; new ones are created.

**Nodes Involved:**  
- Upsert Lead into Airtable

### Node: Upsert Lead into Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Upserts lead records into a selected Airtable base and table.
- **Configuration choices:**  
  - Operation: `upsert`
  - Base/Application ID comes from:
    - `{{ $('Configure Scrape Parameters').item.json.airtableBaseId }}`
  - Table name comes from:
    - `{{ $('Configure Scrape Parameters').item.json.airtableTableName }}`
  - Match field:
    - `Email`
  - Explicit field mapping:
    - `Name` ← `{{ $json.name }}`
    - `Email` ← `{{ $json.email }}`
    - `Phone` ← `{{ $json.phone }}`
    - `Country` ← `{{ $json.country }}`
    - `Website` ← `{{ $json.website }}`
    - `Facebook` ← `{{ $json.facebook }}`
    - `Instagram` ← `{{ $json.instagram }}`
- **Key expressions or variables used:**  
  - `={{ $('Configure Scrape Parameters').item.json.airtableTableName }}`
  - `={{ $('Configure Scrape Parameters').item.json.airtableBaseId }}`
  - `={{ $json.name }}`
  - `={{ $json.email }}`
  - `={{ $json.phone }}`
  - `={{ $json.country }}`
  - `={{ $json.website }}`
  - `={{ $json.facebook }}`
  - `={{ $json.instagram }}`
- **Input and output connections:**  
  - Input: `Remove Duplicate Emails`
  - Output: none
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Invalid Airtable token, base ID, or table name
  - Airtable fields must exist exactly as configured: `Name`, `Email`, `Phone`, `Country`, `Website`, `Facebook`, `Instagram`
  - Upsert on `Email` can fail or behave poorly for blank emails
  - Airtable rate limits may affect large runs
  - If the base schema changes, updates may fail
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow start |  | Configure Scrape Parameters | ## Trigger and scrape setup\n\nManually triggers the workflow, defines scrape parameters, submits the scrape job to ScraperCity, and stores the returned run ID for polling. |
| Configure Scrape Parameters | Set | Defines scrape and Airtable parameters | When clicking 'Execute workflow' | Start WooCommerce Lead Scrape | ## Trigger and scrape setup\n\nManually triggers the workflow, defines scrape parameters, submits the scrape job to ScraperCity, and stores the returned run ID for polling. |
| Start WooCommerce Lead Scrape | HTTP Request | Creates a ScraperCity scrape job | Configure Scrape Parameters | Store Run ID | ## Trigger and scrape setup\n\nManually triggers the workflow, defines scrape parameters, submits the scrape job to ScraperCity, and stores the returned run ID for polling. |
| Store Run ID | Set | Stores returned ScraperCity run ID | Start WooCommerce Lead Scrape | Wait Before First Poll | ## Trigger and scrape setup\n\nManually triggers the workflow, defines scrape parameters, submits the scrape job to ScraperCity, and stores the returned run ID for polling. |
| Wait Before First Poll | Wait | Delays first status check | Store Run ID | Poll Scrape Status | ## Scrape status polling loop\n\nWaits before the first poll, checks the scrape job status, and routes to either download (if complete) or a retry loop that waits 60 seconds and re-polls. |
| Poll Scrape Status | HTTP Request | Checks ScraperCity job status | Wait Before First Poll; Preserve Run ID for Retry | Is Scrape Complete? | ## Scrape status polling loop\n\nWaits before the first poll, checks the scrape job status, and routes to either download (if complete) or a retry loop that waits 60 seconds and re-polls. |
| Is Scrape Complete? | If | Branches on scrape completion status | Poll Scrape Status | Download Scraped Results; Wait 60 Seconds Before Retry | ## Scrape status polling loop\n\nWaits before the first poll, checks the scrape job status, and routes to either download (if complete) or a retry loop that waits 60 seconds and re-polls. |
| Wait 60 Seconds Before Retry | Wait | Delays before retry polling | Is Scrape Complete? | Preserve Run ID for Retry | ## Scrape status polling loop\n\nWaits before the first poll, checks the scrape job status, and routes to either download (if complete) or a retry loop that waits 60 seconds and re-polls. |
| Preserve Run ID for Retry | Set | Re-stores run ID before another poll | Wait 60 Seconds Before Retry | Poll Scrape Status | ## Scrape status polling loop\n\nWaits before the first poll, checks the scrape job status, and routes to either download (if complete) or a retry loop that waits 60 seconds and re-polls. |
| Download Scraped Results | HTTP Request | Downloads completed CSV results | Is Scrape Complete? | Parse and Clean CSV Results | ## Results download and parsing\n\nDownloads the completed scrape as a CSV file from ScraperCity and parses and cleans it into structured data items ready for filtering. |
| Parse and Clean CSV Results | Code | Parses CSV text into structured lead records | Download Scraped Results | Filter Contacts with Email or Phone | ## Results download and parsing\n\nDownloads the completed scrape as a CSV file from ScraperCity and parses and cleans it into structured data items ready for filtering. |
| Filter Contacts with Email or Phone | Filter | Keeps leads with at least one contact field | Parse and Clean CSV Results | Remove Duplicate Emails | ## Contact filtering and deduplication\n\nRetains only contacts that have a valid email or phone number, then removes any duplicate email entries to ensure clean lead data. |
| Remove Duplicate Emails | Remove Duplicates | Deduplicates leads on email | Filter Contacts with Email or Phone | Upsert Lead into Airtable | ## Contact filtering and deduplication\n\nRetains only contacts that have a valid email or phone number, then removes any duplicate email entries to ensure clean lead data. |
| Upsert Lead into Airtable | Airtable | Creates or updates Airtable lead records | Remove Duplicate Emails |  | ## Airtable lead sync\n\nUpserts each verified, deduplicated lead record into the Airtable base, creating new entries or updating existing ones. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Scrape WooCommerce store leads and sync verified contacts into Airtable`.

2. **Add a Manual Trigger node**.
   - Node type: `Manual Trigger`
   - Name it: `When clicking 'Execute workflow'`

3. **Add a Set node** after the trigger.
   - Node type: `Set`
   - Name it: `Configure Scrape Parameters`
   - Add these fields:
     1. `platform` as String = `woocommerce`
     2. `countryCode` as String = `US`
     3. `totalLeads` as Number = `500`
     4. `collectEmails` as Boolean = `true`
     5. `collectPhones` as Boolean = `true`
     6. `airtableBaseId` as String = `appYOUR_BASE_ID`
     7. `airtableTableName` as String = `WooCommerce Leads`

4. **Create ScraperCity credentials** in n8n.
   - Credential type: `HTTP Header Auth`
   - Use the header format required by ScraperCity API.
   - The workflow JSON indicates all ScraperCity requests use the same HTTP Header Auth credential.
   - Assign a recognizable name such as `ScraperCity API Key`.

5. **Add an HTTP Request node** after the Set node.
   - Name: `Start WooCommerce Lead Scrape`
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/store-leads`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Header Auth`
   - Select your `ScraperCity API Key` credential
   - Enable sending body
   - Body content type: JSON
   - Set JSON body to:
     ```json
     {
       "platform": "{{ $json.platform }}",
       "countryCode": "{{ $json.countryCode }}",
       "emails": {{ $json.collectEmails }},
       "phones": {{ $json.collectPhones }},
       "totalLeads": {{ $json.totalLeads }}
     }
     ```
   - Confirm expression mode is enabled where needed.

6. **Add a Set node** after the scrape request.
   - Name: `Store Run ID`
   - Add one field:
     - `runId` as String = `{{ $json.runId }}`
   - This keeps downstream payload minimal and predictable.

7. **Add a Wait node** after `Store Run ID`.
   - Name: `Wait Before First Poll`
   - Set wait amount to `60`
   - Verify in your n8n UI whether the unit is seconds/minutes and set it explicitly to **60 seconds** if needed.

8. **Add an HTTP Request node** after the first Wait node.
   - Name: `Poll Scrape Status`
   - Method: `GET`
   - URL:
     `https://app.scrapercity.com/api/v1/scrape/status/{{ $json.runId }}`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Header Auth`
   - Reuse the `ScraperCity API Key` credential

9. **Add an If node** after the status request.
   - Name: `Is Scrape Complete?`
   - Create one condition:
     - Left value: `{{ $json.status }}`
     - Operator: `equals`
     - Right value: `SUCCEEDED`
   - Use strict validation and case-sensitive comparison if available.

10. **Create the retry branch** from the false output of the If node.
    - Add a Wait node.
    - Name: `Wait 60 Seconds Before Retry`
    - Set the delay to `60 seconds`

11. **Add a Set node** after the retry Wait node.
    - Name: `Preserve Run ID for Retry`
    - Add one field:
      - `runId` as String = `{{ $json.runId }}`
    - Connect this node back into `Poll Scrape Status`

12. **Create the success branch** from the true output of the If node.
    - Add an HTTP Request node.
    - Name: `Download Scraped Results`
    - Method: `GET`
    - URL:
      `https://app.scrapercity.com/api/downloads/{{ $json.runId }}`
    - Authentication: `Generic Credential Type`
    - Generic Auth Type: `HTTP Header Auth`
    - Reuse the `ScraperCity API Key` credential
    - In response settings, set response format to `Text`

13. **Add a Code node** after the download node.
    - Name: `Parse and Clean CSV Results`
    - Paste this JavaScript logic:
      ```javascript
      const csvText = items[0].json.data || items[0].json.body || '';

      if (!csvText || csvText.trim().length === 0) {
        return [];
      }

      const lines = csvText.trim().split('\n');
      if (lines.length < 2) return [];

      const parseCSVLine = (line) => {
        const result = [];
        let current = '';
        let inQuotes = false;
        for (let i = 0; i < line.length; i++) {
          const ch = line[i];
          if (ch === '"') {
            inQuotes = !inQuotes;
          } else if (ch === ',' && !inQuotes) {
            result.push(current.trim());
            current = '';
          } else {
            current += ch;
          }
        }
        result.push(current.trim());
        return result;
      };

      const headers = parseCSVLine(lines[0]).map(h => h.toLowerCase().replace(/\s+/g, '_'));

      const seen = new Set();
      const output = [];

      for (let i = 1; i < lines.length; i++) {
        if (!lines[i].trim()) continue;
        const values = parseCSVLine(lines[i]);
        const record = {};
        headers.forEach((h, idx) => {
          record[h] = values[idx] || '';
        });

        const email = (record.email || '').trim().toLowerCase();
        const dedupeKey = email || ('no-email-' + i);
        if (seen.has(dedupeKey)) continue;
        seen.add(dedupeKey);

        output.push({ json: record });
      }

      return output;
      ```

14. **Add a Filter node** after the Code node.
    - Name: `Filter Contacts with Email or Phone`
    - Set combinator to `OR`
    - Add condition 1:
      - `{{ $json.email }}`
      - Operator: `is not empty`
    - Add condition 2:
      - `{{ $json.phone }}`
      - Operator: `is not empty`

15. **Add a Remove Duplicates node** after the Filter node.
    - Name: `Remove Duplicate Emails`
    - Compare on field:
      - `email`

16. **Create Airtable credentials** in n8n.
    - Credential type: `Airtable Personal Access Token`
    - Use a token with access to the target base/table.
    - Ensure the Airtable base contains these fields:
      - `Name`
      - `Email`
      - `Phone`
      - `Country`
      - `Website`
      - `Facebook`
      - `Instagram`

17. **Add an Airtable node** after `Remove Duplicate Emails`.
    - Name: `Upsert Lead into Airtable`
    - Credential: your Airtable Personal Access Token
    - Operation: `Upsert`
    - Base/Application ID:
      `{{ $('Configure Scrape Parameters').item.json.airtableBaseId }}`
    - Table name:
      `{{ $('Configure Scrape Parameters').item.json.airtableTableName }}`
    - Fields to match on:
      - `Email`

18. **Configure field mapping in the Airtable node**.
    - Map:
      - `Name` = `{{ $json.name }}`
      - `Email` = `{{ $json.email }}`
      - `Phone` = `{{ $json.phone }}`
      - `Country` = `{{ $json.country }}`
      - `Website` = `{{ $json.website }}`
      - `Facebook` = `{{ $json.facebook }}`
      - `Instagram` = `{{ $json.instagram }}`

19. **Connect the nodes in this exact order**:
    - `When clicking 'Execute workflow'` → `Configure Scrape Parameters`
    - `Configure Scrape Parameters` → `Start WooCommerce Lead Scrape`
    - `Start WooCommerce Lead Scrape` → `Store Run ID`
    - `Store Run ID` → `Wait Before First Poll`
    - `Wait Before First Poll` → `Poll Scrape Status`
    - `Poll Scrape Status` → `Is Scrape Complete?`
    - `Is Scrape Complete?` true → `Download Scraped Results`
    - `Download Scraped Results` → `Parse and Clean CSV Results`
    - `Parse and Clean CSV Results` → `Filter Contacts with Email or Phone`
    - `Filter Contacts with Email or Phone` → `Remove Duplicate Emails`
    - `Remove Duplicate Emails` → `Upsert Lead into Airtable`
    - `Is Scrape Complete?` false → `Wait 60 Seconds Before Retry`
    - `Wait 60 Seconds Before Retry` → `Preserve Run ID for Retry`
    - `Preserve Run ID for Retry` → `Poll Scrape Status`

20. **Test the workflow manually**.
    - Start with a low `totalLeads` value such as `10` or `25`
    - Confirm ScraperCity returns a `runId`
    - Confirm the polling response eventually returns `status = SUCCEEDED`
    - Confirm the download endpoint returns CSV text
    - Confirm Airtable receives records

21. **Recommended hardening improvements before production use**:
    - Add a failure branch for statuses like `FAILED` or `CANCELLED`
    - Add a max retry counter to avoid infinite polling
    - Trim whitespace in email/phone before filtering
    - Validate email format before Airtable upsert
    - Consider using a dedicated CSV parser node if CSV complexity increases

**Sub-workflow setup:**  
This workflow does not include any sub-workflow nodes and does not invoke another workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Scrape WooCommerce store leads and sync verified contacts into Airtable | Workflow title |
| The workflow is triggered manually and configured with scrape parameters (platform, country, lead count, email collection). | Workflow description note |
| A scrape job is submitted to the ScraperCity API and the returned run ID is stored. | Workflow description note |
| The workflow polls the ScraperCity API at intervals, retrying every 60 seconds until the scrape job is marked complete. | Workflow description note |
| Once complete, the scraped CSV results are downloaded and parsed into structured records. | Workflow description note |
| Contacts are filtered to retain only those with an email or phone number, and duplicates are removed. | Workflow description note |
| Verified, deduplicated leads are upserted into an Airtable base for ongoing outreach. | Workflow description note |
| Create a ScraperCity account and obtain your API key | `app.scrapercity.com` |
| Add your ScraperCity API key as an n8n credential (HTTP Header Auth) and attach it to the HTTP Request nodes | ScraperCity setup context |
| Set up an Airtable base with fields matching the scraped lead data (e.g. email, phone, store URL) and obtain your Airtable API key | Airtable setup context |
| Configure the Airtable node with your base ID, table name, and API credential | Airtable setup context |
| In 'Configure Scrape Parameters', set your desired platform, countryCode, totalLeads, and collectEmails values | Parameter configuration context |
| Adjust the wait durations in 'Wait Before First Poll' and 'Wait 60 Seconds Before Retry' to match expected scrape times | Timing configuration context |
| Increase 'totalLeads' in the scrape parameters to collect larger datasets. | Customization context |
| Adjust the polling interval (Wait 60 Seconds Before Retry) if scrapes consistently take longer or shorter. | Customization context |
| Add a Slack or email notification node after the Airtable upsert to alert the team when new leads are synced. | Customization context |
| The filter node can be extended to require both email AND phone if stricter lead quality is needed. | Customization context |