Generate roofing contractor leads from Google Maps with ScrapeOps, Sheets and Slack

https://n8nworkflows.xyz/workflows/generate-roofing-contractor-leads-from-google-maps-with-scrapeops--sheets-and-slack-14118


# Generate roofing contractor leads from Google Maps with ScrapeOps, Sheets and Slack

# 1. Workflow Overview

This workflow generates roofing contractor leads for a user-specified city by scraping Google Maps through ScrapeOps, enriching each listing with deeper business details, removing duplicates against an existing Google Sheet, then saving only new leads and sending notifications through Gmail and Slack.

Typical use cases:
- Local lead generation for roofing services
- Sales prospecting by city
- Reusable business discovery workflow for similar trades such as HVAC, plumbing, or electricians

## 1.1 Input Reception & Search Configuration
The workflow starts from a form where the user submits a city. A Set node then standardizes the city input and builds the Google Maps search URL for “roofing contractor”.

## 1.2 Google Maps Search Scrape
The workflow uses ScrapeOps to fetch the rendered Google Maps search result page for roofing contractors in the requested city.

## 1.3 Listing Extraction & Deep Detail Scrape
A Code node parses the search-result HTML into candidate businesses, then a second ScrapeOps node fetches each business detail page from its Google Maps URL.

## 1.4 Business Normalization & Review Extraction
A large Code node matches detail-page HTML back to the original listing, then extracts normalized fields such as business name, phone, website, address, rating, total reviews, and up to three review texts.

## 1.5 Deduplication Against Google Sheets
Existing sheet rows are read, and another Code node compares prior businesses to the newly scraped results using normalized business names and phone numbers. Each result is tagged as `new` or `old`.

## 1.6 Save New Leads & Notify
Only leads marked `new` pass through an IF node, are appended to Google Sheets, and then trigger a Gmail alert followed by a Slack alert.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception & Search Configuration

**Overview:**  
This block collects the target city from a form and constructs the search parameters required to scrape Google Maps. It ensures the rest of the workflow always receives a consistent `city`, `keyword`, and `searchUrl`.

**Nodes Involved:**  
- Form: Enter City to Search  
- Set Google Maps Configuration

### Node Details

#### Form: Enter City to Search
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes a web form and starts the workflow when submitted.
- **Configuration choices:**
  - Form title: `Tell Which City`
  - One required field: `City Name`
  - Custom form path is defined
- **Key expressions or variables used:**
  - Produces `City Name` in the incoming JSON payload
- **Input and output connections:**
  - Input: none, this is the trigger
  - Output: Set Google Maps Configuration
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires n8n instance support for Form Trigger nodes
- **Edge cases or potential failure types:**
  - Empty submissions are prevented by the required field
  - A technically valid but meaningless city name can still be submitted and later cause “no businesses found”
  - Form path conflicts may occur if another workflow uses the same path
- **Sub-workflow reference:** none

#### Set Google Maps Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes input fields and generates the Google Maps search URL.
- **Configuration choices:**
  - `city` is derived from either `$json.city` or `$json['City Name']`
  - `keyword` is hardcoded to `roofing contractor`
  - `baseUrl` is set to `https://www.google.com/maps/search/`
  - `searchUrl` is dynamically assembled as:
    - `https://www.google.com/maps/search/roofing+contractor+in+<city>`
    - spaces in city are replaced with `+`
- **Key expressions or variables used:**
  - `={{ $json.city || $json['City Name'] || '' }}`
  - `={{ 'https://www.google.com/maps/search/roofing+contractor+in+' + String($json.city || $json['City Name'] || '').replace(/\s+/g, '+') }}`
- **Input and output connections:**
  - Input: Form: Enter City to Search
  - Output: ScrapeOps: Search Google Maps
- **Version-specific requirements:**
  - Uses `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - Blank city produces an incomplete search URL
  - Special characters in city names are only partially normalized; some international names may yield inconsistent Maps URLs
  - Keyword is fixed; changing the lead type requires manual update here
- **Sub-workflow reference:** none

---

## 2.2 Google Maps Search Scrape

**Overview:**  
This block fetches the Google Maps search page using ScrapeOps with JavaScript rendering enabled. It is the first scraping stage and returns HTML used for extracting listing-level business data.

**Nodes Involved:**  
- ScrapeOps: Search Google Maps

### Node Details

#### ScrapeOps: Search Google Maps
- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`  
  Sends the Google Maps search URL through ScrapeOps and returns rendered HTML.
- **Configuration choices:**
  - URL: `{{$json.searchUrl}}`
  - Return type: `htmlResponse`
  - JS rendering: enabled
  - Wait time: `12000` ms
  - Residential proxy: disabled
- **Key expressions or variables used:**
  - `={{ $json.searchUrl }}`
  - Wait expression: `=12000`
- **Input and output connections:**
  - Input: Set Google Maps Configuration
  - Output: Parse Business Listings
- **Version-specific requirements:**
  - Uses ScrapeOps community/custom node `typeVersion: 1`
  - Requires ScrapeOps credentials configured in n8n
- **Edge cases or potential failure types:**
  - Invalid or expired ScrapeOps API key
  - Google Maps response structure changing
  - Insufficient render wait time causing partial HTML
  - Rate limits, anti-bot responses, or blocked requests
  - Empty/partial HTML if Maps does not fully load
- **Sub-workflow reference:** none

---

## 2.3 Listing Extraction & Deep Detail Scrape

**Overview:**  
This block parses the Google Maps search-result HTML into individual business candidates, then fetches each business’s Google Maps detail page for deeper extraction. It is the transition from search-page scraping to record-level enrichment.

**Nodes Involved:**  
- Parse Business Listings  
- ScrapeOps: Fetch Business Details

### Node Details

#### Parse Business Listings
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses search result HTML to extract listing-level records such as business name, rating, address, phone, website, map URL, and city.
- **Configuration choices:**
  - Reads HTML from possible fields: `response`, raw JSON string, `body`, or `data`
  - Validates that city exists before parsing
  - Truncates HTML to 1,000,000 characters to reduce memory load
  - Uses several regex-based extraction strategies:
    - Google Maps place URLs
    - Card-based fallback extraction
    - Business name, rating, review count, address, phone, website
  - Returns an error-style item if city is invalid or no businesses are found
- **Key expressions or variables used:**
  - `$('Set Google Maps Configuration').item.json`
  - `config.city`
  - `config.keyword`
  - `item.json.response`
- **Input and output connections:**
  - Input: ScrapeOps: Search Google Maps
  - Output: ScrapeOps: Fetch Business Details
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires Code node JavaScript support
- **Edge cases or potential failure types:**
  - Regex parsing can break if Google Maps HTML changes
  - No `mapUrl` found for some cards
  - Search result HTML may be incomplete if ScrapeOps rendered too early
  - Output may include an error item instead of real businesses
  - There is a code-level issue: within the loop destructuring, `currentMapUrl` is declared as part of the loop item and later reassigned. In strict JavaScript this can cause assignment errors if that branch executes
- **Sub-workflow reference:** none

#### ScrapeOps: Fetch Business Details
- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`  
  Fetches each Google Maps business detail page using the `mapUrl` extracted from the previous node.
- **Configuration choices:**
  - URL: `{{$json.mapUrl}}`
  - Return type: `htmlResponse`
  - JS rendering: enabled
  - Wait time: `18000` ms
  - Residential proxy: disabled
- **Key expressions or variables used:**
  - `={{ $json.mapUrl }}`
  - Wait expression: `=18000`
- **Input and output connections:**
  - Input: Parse Business Listings
  - Output: Parse Full Business Info
- **Version-specific requirements:**
  - Uses ScrapeOps node `typeVersion: 1`
  - Requires ScrapeOps credentials
- **Edge cases or potential failure types:**
  - Blank or invalid `mapUrl`
  - Timeout or incomplete JS-rendered content
  - Anti-bot response from Google Maps
  - Higher latency because each listing is fetched separately
- **Sub-workflow reference:** none

---

## 2.4 Business Normalization & Review Extraction

**Overview:**  
This block combines deep-scraped detail HTML with the original listing data, then outputs fully normalized business records. It also extracts rating and review count from detail pages only, and attempts to capture up to three review texts.

**Nodes Involved:**  
- Parse Full Business Info

### Node Details

#### Parse Full Business Info
- **Type and technical role:** `n8n-nodes-base.code`  
  Multi-item processing node that correlates each detail-page HTML response with the corresponding search result item and produces enriched lead records.
- **Configuration choices:**
  - Reads all input items via `$input.all()`
  - Attempts to extract `mapUrl` from HTML
  - Cross-matches current detail HTML against:
    - `Parse Business Listings`
    - `ScrapeOps: Fetch Business Details`
    - item index fallback
  - If matching fails, emits a `MATCHING_ERROR` record
  - Explicitly avoids trusting search-page rating/review counts and re-extracts these from detail-page HTML only
  - Extracts:
    - businessName
    - phone
    - website
    - rating
    - totalReviews
    - address
    - city
    - category
    - checkedAt
    - review1, review2, review3
  - Limits HTML to 2,000,000 characters
  - Uses multiple regex and heuristic strategies for review extraction:
    - `data-expanded-text`
    - `data-original-text`
    - review blocks
    - known class selectors
    - JSON-LD
    - meta tags
    - broad fallback candidate scanning
- **Key expressions or variables used:**
  - `$input.all()`
  - `$(' Parse Business Listings')`
  - `$(' ScrapeOps: Fetch Business Details')`
  - `inputItem.index`
- **Input and output connections:**
  - Input: ScrapeOps: Fetch Business Details
  - Output: Read Previous Entries from Sheet
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Relies on Code node access to other node outputs using `$()`
- **Edge cases or potential failure types:**
  - Node-name references include leading spaces and therefore depend on exact node names
  - Match logic can fail when Google Maps changes URL formats or when multiple businesses have similar URLs
  - Detail-page HTML may omit phone, website, address, or review sections
  - Overly large HTML may still cause performance issues despite truncation
  - Review extraction is heuristic and can pull noisy or duplicated text
  - If the prior node emitted an error record without useful HTML, this node may still generate fallback/error outputs
- **Sub-workflow reference:** none

---

## 2.5 Deduplication Against Google Sheets

**Overview:**  
This block loads existing rows from Google Sheets and compares them against freshly scraped businesses. It labels each result as `new` or `old` based on normalized business names and phone numbers.

**Nodes Involved:**  
- Read Previous Entries from Sheet  
- Compare & Deduplicate Leads  
- Filter New Leads Only

### Node Details

#### Read Previous Entries from Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads prior lead records from the target spreadsheet to support deduplication.
- **Configuration choices:**
  - Uses the sheet with `gid=0`
  - Spreadsheet is referenced by full Google Sheets URL
  - Uses Google Sheets OAuth2 credentials
  - `continueOnFail: true`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - No dynamic mapping in this node; the document and sheet are statically selected
- **Input and output connections:**
  - Input: Parse Full Business Info
  - Output: Compare & Deduplicate Leads
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`
  - Requires Google Sheets OAuth2 credential with access to the spreadsheet
- **Edge cases or potential failure types:**
  - Spreadsheet not shared with the credential owner
  - Wrong sheet tab selected
  - Header mismatch with downstream comparison assumptions
  - Because `alwaysOutputData` is enabled, downstream logic can still run even on partial failures
- **Sub-workflow reference:** none

#### Compare & Deduplicate Leads
- **Type and technical role:** `n8n-nodes-base.code`  
  Reads both current normalized lead data and historical sheet rows, then marks records as `new` or `old`.
- **Configuration choices:**
  - Normalizes phones by keeping digits and optional leading `+`
  - Normalizes business names to lowercase, trimmed, single-space text
  - Pulls current items from `Parse Full Business Info`
  - Pulls historical items from `Read Previous Entries from Sheet`
  - Supports several alternate column naming conventions from Sheets
  - Deduplicates both:
    - against previous sheet entries
    - within the current execution
  - Outputs status plus all lead fields
- **Key expressions or variables used:**
  - `$('Parse Full Business Info')`
  - `$('Read Previous Entries from Sheet')`
  - status field set to `new` or `old`
- **Input and output connections:**
  - Input: Read Previous Entries from Sheet
  - Output: Filter New Leads Only
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If the Parse Full Business Info node output is unavailable, it returns a synthetic error item
  - If no businesses are found, it returns a placeholder item with status `old`
  - Same business with changed spelling or changed phone may bypass deduplication
  - Distinct branches are not merged formally; this node relies on direct cross-node reads
- **Sub-workflow reference:** none

#### Filter New Leads Only
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters the deduplicated stream so only rows with `status = new` continue to saving.
- **Configuration choices:**
  - String condition:
    - `value1 = {{$json.status}}`
    - `value2 = new`
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: Compare & Deduplicate Leads
  - Output:
    - True branch: Save New Leads to Sheet
    - False branch: unused
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Placeholder “no data” or “no businesses found” records should remain filtered out because status is `old`
  - If status is missing, the condition evaluates false
- **Sub-workflow reference:** none

---

## 2.6 Save New Leads & Notify

**Overview:**  
This block writes only new leads to Google Sheets and sends one Gmail message and one Slack message per execution. Notifications are execution-level summaries rather than per-row alerts because the Gmail and Slack nodes are configured to execute once.

**Nodes Involved:**  
- Save New Leads to Sheet  
- Send Gmail Alert  
- Send Slack Alert

### Node Details

#### Save New Leads to Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends newly discovered businesses to the destination Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Target sheet: `gid=0`
  - Explicit field mapping for:
    - city
    - phone
    - mapUrl
    - rating
    - status
    - address
    - review1
    - review2
    - review3
    - website
    - category
    - checkedAt
    - businessName
    - totalReviews
  - Mapping mode: manual definition
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - Each column maps from `$json.<field>`
- **Input and output connections:**
  - Input: Filter New Leads Only
  - Output: Send Gmail Alert
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - The cached result URL contains a typo (`hhttps`) but the actual document URL appears correct; UI metadata may be inconsistent without affecting runtime
  - Sheet headers must match the mapped fields or appended data may not align as expected
  - If Google Sheets auth fails, notifications may still run because `continueOnFail` is enabled upstream
- **Sub-workflow reference:** none

#### Send Gmail Alert
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email notification after new leads are saved.
- **Configuration choices:**
  - Recipient: `example@.com` placeholder
  - Email type: plain text
  - Attribution disabled
  - Subject dynamically includes the selected city:
    - `📍 New Businesses Found in <city> (Roofing Contractor)`
  - Message body includes the Google Sheet URL
  - `executeOnce: true`
- **Key expressions or variables used:**
  - `={{ '📍 New Businesses Found in ' + $('Set Google Maps Configuration').first().json.city + ' (Roofing Contractor)' }}`
  - `={{ 'New Roofing Contractor Found\n\n' + 'Sheet URL: ' + '<sheet-url>' }}`
- **Input and output connections:**
  - Input: Save New Leads to Sheet
  - Output: Send Slack Alert
- **Version-specific requirements:**
  - Uses `typeVersion: 2.1`
  - Requires Gmail OAuth2 credentials
- **Edge cases or potential failure types:**
  - Recipient is clearly a placeholder and must be replaced before production use
  - Gmail credential scope/authentication errors
  - Subject expression fails if Set Google Maps Configuration is unavailable
  - Because `executeOnce` is true, only one email is sent even if multiple rows are added
- **Sub-workflow reference:** none

#### Send Slack Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack notification containing the sheet link.
- **Configuration choices:**
  - Sends message as a direct message/user-targeted notification
  - Target user ID: `U08P301T9HA`
  - Message text contains the Google Sheet URL
  - Includes link back to the workflow in Slack message options
  - `executeOnce: true`
- **Key expressions or variables used:**
  - Static text: `New Roofing Contractor Found: <sheet-url>`
- **Input and output connections:**
  - Input: Send Gmail Alert
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
  - Requires Slack API credential
- **Edge cases or potential failure types:**
  - Invalid Slack user ID or insufficient bot permissions
  - Workspace/app installation issues
  - Because `executeOnce` is true, only one Slack message is sent per execution
- **Sub-workflow reference:** none

---

## 2.7 Documentation / Sticky Notes Layer

**Overview:**  
These nodes are non-executing annotations used to explain the purpose, setup, and organization of the workflow. They are important for human maintainers and should be preserved when reproducing the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation block explaining the entire workflow, setup steps, and customization guidance.
- **Configuration choices:**
  - Large introductory note with workflow summary, setup links, and customization suggestions
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none at runtime
- **Sub-workflow reference:** none

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Section label for input/configuration
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Section label for deep scraping, includes ScrapeOps Proxy link
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Section label for deduplication logic
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Section label for save-and-alert stage
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form: Enter City to Search | Form Trigger | Workflow entry point; captures the target city from a web form |  | Set Google Maps Configuration | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 1. Input & Configuration<br>Capture the target city via form and set the Google Maps search keyword and parameters. |
| Set Google Maps Configuration | Set | Normalizes city input and builds the Google Maps search URL | Form: Enter City to Search | ScrapeOps: Search Google Maps | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 1. Input & Configuration<br>Capture the target city via form and set the Google Maps search keyword and parameters. |
| ScrapeOps: Search Google Maps | @scrapeops/n8n-nodes-scrapeops.ScrapeOps | Fetches the rendered Google Maps search results page | Set Google Maps Configuration |  Parse Business Listings | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 2. Deep Scrape Google Maps<br>Search Maps for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
|  Parse Business Listings | Code | Extracts listing-level business data from Google Maps search HTML | ScrapeOps: Search Google Maps |  ScrapeOps: Fetch Business Details | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 2. Deep Scrape Google Maps<br>Search Maps for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
|  ScrapeOps: Fetch Business Details | @scrapeops/n8n-nodes-scrapeops.ScrapeOps | Fetches each individual business detail page from its Google Maps URL |  Parse Business Listings | Parse Full Business Info | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 2. Deep Scrape Google Maps<br>Search Maps for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
| Parse Full Business Info | Code | Matches detail-page HTML to listings and produces normalized business records with reviews |  ScrapeOps: Fetch Business Details | Read Previous Entries from Sheet | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 2. Deep Scrape Google Maps<br>Search Maps for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
| Read Previous Entries from Sheet | Google Sheets | Reads existing lead rows from Google Sheets for deduplication | Parse Full Business Info | Compare & Deduplicate Leads | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 3. Deduplicate Leads<br>Load existing sheet entries, compare against new results, and keep only leads not previously saved. |
| Compare & Deduplicate Leads | Code | Marks scraped businesses as new or old by comparing against prior sheet rows | Read Previous Entries from Sheet | Filter New Leads Only | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 3. Deduplicate Leads<br>Load existing sheet entries, compare against new results, and keep only leads not previously saved. |
| Filter New Leads Only | If | Passes only leads with status `new` to the save step | Compare & Deduplicate Leads | Save New Leads to Sheet | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 3. Deduplicate Leads<br>Load existing sheet entries, compare against new results, and keep only leads not previously saved. |
| Save New Leads to Sheet | Google Sheets | Appends only new leads to the destination sheet | Filter New Leads Only | Send Gmail Alert | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 4. Save & Alert<br>Append new roofing contractor leads to Google Sheets and send notifications via Gmail and Slack. |
| Send Gmail Alert | Gmail | Sends a one-time email notification with the sheet link | Save New Leads to Sheet |  Send Slack Alert | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 4. Save & Alert<br>Append new roofing contractor leads to Google Sheets and send notifications via Gmail and Slack. |
|  Send Slack Alert | Slack | Sends a one-time Slack notification with the sheet link | Send Gmail Alert |  | # 🏠 Roofing Contractor Finder → Google Sheets + Alerts<br>This workflow automates finding roofing contractors in any city by scraping Google Maps. It performs a deep scrape to extract business name, phone, website, rating, reviews, and address; cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps listings for roofing contractors via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out businesses already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen contractors.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for alert nodes.<br>- Open the form URL and enter a city to run.<br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find plumbers, electricians, HVAC contractors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a schedule.<br>## 4. Save & Alert<br>Append new roofing contractor leads to Google Sheets and send notifications via Gmail and Slack. |
| Sticky Note | Sticky Note | Top-level workflow documentation |  |  |  |
| Sticky Note1 | Sticky Note | Visual section header for input/configuration |  |  |  |
| Sticky Note2 | Sticky Note | Visual section header for scraping block |  |  |  |
| Sticky Note3 | Sticky Note | Visual section header for deduplication block |  |  |  |
| Sticky Note4 | Sticky Note | Visual section header for save/alert block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Roofing Contractor Finder with ScrapeOps & Google Maps`.
   - Keep execution order at the default compatible mode used here: `v1`.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: `Form: Enter City to Search`
   - Set a form title such as `Tell Which City`
   - Add one required field:
     - Label: `City Name`
   - Save the node so n8n generates the form/webhook URL.

3. **Add a Set node for search configuration**
   - Node type: **Set**
   - Name: `Set Google Maps Configuration`
   - Create these fields:
     - `city` = `{{ $json.city || $json['City Name'] || '' }}`
     - `keyword` = `roofing contractor`
     - `baseUrl` = `https://www.google.com/maps/search/`
     - `searchUrl` = `{{ 'https://www.google.com/maps/search/roofing+contractor+in+' + String($json.city || $json['City Name'] || '').replace(/\s+/g, '+') }}`
   - Connect:
     - `Form: Enter City to Search` → `Set Google Maps Configuration`

4. **Add the first ScrapeOps node**
   - Node type: **ScrapeOps**
   - Name: `ScrapeOps: Search Google Maps`
   - Configure:
     - URL: `{{ $json.searchUrl }}`
     - Return type: `htmlResponse`
     - Advanced options:
       - `render_js` = true
       - `wait` = `12000`
       - `residential_proxy` = false
   - Create/select ScrapeOps credentials:
     - Credential type: ScrapeOps API
     - Paste your ScrapeOps API key
   - Connect:
     - `Set Google Maps Configuration` → `ScrapeOps: Search Google Maps`

5. **Add the first Code node**
   - Node type: **Code**
   - Name: ` Parse Business Listings`
   - Paste the parsing script from the workflow logic, or rebuild its behavior with these requirements:
     - Validate city exists
     - Read HTML from `response`, `body`, `data`, or raw string
     - Limit HTML size to 1,000,000 chars
     - Extract business listings from Google Maps search results
     - Extract at least:
       - `businessName`
       - `phone`
       - `website`
       - `rating`
       - `totalReviews`
       - `address`
       - `city`
       - `mapUrl`
       - `category`
       - `checkedAt`
     - Return an error-style item if no valid businesses are found
   - Important:
     - Keep the exact node name if later nodes use `$()` references with that name, including leading spaces if you want full parity with the original.
   - Connect:
     - `ScrapeOps: Search Google Maps` → ` Parse Business Listings`

6. **Add the second ScrapeOps node**
   - Node type: **ScrapeOps**
   - Name: ` ScrapeOps: Fetch Business Details`
   - Configure:
     - URL: `{{ $json.mapUrl }}`
     - Return type: `htmlResponse`
     - Advanced options:
       - `render_js` = true
       - `wait` = `18000`
       - `residential_proxy` = false
   - Use the same ScrapeOps credential as above.
   - Connect:
     - ` Parse Business Listings` → ` ScrapeOps: Fetch Business Details`

7. **Add the second Code node for normalization**
   - Node type: **Code**
   - Name: `Parse Full Business Info`
   - Rebuild logic with these requirements:
     - Process all input items with `$input.all()`
     - Read deep HTML from the ScrapeOps detail response
     - Match each detail page back to its original listing using:
       - `mapUrl`
       - place ID extraction
       - item index fallback
     - Emit `MATCHING_ERROR` if correlation fails
     - Extract or normalize:
       - `businessName`
       - `phone`
       - `website`
       - `rating`
       - `totalReviews`
       - `address`
       - `city`
       - `mapUrl`
       - `category`
       - `checkedAt`
       - `review1`
       - `review2`
       - `review3`
     - Re-extract rating and total reviews from detail-page HTML only
     - Limit HTML to 2,000,000 chars
   - If reproducing exactly, preserve references to:
     - `$(' Parse Business Listings')`
     - `$(' ScrapeOps: Fetch Business Details')`
   - Connect:
     - ` ScrapeOps: Fetch Business Details` → `Parse Full Business Info`

8. **Prepare the Google Sheet**
   - Duplicate or create a spreadsheet with columns matching the data model.
   - Recommended columns:
     - `businessName`
     - `phone`
     - `website`
     - `rating`
     - `totalReviews`
     - `address`
     - `city`
     - `category`
     - `mapUrl`
     - `checkedAt`
     - `status`
     - `review1`
     - `review2`
     - `review3`
   - Share the sheet with the Google account used by n8n.

9. **Add a Google Sheets read node**
   - Node type: **Google Sheets**
   - Name: `Read Previous Entries from Sheet`
   - Operation: read/get rows
   - Select the spreadsheet and the target tab, e.g. `gid=0`
   - Enable:
     - `Always Output Data`
     - `Continue On Fail`
   - Configure Google Sheets OAuth2 credentials with access to the sheet.
   - Connect:
     - `Parse Full Business Info` → `Read Previous Entries from Sheet`

10. **Add the deduplication Code node**
    - Node type: **Code**
    - Name: `Compare & Deduplicate Leads`
    - Implement logic to:
      - Read current leads from `Parse Full Business Info`
      - Read historical rows from `Read Previous Entries from Sheet`
      - Normalize business names to lowercase trimmed values
      - Normalize phone numbers to digits with optional leading `+`
      - Compare current leads to prior rows
      - Mark each row with:
        - `status = new` if unseen
        - `status = old` if already present
      - Deduplicate repeated businesses within the current run
    - Preserve output fields:
      - `businessName`, `phone`, `website`, `rating`, `totalReviews`, `address`, `city`, `mapUrl`, `category`, `status`, `checkedAt`, `review1`, `review2`, `review3`
    - Connect:
      - `Read Previous Entries from Sheet` → `Compare & Deduplicate Leads`

11. **Add an IF node**
    - Node type: **IF**
    - Name: `Filter New Leads Only`
    - Condition:
      - left value: `{{ $json.status }}`
      - operator: equals
      - right value: `new`
    - Connect:
      - `Compare & Deduplicate Leads` → `Filter New Leads Only`

12. **Add a Google Sheets append node**
    - Node type: **Google Sheets**
    - Name: `Save New Leads to Sheet`
    - Operation: `Append`
    - Select the same spreadsheet and tab
    - Map columns manually:
      - `businessName` ← `{{ $json.businessName }}`
      - `phone` ← `{{ $json.phone }}`
      - `website` ← `{{ $json.website }}`
      - `rating` ← `{{ $json.rating }}`
      - `totalReviews` ← `{{ $json.totalReviews }}`
      - `address` ← `{{ $json.address }}`
      - `city` ← `{{ $json.city }}`
      - `category` ← `{{ $json.category }}`
      - `mapUrl` ← `{{ $json.mapUrl }}`
      - `checkedAt` ← `{{ $json.checkedAt }}`
      - `status` ← `{{ $json.status }}`
      - `review1` ← `{{ $json.review1 }}`
      - `review2` ← `{{ $json.review2 }}`
      - `review3` ← `{{ $json.review3 }}`
    - Enable `Continue On Fail` if you want parity with the original.
    - Connect the **true** output of:
      - `Filter New Leads Only` → `Save New Leads to Sheet`

13. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: `Send Gmail Alert`
    - Configure Gmail OAuth2 credentials
    - Set:
      - To: replace placeholder with a real recipient
      - Subject: `{{ '📍 New Businesses Found in ' + $('Set Google Maps Configuration').first().json.city + ' (Roofing Contractor)' }}`
      - Body (text mode): something like  
        `{{ 'New Roofing Contractor Found\n\nSheet URL: https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit?gid=0#gid=0' }}`
      - Email type: `text`
      - Append attribution: false
    - Enable `Execute Once`
    - Connect:
      - `Save New Leads to Sheet` → `Send Gmail Alert`

14. **Add a Slack node**
    - Node type: **Slack**
    - Name: ` Send Slack Alert`
    - Configure Slack credentials
    - Set message text to include the sheet link, for example:
      - `New Roofing Contractor Found: https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit?gid=0#gid=0`
    - Choose whether to send to:
      - a user
      - a channel
    - If reproducing exactly, the original uses user selection by ID and includes the workflow link.
    - Enable `Execute Once`
    - Connect:
      - `Send Gmail Alert` → ` Send Slack Alert`

15. **Add optional sticky notes for maintainability**
    - Add a large overview note describing the end-to-end purpose
    - Add section notes for:
      - Input & Configuration
      - Deep Scrape Google Maps
      - Deduplicate Leads
      - Save & Alert

16. **Test the workflow**
    - Open the form URL
    - Submit a city, e.g. `Dallas`
    - Confirm:
      - Search HTML is returned
      - Listings are extracted
      - Detail pages are fetched
      - Final rows are tagged `new` or `old`
      - Only `new` items reach the append node
      - Gmail and Slack send exactly one notification each per run

17. **Recommended hardening before production**
    - Replace placeholder Gmail recipient
    - Replace static sheet URLs with your own duplicated sheet
    - Consider increasing ScrapeOps wait time if Maps pages load incompletely
    - Consider adding an error branch after the parsing nodes
    - Consider adding a Schedule Trigger if you want automated recurring searches
    - Consider cleaning the node names to remove leading spaces, but if you do, also update every `$()` node reference in Code nodes

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Register for a free ScrapeOps API key | https://scrapeops.io/app/register/n8n |
| ScrapeOps n8n overview documentation | https://scrapeops.io/docs/n8n/overview/ |
| ScrapeOps Proxy documentation referenced in the workflow | https://scrapeops.io/docs/n8n/proxy-api/ |
| Google Sheet template referenced by the workflow | https://docs.google.com/spreadsheets/d/16oOK5vqHRua4e0tSaywCjwVA0tXQN61Tl2PkfQ487pU/edit?gid=0#gid=0 |
| Customization note: change the `keyword` field in `Set Google Maps Configuration` to target other trades | Applies to lead type reuse |
| Customization note: replace the Form Trigger with a Schedule Trigger for automated runs | Applies to scheduling/automation strategy |
| Alerts are summary-style because Gmail and Slack nodes are configured with `executeOnce: true` | Operational behavior |
| The workflow relies heavily on regex against Google Maps HTML, so it is sensitive to frontend markup changes | Maintenance consideration |
| Some node names intentionally include leading spaces, and the Code nodes reference them exactly via `$()` | Important when renaming nodes |