Scrape Google Maps realtor leads with ScrapeOps, Google Sheets, Gmail and Slack

https://n8nworkflows.xyz/workflows/scrape-google-maps-realtor-leads-with-scrapeops--google-sheets--gmail-and-slack-14111


# Scrape Google Maps realtor leads with ScrapeOps, Google Sheets, Gmail and Slack

# 1. Workflow Overview

This workflow collects real estate agent leads from Google Maps for a user-specified city, enriches each result with deeper business details, removes duplicates against an existing Google Sheet, stores only new leads, and sends email and Slack notifications.

Its main use case is local lead generation: a user opens a form, enters a city, and the workflow searches Google Maps for “real estate agent” businesses in that city. It then performs a second scrape per listing to capture richer information such as phone, website, address, ratings, and up to three review texts.

## 1.1 Input Reception and Search Configuration
The workflow starts with a form where the user submits a city name. A Set node then standardizes the city value and builds the Google Maps search URL.

## 1.2 Google Maps Search Scraping
The workflow uses ScrapeOps to load the Google Maps search results page with JavaScript rendering enabled. A Code node parses the returned HTML and extracts candidate business listings.

## 1.3 Business Detail Deep Scraping and Parsing
For each extracted business listing, the workflow requests the detail page URL via ScrapeOps again. Another Code node correlates each detail response back to its original listing and extracts normalized structured business data including reviews and LGBTQ+-friendly indicators.

## 1.4 Deduplication Against Google Sheets
The workflow reads previously saved rows from a Google Sheet, compares newly scraped businesses against historical entries, and marks each result as `new` or `old`.

## 1.5 New Lead Filtering, Persistence, and Notifications
Only leads marked `new` pass through an IF node. These records are appended to Google Sheets, then a Gmail alert and Slack message are sent.

## 1.6 Embedded Documentation
The workflow also contains multiple Sticky Note nodes that explain the overall purpose, setup, and the visual grouping of logical sections in the canvas.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Search Configuration

### Overview
This block captures the target city through an n8n form and prepares all Google Maps search parameters required downstream. It standardizes the search keyword and generates a Google Maps search URL.

### Nodes Involved
- Form: Enter City to Search
- Set Google Maps Configuration

### Node Details

#### Form: Enter City to Search
- **Type and technical role:** `n8n-nodes-base.formTrigger`; entry point that exposes a hosted form and triggers the workflow on submission.
- **Configuration choices:**
  - Form title: `Tell Which City`
  - One required field: `City Name`
  - Custom path is set using the webhook path value.
- **Key expressions or variables used:**
  - Outputs a field named `City Name`.
- **Input and output connections:**
  - No input; this is the workflow entry point.
  - Outputs to `Set Google Maps Configuration`.
- **Version-specific requirements:**
  - Uses `typeVersion: 1`.
  - Requires n8n instance support for Form Trigger nodes.
- **Edge cases or potential failure types:**
  - If the form is inactive or workflow is not active, public submission may not work.
  - Missing city cannot occur through the form because the field is required, but blank-like input may still be entered by users.
- **Sub-workflow reference:** None.

#### Set Google Maps Configuration
- **Type and technical role:** `n8n-nodes-base.set`; creates normalized config fields for the rest of the flow.
- **Configuration choices:**
  - Sets:
    - `city` from either `$json.city` or `$json['City Name']`
    - `keyword` fixed to `real estate agent`
    - `baseUrl` fixed to `https://www.google.com/maps/search/`
    - `searchUrl` dynamically assembled as a Google Maps search query
- **Key expressions or variables used:**
  - `={{ $json.city || $json['City Name'] || '' }}`
  - `={{ 'https://www.google.com/maps/search/real+estate+agent+in+' + String($json.city || $json['City Name'] || '').replace(/\s+/g, '+') }}`
- **Input and output connections:**
  - Input from `Form: Enter City to Search`
  - Output to `ScrapeOps: Search Google Maps`
- **Version-specific requirements:**
  - Uses Set node `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - If the city string is empty, downstream parsing node returns an error item.
  - Search URL generation assumes a simple keyword; special characters in city names may produce imperfect Google Maps URLs.
- **Sub-workflow reference:** None.

---

## 2.2 Google Maps Search Scraping

### Overview
This block requests the Google Maps search results page and extracts listing-level business candidates from the rendered HTML. It is the first scraping stage and produces one item per business candidate.

### Nodes Involved
- ScrapeOps: Search Google Maps
- Parse Business Listings

### Node Details

#### ScrapeOps: Search Google Maps
- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`; fetches Google Maps search results through ScrapeOps proxy/rendering.
- **Configuration choices:**
  - URL comes from `searchUrl`
  - Return type: `htmlResponse`
  - JavaScript rendering enabled: `render_js: true`
  - Wait time: `12000`
  - Residential proxy disabled
- **Key expressions or variables used:**
  - `={{ $json.searchUrl }}`
- **Input and output connections:**
  - Input from `Set Google Maps Configuration`
  - Output to `Parse Business Listings`
- **Version-specific requirements:**
  - Uses custom ScrapeOps node `typeVersion: 1`
  - Requires ScrapeOps credentials configured in n8n
- **Edge cases or potential failure types:**
  - Invalid or expired ScrapeOps API credentials
  - HTTP blocking or anti-bot changes on Google Maps
  - JavaScript render timeout
  - Empty or partial HTML if page fails to render fully
- **Sub-workflow reference:** None.

#### Parse Business Listings
- **Type and technical role:** `n8n-nodes-base.code`; parses Google Maps search-result HTML into structured listing records.
- **Configuration choices:**
  - Validates city presence using config from `Set Google Maps Configuration`
  - Reads HTML from several possible fields: `response`, raw string JSON, `body`, or `data`
  - Truncates HTML to 1,000,000 characters to reduce memory pressure
  - Extracts listing identifiers primarily via Maps place URLs
  - Falls back to business-card-like HTML structures if URLs are not found directly
  - Extracts:
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
- **Key expressions or variables used:**
  - Accesses config using `$('Set Google Maps Configuration').item.json`
  - Uses regex-heavy parsing for:
    - map URLs
    - business names
    - phone numbers
    - addresses
    - websites
    - ratings and review counts
- **Input and output connections:**
  - Input from `ScrapeOps: Search Google Maps`
  - Output to `ScrapeOps: Fetch Business Details`
- **Version-specific requirements:**
  - Uses Code node `typeVersion: 2`
  - Relies on n8n expression access to another node via `$()`
- **Edge cases or potential failure types:**
  - Returns an error item if city is invalid or no businesses are found
  - HTML structure changes in Google Maps can break regex extraction
  - If map URLs are missing, fallback card parsing may be less reliable
  - Code mutates `currentMapUrl` after destructuring from loop data; depending on JS engine semantics in n8n, this is valid because it is a local binding, but parsing reliability still depends on extracted card context
  - Extracted phone/address/website can contain false positives due to HTML ambiguity
- **Sub-workflow reference:** None.

---

## 2.3 Business Detail Deep Scraping and Parsing

### Overview
This block visits each listing’s Maps detail URL and extracts deeper attributes not reliably available in the search result cards. It also attempts to match each detail-page HTML response back to the corresponding listing item before normalizing the final lead object.

### Nodes Involved
- ScrapeOps: Fetch Business Details
- Parse Full Business Info

### Node Details

#### ScrapeOps: Fetch Business Details
- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`; deep-scrapes each Google Maps business detail page.
- **Configuration choices:**
  - URL is taken from each listing’s `mapUrl`
  - Return type: `htmlResponse`
  - JavaScript rendering enabled
  - Wait time: `12000`
  - Residential proxy disabled
- **Key expressions or variables used:**
  - `={{ $json.mapUrl }}`
- **Input and output connections:**
  - Input from `Parse Business Listings`
  - Output to `Parse Full Business Info`
- **Version-specific requirements:**
  - Uses custom ScrapeOps node `typeVersion: 1`
  - Requires ScrapeOps credentials
- **Edge cases or potential failure types:**
  - Empty `mapUrl` values can cause invalid fetches
  - Anti-bot or CAPTCHA responses may produce unusable HTML
  - Rate limiting if many listings are processed
  - JS render and network timeouts
- **Sub-workflow reference:** None.

#### Parse Full Business Info
- **Type and technical role:** `n8n-nodes-base.code`; consolidates original listing data with detail-page scraping results and outputs one normalized business record per item.
- **Configuration choices:**
  - Processes all input items using `$input.all()`
  - Extracts HTML content from `response`, raw JSON, `body`, or `data`
  - Tries to reconstruct `mapUrl` from detail HTML if missing
  - Matches detail HTML to original listing data from `Parse Business Listings`
    - First by place ID extracted from map URLs
    - Then by comparing corresponding ScrapeOps detail responses
    - Then by item index fallback
  - Extracts or refines:
    - `businessName`
    - `phone`
    - `website`
    - `rating`
    - `totalReviews`
    - `address`
    - `city`
    - `mapUrl`
    - `category`
    - `lgbtqFriendly`
    - `review1`, `review2`, `review3`
    - `checkedAt`
  - Truncates HTML at 2,000,000 characters to reduce memory overhead
- **Key expressions or variables used:**
  - `$('Parse Business Listings')`
  - `$('ScrapeOps: Fetch Business Details')`
  - Multiple regex patterns for:
    - place URLs and IDs
    - phone extraction
    - website extraction
    - address extraction
    - LGBTQ+ indicator detection
    - rating/review count extraction
    - review text extraction from several sources:
      - expanded text attributes
      - original text attributes
      - review blocks
      - multiple CSS classes
      - JSON-LD
      - meta tags
- **Input and output connections:**
  - Input from `ScrapeOps: Fetch Business Details`
  - Output to `Read Previous Entries from Sheet`
- **Version-specific requirements:**
  - Uses Code node `typeVersion: 2`
  - Depends on cross-node access features in Code nodes
- **Edge cases or potential failure types:**
  - If matching fails, outputs a `MATCHING_ERROR` item
  - If Google Maps HTML changes, regexes may fail silently or return partial data
  - Reviews may be duplicated or contain UI text despite cleaning logic
  - Large HTML may still challenge execution memory for large batches
  - Fallback matching by index assumes relative order remains stable
- **Sub-workflow reference:** None.

---

## 2.4 Deduplication Against Google Sheets

### Overview
This block loads previously stored leads from Google Sheets and compares them with the newly normalized business records. It marks each current lead as `new` or `old` based primarily on normalized business name and phone.

### Nodes Involved
- Read Previous Entries from Sheet
- Compare & Deduplicate Leads

### Node Details

#### Read Previous Entries from Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads all existing entries from the configured Google Sheet tab.
- **Configuration choices:**
  - Spreadsheet specified by URL
  - Sheet tab selected from list: `Real Estate Agent Finder`
  - No special read filters configured
  - `continueOnFail: true`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - No custom expressions in parameters
- **Input and output connections:**
  - Input from `Parse Full Business Info`
  - Output to `Compare & Deduplicate Leads`
- **Version-specific requirements:**
  - Uses Google Sheets node `typeVersion: 4.4`
  - Requires Google Sheets OAuth2 credential
- **Edge cases or potential failure types:**
  - Spreadsheet not found or access denied
  - Wrong worksheet selected
  - Empty sheet results in no historical rows, which the compare code handles
  - Because `continueOnFail` and `alwaysOutputData` are enabled, downstream logic may receive error-shaped items instead of a hard stop
- **Sub-workflow reference:** None.

#### Compare & Deduplicate Leads
- **Type and technical role:** `n8n-nodes-base.code`; compares current parsed results with previously saved sheet rows and adds a `status` field.
- **Configuration choices:**
  - Pulls current items directly from `Parse Full Business Info`
  - Pulls historical items directly from `Read Previous Entries from Sheet`
  - Normalizes:
    - phone by stripping formatting except leading `+`
    - business name by lowercasing and collapsing whitespace
  - Builds a set of previous keys:
    - `normalizedName||normalizedPhone`
    - and, in some cases, plain `normalizedName`
  - Removes duplicates within the current execution using a `seen` set
  - Emits each item with `status: 'new'` or `status: 'old'`
- **Key expressions or variables used:**
  - `$('Parse Full Business Info')`
  - `$('Read Previous Entries from Sheet')`
- **Input and output connections:**
  - Input from `Read Previous Entries from Sheet`
  - Output to `Filter New Leads Only`
- **Version-specific requirements:**
  - Uses Code node `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If access to upstream node data fails, returns an error-shaped item
  - If no current businesses are available, returns a sentinel item marked `old`
  - Name-only matching can treat some legitimate distinct leads as duplicates if they share the same normalized business name
  - Historical column naming flexibility is implemented, but unexpected sheet headers may still reduce deduplication quality
- **Sub-workflow reference:** None.

---

## 2.5 New Lead Filtering, Persistence, and Notifications

### Overview
This block keeps only leads marked as new, writes them to Google Sheets, and sends one Gmail and one Slack notification for the run.

### Nodes Involved
- Filter New Leads Only
- Save New Leads to Sheet
- Send Gmail Alert
- Send Slack Alert

### Node Details

#### Filter New Leads Only
- **Type and technical role:** `n8n-nodes-base.if`; keeps only items where `status` equals `new`.
- **Configuration choices:**
  - String condition: `$json.status == 'new'`
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input from `Compare & Deduplicate Leads`
  - True output goes to `Save New Leads to Sheet`
  - No false-branch connection is configured
- **Version-specific requirements:**
  - Uses IF node `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Items missing `status` are treated as non-matching and silently dropped
  - Sentinel error/empty items with `status: old` do not continue
- **Sub-workflow reference:** None.

#### Save New Leads to Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends only new lead records to the Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet and tab point to the same `Real Estate Agent Finder` sheet
  - Explicit field mapping for:
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
    - `lgbtqFriendly`
    - `status`
    - `review1`
    - `review2`
    - `review3`
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - Each column maps directly from `$json.<field>`
- **Input and output connections:**
  - Input from `Filter New Leads Only`
  - Output to ` Send Gmail Alert`
- **Version-specific requirements:**
  - Uses Google Sheets node `typeVersion: 4.4`
  - Requires Google Sheets OAuth2 credential
- **Edge cases or potential failure types:**
  - Sheet schema/header mismatch can cause missing writes or field drops
  - Access denied or wrong document ID
  - Because `continueOnFail` is enabled, alerts may still run even if some rows fail to append
- **Sub-workflow reference:** None.

#### Send Gmail Alert
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an email notification after new leads are saved.
- **Configuration choices:**
  - Recipient is hardcoded: `example@.com`
  - Subject includes searched city from `Set Google Maps Configuration`
  - Message body includes static text and a sheet URL
  - Plain-text email
  - `appendAttribution` disabled
  - `executeOnce: true`
- **Key expressions or variables used:**
  - Subject:
    - `={{ '📍 New Businesses Found in ' + $('Set Google Maps Configuration').first().json.city + ' (Real Estate Agent)' }}`
  - Message:
    - `={{ 'New Real Estate Agent Found\n\n' + 'Sheet URL: ' + 'https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0' }}`
- **Input and output connections:**
  - Input from `Save New Leads to Sheet`
  - Output to `Send Slack Alert`
- **Version-specific requirements:**
  - Uses Gmail node `typeVersion: 2.1`
  - Requires Gmail OAuth2 credential
- **Edge cases or potential failure types:**
  - Recipient email is placeholder-like and may be invalid until replaced
  - Gmail OAuth token expiry or insufficient permissions
  - Because `executeOnce` is true, only one email is sent per execution even if many leads are appended
- **Sub-workflow reference:** None.

#### Send Slack Alert
- **Type and technical role:** `n8n-nodes-base.slack`; sends a Slack notification after the Gmail alert.
- **Configuration choices:**
  - Message text contains a static sheet link
  - Target selected as a Slack user ID, not a channel
  - `includeLinkToWorkflow` enabled
  - `executeOnce: true`
- **Key expressions or variables used:**
  - Static text only in this configuration
- **Input and output connections:**
  - Input from ` Send Gmail Alert`
  - No further outputs
- **Version-specific requirements:**
  - Uses Slack node `typeVersion: 2.3`
  - Requires Slack API credential
- **Edge cases or potential failure types:**
  - Slack user ID may be invalid or inaccessible for the app
  - If the intended target is a channel, current configuration would need to change
  - Because `executeOnce` is true, one Slack message is sent per run rather than per lead
- **Sub-workflow reference:** None.

---

## 2.6 Embedded Documentation and Canvas Notes

### Overview
These nodes do not execute business logic but document the workflow purpose and visual organization. They are useful for maintainers and AI agents interpreting the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; high-level descriptive note covering workflow purpose, setup, and customization.
- **Configuration choices:**
  - Large note with setup links and usage guidance
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** sticky note for block 1
- **Configuration choices:** labels input/configuration area
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** sticky note for block 2
- **Configuration choices:** labels deep scraping area and includes ScrapeOps Proxy link
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** sticky note for block 3
- **Configuration choices:** labels deduplication area
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** sticky note for block 4
- **Configuration choices:** labels save/alert area
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form: Enter City to Search | formTrigger | Workflow entry point; collects city name from user form |  | Set Google Maps Configuration | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 1. Input & Configuration<br>Capture the target city via form and set the Google Maps search keyword and parameters. |
| Set Google Maps Configuration | set | Normalizes city and builds Google Maps search URL | Form: Enter City to Search | ScrapeOps: Search Google Maps | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 1. Input & Configuration<br>Capture the target city via form and set the Google Maps search keyword and parameters. |
| ScrapeOps: Search Google Maps | @scrapeops/n8n-nodes-scrapeops.ScrapeOps | Fetches rendered Google Maps search results HTML | Set Google Maps Configuration | Parse Business Listings | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 2. Deep Scrape Google Maps<br>Search Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
| Parse Business Listings | code | Parses Google Maps search results into listing records | ScrapeOps: Search Google Maps | ScrapeOps: Fetch Business Details | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 2. Deep Scrape Google Maps<br>Search Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
| ScrapeOps: Fetch Business Details | @scrapeops/n8n-nodes-scrapeops.ScrapeOps | Fetches rendered business detail page HTML for each listing | Parse Business Listings | Parse Full Business Info | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 2. Deep Scrape Google Maps<br>Search Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
| Parse Full Business Info | code | Correlates listing/detail HTML and outputs normalized business records | ScrapeOps: Fetch Business Details | Read Previous Entries from Sheet | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 2. Deep Scrape Google Maps<br>Search Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/), then deep-scrape each listing for phone, website, reviews, and address. |
| Read Previous Entries from Sheet | googleSheets | Loads historical leads from Google Sheets | Parse Full Business Info | Compare & Deduplicate Leads | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 3. Deduplicate Leads<br>Load existing sheet entries, compare against new results, and keep only agents not previously saved. |
| Compare & Deduplicate Leads | code | Marks scraped leads as new or old versus sheet history | Read Previous Entries from Sheet | Filter New Leads Only | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 3. Deduplicate Leads<br>Load existing sheet entries, compare against new results, and keep only agents not previously saved. |
| Filter New Leads Only | if | Keeps only records with status=new | Compare & Deduplicate Leads | Save New Leads to Sheet | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 3. Deduplicate Leads<br>Load existing sheet entries, compare against new results, and keep only agents not previously saved. |
| Save New Leads to Sheet | googleSheets | Appends new leads to Google Sheets | Filter New Leads Only |  Send Gmail Alert | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 4. Save & Alert<br>Append new agent leads to Google Sheets and send notifications via Gmail and Slack. |
|  Send Gmail Alert | gmail | Sends one email alert summarizing new leads | Save New Leads to Sheet | Send Slack Alert | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 4. Save & Alert<br>Append new agent leads to Google Sheets and send notifications via Gmail and Slack. |
| Send Slack Alert | slack | Sends one Slack notification with sheet link |  Send Gmail Alert |  | # 🏘️ Real Estate Agent Finder → Google Sheets + Alerts<br>This workflow automates finding real estate agents in any city by scraping Google Maps. It performs a deep scrape to extract agent name, phone, website, rating, reviews, and address — cross-references results with Google Sheets to remove duplicates — then saves fresh leads and sends alerts via Gmail and Slack.<br><br>### How it works<br>1. 📝 **Form: Enter City to Search** captures the target city from a simple web form.<br>2. ⚙️ **Set Google Maps Configuration** sets the search keyword and location parameters.<br>3. 🌐 **ScrapeOps: Search Google Maps** scrapes Google Maps for real estate agents via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **Parse Business Listings** extracts name, address, rating, and Maps URL from results.<br>5. 📡 **ScrapeOps: Fetch Business Details** deep-scrapes each listing for phone, website, reviews, and more.<br>6. 🗺️ **Parse Full Business Info** normalizes all fields into a clean structured record.<br>7. 📂 **Read Previous Entries from Sheet** loads existing leads to check for duplicates.<br>8. 🧹 **Compare & Deduplicate Leads** filters out agents already saved in the sheet.<br>9. 🔀 **Filter New Leads Only** keeps only fresh, unseen results.<br>10. 💾 **Save New Leads to Sheet** appends new leads to Google Sheets.<br>11. 📧 **Send Gmail Alert** notifies you of new leads by email.<br>12. 📣 **Send Slack Alert** posts a summary to your Slack channel.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0) and connect it to the Sheet nodes.<br>- Configure Gmail and Slack credentials for the alert nodes.<br>- Open the form URL, enter a city, and run.<br><br>### Customization<br>- Change the `keyword` in **Set Google Maps Configuration** to find property managers, mortgage brokers, home inspectors, etc.<br>- Replace the Form Trigger with a Schedule Trigger to run automatically on a recurring schedule.<br><br>## 4. Save & Alert<br>Append new agent leads to Google Sheets and send notifications via Gmail and Slack. |
| Sticky Note | stickyNote | Canvas documentation and setup note |  |  |  |
| Sticky Note1 | stickyNote | Visual label for input/config block |  |  |  |
| Sticky Note2 | stickyNote | Visual label for scraping block |  |  |  |
| Sticky Note3 | stickyNote | Visual label for deduplication block |  |  |  |
| Sticky Note4 | stickyNote | Visual label for save/alert block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Real Estate Agent Finder with ScrapeOps and Google Sheets`.
   - Keep execution order at the default compatible setting if available; this source workflow uses `executionOrder: v1`.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: `Form: Enter City to Search`
   - Set form title to `Tell Which City`
   - Add one required field:
     - Label: `City Name`
   - Save the node.
   - If you want public usage, later activate the workflow to expose the form URL.

3. **Add a Set node for search configuration**
   - Node type: **Set**
   - Name: `Set Google Maps Configuration`
   - Add the following fields:
     1. `city` as string:
        - `={{ $json.city || $json['City Name'] || '' }}`
     2. `keyword` as string:
        - `real estate agent`
     3. `baseUrl` as string:
        - `https://www.google.com/maps/search/`
     4. `searchUrl` as string:
        - `={{ 'https://www.google.com/maps/search/real+estate+agent+in+' + String($json.city || $json['City Name'] || '').replace(/\s+/g, '+') }}`
   - Connect `Form: Enter City to Search` → `Set Google Maps Configuration`.

4. **Install and configure the ScrapeOps n8n node if not already installed**
   - Ensure the custom node package for ScrapeOps is available in your n8n environment.
   - Create ScrapeOps credentials using your ScrapeOps API key.
   - The setup note references:
     - Registration: `https://scrapeops.io/app/register/n8n`
     - Docs: `https://scrapeops.io/docs/n8n/overview/`

5. **Add the first ScrapeOps node for Google Maps search**
   - Node type: **ScrapeOps**
   - Name: `ScrapeOps: Search Google Maps`
   - URL:
     - `={{ $json.searchUrl }}`
   - Return type:
     - `htmlResponse`
   - Advanced options:
     - `wait`: `12000`
     - `render_js`: enabled
     - `residential_proxy`: disabled
   - Attach your ScrapeOps credentials.
   - Connect `Set Google Maps Configuration` → `ScrapeOps: Search Google Maps`.

6. **Add a Code node to parse search results**
   - Node type: **Code**
   - Name: `Parse Business Listings`
   - Paste the code logic from the workflow into the Code node.
   - This code must:
     - validate the city
     - read HTML from ScrapeOps response
     - trim oversized HTML
     - extract listing URLs and business data
     - output one item per candidate business
   - Connect `ScrapeOps: Search Google Maps` → `Parse Business Listings`.

7. **Add the second ScrapeOps node for detail-page scraping**
   - Node type: **ScrapeOps**
   - Name: `ScrapeOps: Fetch Business Details`
   - URL:
     - `={{ $json.mapUrl }}`
   - Return type:
     - `htmlResponse`
   - Advanced options:
     - `wait`: `12000`
     - `render_js`: enabled
     - `residential_proxy`: disabled
   - Use the same ScrapeOps credentials.
   - Connect `Parse Business Listings` → `ScrapeOps: Fetch Business Details`.

8. **Add a second Code node to normalize full business info**
   - Node type: **Code**
   - Name: `Parse Full Business Info`
   - Paste the code logic from the workflow into the Code node.
   - This code must:
     - process all incoming detail-page items
     - locate the source listing from `Parse Business Listings`
     - recover `mapUrl` if needed
     - extract phone, website, address, rating, total reviews
     - detect LGBTQ+-friendly patterns
     - extract up to three reviews
     - emit a normalized business record
   - Connect `ScrapeOps: Fetch Business Details` → `Parse Full Business Info`.

9. **Prepare the Google Sheet**
   - Duplicate or create a spreadsheet matching the expected columns.
   - Recommended sheet URL from the note:
     - `https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0`
   - Create a sheet tab named `Real Estate Agent Finder` or select your own tab.
   - Include headers for:
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
     - `lgbtqFriendly`
     - `status`
     - `review1`
     - `review2`
     - `review3`

10. **Create Google Sheets credentials**
    - In n8n, create **Google Sheets OAuth2** credentials.
    - Ensure the connected Google account has access to the target spreadsheet.

11. **Add a Google Sheets node to read prior entries**
    - Node type: **Google Sheets**
    - Name: `Read Previous Entries from Sheet`
    - Operation: read rows/default sheet read operation
    - Select your spreadsheet document
    - Select the target sheet tab: `Real Estate Agent Finder`
    - Enable:
      - `Continue On Fail`
      - `Always Output Data`
    - Connect `Parse Full Business Info` → `Read Previous Entries from Sheet`.

12. **Add a Code node for deduplication**
    - Node type: **Code**
    - Name: `Compare & Deduplicate Leads`
    - Paste the deduplication code from the workflow.
    - Ensure it:
      - loads current items from `Parse Full Business Info`
      - loads historical items from `Read Previous Entries from Sheet`
      - normalizes business names and phones
      - marks each item with `status: new` or `status: old`
      - preserves fields including reviews and `lgbtqFriendly`
    - Connect `Read Previous Entries from Sheet` → `Compare & Deduplicate Leads`.

13. **Add an IF node to keep only new leads**
    - Node type: **If**
    - Name: `Filter New Leads Only`
    - Condition:
      - compare string
      - left value: `={{ $json.status }}`
      - right value: `new`
    - Connect `Compare & Deduplicate Leads` → `Filter New Leads Only`.

14. **Add a Google Sheets node to append new rows**
    - Node type: **Google Sheets**
    - Name: `Save New Leads to Sheet`
    - Operation: `Append`
    - Select the same spreadsheet and same tab.
    - Use manual column mapping.
    - Map:
      - `businessName` → `={{ $json.businessName }}`
      - `phone` → `={{ $json.phone }}`
      - `website` → `={{ $json.website }}`
      - `rating` → `={{ $json.rating }}`
      - `totalReviews` → `={{ $json.totalReviews }}`
      - `address` → `={{ $json.address }}`
      - `city` → `={{ $json.city }}`
      - `category` → `={{ $json.category }}`
      - `mapUrl` → `={{ $json.mapUrl }}`
      - `checkedAt` → `={{ $json.checkedAt }}`
      - `lgbtqFriendly` → `={{ $json.lgbtqFriendly }}`
      - `status` → `={{ $json.status }}`
      - `review1` → `={{ $json.review1 }}`
      - `review2` → `={{ $json.review2 }}`
      - `review3` → `={{ $json.review3 }}`
    - Enable `Continue On Fail`.
    - Connect the **true** output of `Filter New Leads Only` → `Save New Leads to Sheet`.

15. **Create Gmail credentials**
    - In n8n, create **Gmail OAuth2** credentials.
    - Make sure the connected Gmail account is permitted to send mail.

16. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: ` Send Gmail Alert`
    - Set recipient:
      - Replace placeholder `example@.com` with a real destination address
    - Subject:
      - `={{ '📍 New Businesses Found in ' + $('Set Google Maps Configuration').first().json.city + ' (Real Estate Agent)' }}`
    - Email type:
      - `text`
    - Message:
      - `={{ 'New Real Estate Agent Found\n\n' + 'Sheet URL: ' + 'https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0' }}`
    - In options, disable attribution append if desired.
    - Set `Execute Once` to true.
    - Connect `Save New Leads to Sheet` → ` Send Gmail Alert`.

17. **Create Slack credentials**
    - In n8n, create **Slack API** credentials.
    - Ensure the Slack app has permission to message the intended target.

18. **Add a Slack node**
    - Node type: **Slack**
    - Name: `Send Slack Alert`
    - Message text:
      - `New Real Estate Agent Found: https://docs.google.com/spreadsheets/d/1C7OAR6d6bngkrCw-On7zoIYY0QjobLVdDaP_d7u-hKU/edit?gid=0#gid=0`
    - Select target type according to your needs:
      - In the source workflow, target is a **user** with a specific user ID
    - Enable `includeLinkToWorkflow` if you want a workflow link in the Slack message.
    - Set `Execute Once` to true.
    - Connect ` Send Gmail Alert` → `Send Slack Alert`.

19. **Optionally add sticky notes for maintainability**
    - Add a large overview note describing the workflow, setup links, and customization ideas.
    - Add smaller notes for:
      - `1. Input & Configuration`
      - `2. Deep Scrape Google Maps`
      - `3. Deduplicate Leads`
      - `4. Save & Alert`

20. **Test the workflow**
    - Run the form trigger manually or open the form URL.
    - Submit a city such as `Austin`, `Miami`, or `Denver`.
    - Check outputs at each stage:
      - `Set Google Maps Configuration`: verify `searchUrl`
      - `ScrapeOps: Search Google Maps`: verify HTML exists
      - `Parse Business Listings`: verify one item per business
      - `ScrapeOps: Fetch Business Details`: verify detail HTML
      - `Parse Full Business Info`: verify normalized records
      - `Compare & Deduplicate Leads`: verify `status`
      - `Save New Leads to Sheet`: verify rows appended
      - Alert nodes: verify email and Slack delivery

21. **Validate known constraints before production use**
    - Replace all placeholder destinations and sheet URLs if using your own assets.
    - Expect parsing drift if Google Maps changes HTML structure.
    - Keep ScrapeOps rendering enabled because Google Maps depends heavily on JavaScript.
    - Consider adding explicit error handling branches if you want alerts when scraping fails.
    - Consider replacing the Form Trigger with a Schedule Trigger if you want recurring automated searches.

22. **Activate the workflow**
    - Once tested, activate it so the form endpoint is live.
    - If you later switch to scheduled execution, deactivate the form trigger and introduce a schedule-based entry plus a city source.

## Credential Summary

- **ScrapeOps**
  - Required for both ScrapeOps nodes
  - Must support rendered HTML fetching
- **Google Sheets OAuth2**
  - Required for both sheet nodes
  - Needs read/write access to the spreadsheet
- **Gmail OAuth2**
  - Required for the Gmail alert node
- **Slack API**
  - Required for the Slack alert node

## Sub-workflow Setup
This workflow does **not** invoke any sub-workflow and has only one entry point:
- `Form: Enter City to Search`

## Output Expectations
For successful new leads, the final persisted schema is:
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
- `lgbtqFriendly`
- `status`
- `review1`
- `review2`
- `review3`

## Important Implementation Notes
- The workflow uses Code nodes heavily and depends on cross-node data access through `$('<Node Name>')`.
- Both parsing steps are regex-based and therefore sensitive to Google Maps DOM changes.
- Alerts are configured with `executeOnce`, so they notify once per run, not once per lead.
- The deduplication strategy is name/phone-centric and may need refinement for franchises or duplicate names across cities.