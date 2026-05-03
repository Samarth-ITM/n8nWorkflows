Track Redfin real estate listings with ScrapeOps, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/track-redfin-real-estate-listings-with-scrapeops--google-sheets--and-slack-13991


# Track Redfin real estate listings with ScrapeOps, Google Sheets, and Slack

# 1. Workflow Overview

This workflow monitors Redfin real estate listings on a recurring schedule, using ScrapeOps to fetch and parse listing pages, Google Sheets to store property rows, and Slack to send a completion message.

Its main use case is automated property tracking for a specific Redfin search URL such as a city, ZIP code, or filtered results page. It is designed for users who want periodic snapshots of listings without manually browsing Redfin.

The workflow is organized into four functional blocks:

## 1.1 Trigger & Configuration
The workflow starts on a 6-hour schedule and defines the Redfin search page to scrape.

## 1.2 Fetch & Parse Listings
It loads the Redfin page through ScrapeOps Proxy with JavaScript rendering enabled, then sends the resulting HTML to the ScrapeOps Parser API configured for the Redfin domain.

## 1.3 Transform & Filter
The parsed response is enriched with search metadata, split into one item per property, normalized into sheet-friendly fields, and filtered to keep only valid listings.

## 1.4 Save & Notify
Valid listings are appended to Google Sheets, and a Slack message is sent as a run summary.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Configuration

### Overview
This block defines when the workflow runs and which Redfin search page will be scraped. It acts as the entry point and runtime configuration layer.

### Nodes Involved
- ` Schedule Trigger`
- `Set Search Parameters`

### Node Details

#### ` Schedule Trigger`
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry node that launches the workflow automatically at a fixed interval.
- **Configuration choices:**
  - Runs every 6 hours.
- **Key expressions or variables used:**
  - No custom expression in the node itself.
  - Produces standard schedule metadata such as timestamp and readable date/time.
- **Input and output connections:**
  - No input, as it is a trigger.
  - Outputs to `Set Search Parameters`.
- **Version-specific requirements:**
  - Uses type version `1.1`.
- **Edge cases or potential failure types:**
  - Workflow must be activated for the schedule to run.
  - Server timezone may affect perceived execution timing.
  - Missed executions can happen if the n8n instance is offline.
- **Sub-workflow reference:**
  - None.

#### `Set Search Parameters`
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines the Redfin URL that downstream nodes use for scraping and parsing.
- **Configuration choices:**
  - Creates a single field:
    - `redfin_url = https://www.redfin.com/city/21853/MD/California`
- **Key expressions or variables used:**
  - Static string value for `redfin_url`.
- **Input and output connections:**
  - Input from ` Schedule Trigger`
  - Output to `ScrapeOps: Fetch Redfin Page (Proxy)`
- **Version-specific requirements:**
  - Uses type version `3.2`.
- **Edge cases or potential failure types:**
  - If the Redfin URL is invalid, changed, blocked, or points to an empty page, downstream scraping/parsing may fail or return no results.
  - If Redfin page structure changes significantly, parser output may degrade.
- **Sub-workflow reference:**
  - None.

---

## 2.2 Fetch & Parse Listings

### Overview
This block retrieves the target Redfin page with browser-like rendering and then extracts structured listing data using ScrapeOps parsing specialized for Redfin.

### Nodes Involved
- `ScrapeOps: Fetch Redfin Page (Proxy)`
- `ScrapeOps: Parse Redfin Listings`

### Node Details

#### `ScrapeOps: Fetch Redfin Page (Proxy)`
- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`  
  Uses ScrapeOps Proxy API to request the Redfin page with anti-bot and rendering options.
- **Configuration choices:**
  - URL comes from `{{$json.redfin_url}}`
  - Advanced options:
    - `wait: 5000`
    - `scroll: 2000`
    - `country: us`
    - `render_js: true`
    - `device_type: desktop`
    - `residential_proxy: true`
- **Key expressions or variables used:**
  - `={{ $json.redfin_url }}`
- **Input and output connections:**
  - Input from `Set Search Parameters`
  - Output to `ScrapeOps: Parse Redfin Listings`
- **Version-specific requirements:**
  - Uses type version `1`.
  - Requires the ScrapeOps n8n node package to be installed in the environment.
  - Requires ScrapeOps credentials configured in n8n.
- **Edge cases or potential failure types:**
  - ScrapeOps authentication failure or missing API key.
  - Redfin anti-bot response despite proxy usage.
  - Rendering timeout or incomplete page load.
  - Empty results if wait/scroll settings are insufficient.
  - Network errors, quota limits, or provider-side outage.
- **Sub-workflow reference:**
  - None.

#### `ScrapeOps: Parse Redfin Listings`
- **Type and technical role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`  
  Sends fetched page HTML to the ScrapeOps Parser API and requests structured extraction for the Redfin domain.
- **Configuration choices:**
  - `apiType = parserApi`
  - `parserUrl` uses the original search URL from `Set Search Parameters`
  - `parserHtml` uses the upstream fetched content
  - `parserDomain = redfin`
- **Key expressions or variables used:**
  - `={{ $('Set Search Parameters').item.json.redfin_url }}`
  - `={{ $json }}`
- **Input and output connections:**
  - Input from `ScrapeOps: Fetch Redfin Page (Proxy)`
  - Output to `Add Search Metadata`
- **Version-specific requirements:**
  - Uses type version `1`.
  - Also depends on the ScrapeOps node package and credentials.
- **Edge cases or potential failure types:**
  - Parser may fail if the upstream node does not return expected HTML/body format.
  - Parser output structure may change if ScrapeOps updates Redfin parsing schema.
  - Invalid parser domain or malformed HTML can produce empty `data.search_results`.
- **Sub-workflow reference:**
  - None.

---

## 2.3 Transform & Filter

### Overview
This block converts parser output into operational records. It extracts search-level metadata, splits the listing array into separate items, maps listing fields into normalized columns, and filters out unusable properties.

### Nodes Involved
- `Add Search Metadata`
- `Split Properties Into Items`
- `Format Property Fields`
- `Filter Valid Properties`

### Node Details

#### `Add Search Metadata`
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds summary data from the parsed Redfin response, including timestamps and search context.
- **Configuration choices:**
  - Creates fields intended to include:
    - `timestamp`
    - `search_title`
    - `search_type`
    - `total_properties`
    - region information
    - price range
    - average price
    - source URL
    - `search_results`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString() }}`
  - `={{ $json.data.search_information.search_title || 'N/A' }}`
  - `={{ $json.data.search_information.search_type || 'N/A' }}`
  - `={{ $json.data.search_information.total_count || 0 }}`
  - `={{ $json.data.search_information.region.name + ', ' + $json.data.search_information.region.state || 'N/A' }}`
  - `={{ ($json.data.search_information.min_price || 0) + ' - ' + ($json.data.search_information.max_price || 0) }}`
  - `={{ $json.data.search_information.average_price || 0 }}`
  - `={{ $json.url || 'N/A' }}`
  - `={{ $json.data.search_results }}`
- **Input and output connections:**
  - Input from `ScrapeOps: Parse Redfin Listings`
  - Output to `Split Properties Into Items`
- **Version-specific requirements:**
  - Uses type version `3.2`.
- **Edge cases or potential failure types:**
  - Several field names appear malformed in the configuration, with names starting with `=` or containing full expressions. This may create unexpected output keys in n8n.
  - The expression for region concatenation can error if `region` is missing, because `.name` and `.state` are accessed before fallback handling.
  - If `search_results` is missing or not valid JSON/array, the next node fails.
- **Sub-workflow reference:**
  - None.

#### `Split Properties Into Items`
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts the search results payload into one n8n item per property.
- **Configuration choices:**
  - JavaScript logic:
    - Reads `search_results` from the first input item.
    - If it is a string, parses it with `JSON.parse`.
    - Verifies the result is an array.
    - Returns each property as its own item.
- **Key expressions or variables used:**
  - `const searchResults = $input.first().json.search_results;`
  - `JSON.parse(searchResults)`
  - Throws `new Error('search_results is not an array')` if validation fails.
- **Input and output connections:**
  - Input from `Add Search Metadata`
  - Output to `Format Property Fields`
- **Version-specific requirements:**
  - Uses type version `2`.
- **Edge cases or potential failure types:**
  - Invalid JSON string in `search_results`.
  - `search_results` missing or null.
  - Non-array payload causes explicit node failure.
  - Large result sets may increase execution time and memory usage.
- **Sub-workflow reference:**
  - None.

#### `Format Property Fields`
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes raw property objects into consistent fields for later filtering and storage.
- **Configuration choices:**
  - Produces fields such as:
    - `property_address`
    - `property_price`
    - `property_type`
    - `bedrooms`
    - `bathrooms`
    - `bathrooms_full`
    - `bathrooms_half`
    - `square_footage`
    - `lot_area`
    - `year_built`
    - `listing_status`
    - `days_on_market`
    - `mls_id`
    - `property_url`
    - `photo_url`
    - `location`
    - `description`
    - `hoa_fee`
    - `price_per_sqft`
    - `sold_date`
    - `time_zone`
    - `country`
    - `badges`
    - `latitude`
    - `longitude`
- **Key expressions or variables used:**
  - `={{ $json.address || $json.title || 'N/A' }}`
  - `={{ $json.price || 0 }}`
  - `={{ $json.property_type || 'N/A' }}`
  - `={{ ($json.description || '').substring(0, 200) + '...' }}`
  - `={{ ($json.badges || []).join(', ') || 'None' }}`
  - `={{ $json.lat_long?.latitude || 0 }}`
  - `={{ $json.lat_long?.longitude || 0 }}`
- **Input and output connections:**
  - Input from `Split Properties Into Items`
  - Output to `Filter Valid Properties`
- **Version-specific requirements:**
  - Uses type version `3.2`.
- **Edge cases or potential failure types:**
  - Several configured field names again start with `=` or appear malformed.
  - There is an empty object in the field list, which may indicate an accidental blank field entry.
  - `price_per_sqft` uses `price_per_sqrf`, which may be a typo and produce zeros/undefined values.
  - This node creates `property_price`, but the filter node checks `price`, not `property_price`.
  - Original fields may or may not still be present depending on Set-node behavior and n8n settings; this affects downstream consistency.
- **Sub-workflow reference:**
  - None.

#### `Filter Valid Properties`
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters items to retain only listings with a non-placeholder address and a non-zero price.
- **Configuration choices:**
  - Conditions are combined with `AND`.
  - Condition 1: `property_address != 'N/A'`
  - Condition 2: numeric `price != 0`
- **Key expressions or variables used:**
  - `={{ $json.property_address }}`
  - `={{ $json.price }}`
- **Input and output connections:**
  - Input from `Format Property Fields`
  - True output goes to:
    - `Save Listings to Google Sheets`
    - `Send Slack Summary`
  - False branch is unused.
- **Version-specific requirements:**
  - Uses type version `2`.
- **Edge cases or potential failure types:**
  - Likely logic inconsistency: upstream formatting creates `property_price`, but this node validates `price`.
  - Because of that mismatch, valid properties could be rejected if `price` no longer exists.
  - Strict type validation may fail if values are strings instead of numbers.
- **Sub-workflow reference:**
  - None.

---

## 2.4 Save & Notify

### Overview
This block persists filtered properties to Google Sheets and sends a Slack message after or during processing. It is the output layer of the workflow.

### Nodes Involved
- `Save Listings to Google Sheets`
- `Send Slack Summary`

### Node Details

#### `Save Listings to Google Sheets`
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends property data as rows into a specific spreadsheet tab.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet ID: `1FYbt_m8nUdlkSmCzwZeBjgp4js6sdIKpiyyTIPBsigQ`
  - Sheet: `gid=0` / cached as `Sheet1`
  - Mapping mode: define fields manually
  - Type conversion disabled
  - Uses a fixed schema including columns such as Address, Price, Bedrooms, Bathrooms, Photo_URL, Property_URL, Coordinates, etc.
- **Key expressions or variables used:**
  - `Price = {{ $('Filter Valid Properties').item.json.price }}`
  - `Badges = {{ $json.badges }}`
  - `MLS_ID = {{ $json.mls_id }}`
  - `Status = {{ $json.listing_status }}`
  - `Address = {{ $json.address }}`
  - `HOA_Fee = {{ $json.hoa_fee }}`
  - `Has_HOA = {{ $json.hoa }}`
  - `Bedrooms = {{ $json.bedrooms }}`
  - `Location = {{ $json.property_address }}`
  - `Bathrooms = {{ $json.bathrooms }}`
  - `Lot_Acres = {{ $json.lot_area }}`
  - `Photo_URL = {{ $json.photo }}`
  - `Scraped_At = {{ new Date().toISOString() }}`
  - `Year_Built = {{ $json.year_built }}`
  - `Coordinates = {{ $json.lat_long.latitude }}, {{ $json.lat_long.longitude }}`
  - `Description = {{ $json.description }}`
  - `Square_Feet = {{ $json.square_footage }}`
  - `Property_URL = {{ $json.property_url }}`
  - `Property_Type = {{ $('Filter Valid Properties').item.json.property_type }}`
  - `Price_Per_SqFt = {{ $json.price_per_sqrf }}`
- **Input and output connections:**
  - Input from `Filter Valid Properties`
  - Output to `Send Slack Summary`
- **Version-specific requirements:**
  - Uses type version `4.4`.
  - Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - There are multiple field mismatches:
    - `Address` maps from `$json.address`, while normalized field is `property_address`.
    - `Photo_URL` maps from `$json.photo`, while normalized field appears to be `photo_url`.
    - `Coordinates` expects `$json.lat_long.*`, while normalized fields are `latitude` and `longitude`.
    - `Price_Per_SqFt` again references `price_per_sqrf`.
  - If the Google Sheet columns do not match exactly, append may fail or create incomplete rows.
  - Spreadsheet permissions, expired OAuth token, or wrong document ID can break execution.
  - Concurrent high-volume appends may hit API quotas.
- **Sub-workflow reference:**
  - None.

#### `Send Slack Summary`
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a message to a Slack channel indicating scrape completion.
- **Configuration choices:**
  - Sends a fixed text message:
    - `🏠 Redfin Scrape Complete! | Sheet: https://docs.google.com/spreadsheets/d/1FYbt_m8nUdlkSmCzwZeBjgp4js6sdIKpiyyTIPBsigQ`
  - Target is a selected Slack channel.
  - `executeOnce = true`, so the message should be sent once per execution rather than once per item.
- **Key expressions or variables used:**
  - No dynamic count expression is included, despite the sticky note mentioning listing count.
- **Input and output connections:**
  - Inputs from:
    - `Filter Valid Properties`
    - `Save Listings to Google Sheets`
  - No downstream output.
- **Version-specific requirements:**
  - Uses type version `2.1`.
  - Requires Slack API credentials.
- **Edge cases or potential failure types:**
  - Slack auth/permission issues.
  - Channel not found or bot not invited.
  - Because the node has two incoming branches, runtime behavior depends on item flow and execution semantics; `executeOnce` reduces duplicate posting but does not make this a true merge node.
  - Message is static, so it does not confirm actual row count or success count.
- **Sub-workflow reference:**
  - None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ` Schedule Trigger` | Schedule Trigger | Launches the workflow every 6 hours |  | `Set Search Parameters` | ## 1. Trigger & Configuration<br>Fires on a schedule and sets the Redfin search URL to scrape. |
| `Set Search Parameters` | Set | Defines the Redfin search URL | ` Schedule Trigger` | `ScrapeOps: Fetch Redfin Page (Proxy)` | ## 1. Trigger & Configuration<br>Fires on a schedule and sets the Redfin search URL to scrape. |
| `ScrapeOps: Fetch Redfin Page (Proxy)` | ScrapeOps | Fetches the Redfin page with JS rendering and proxy options | `Set Search Parameters` | `ScrapeOps: Parse Redfin Listings` | ## 2. Fetch & Parse Listings<br>Load the Redfin page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with JS rendering, then extract structured listing data using the [ScrapeOps Parser API](https://scrapeops.io/docs/n8n/parser-api/). |
| `ScrapeOps: Parse Redfin Listings` | ScrapeOps | Parses Redfin HTML into structured listing JSON | `ScrapeOps: Fetch Redfin Page (Proxy)` | `Add Search Metadata` | ## 2. Fetch & Parse Listings<br>Load the Redfin page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with JS rendering, then extract structured listing data using the [ScrapeOps Parser API](https://scrapeops.io/docs/n8n/parser-api/). |
| `Add Search Metadata` | Set | Adds scrape timestamp and search-level metadata | `ScrapeOps: Parse Redfin Listings` | `Split Properties Into Items` | ## 3. Transform & Filter<br>Lift search summary fields, split results into individual items, normalize property fields, and drop listings missing an address or valid price. |
| `Split Properties Into Items` | Code | Converts listing array into one item per property | `Add Search Metadata` | `Format Property Fields` | ## 3. Transform & Filter<br>Lift search summary fields, split results into individual items, normalize property fields, and drop listings missing an address or valid price. |
| `Format Property Fields` | Set | Normalizes property fields for filtering and storage | `Split Properties Into Items` | `Filter Valid Properties` | ## 3. Transform & Filter<br>Lift search summary fields, split results into individual items, normalize property fields, and drop listings missing an address or valid price. |
| `Filter Valid Properties` | If | Keeps only listings with address and non-zero price | `Format Property Fields` | `Save Listings to Google Sheets`, `Send Slack Summary` | ## 3. Transform & Filter<br>Lift search summary fields, split results into individual items, normalize property fields, and drop listings missing an address or valid price. |
| `Save Listings to Google Sheets` | Google Sheets | Appends valid listings into the spreadsheet | `Filter Valid Properties` | `Send Slack Summary` | ## 4. Save & Notify<br>Append valid listings to Google Sheets and optionally post a Slack summary with listing count and a link to the sheet. |
| `Send Slack Summary` | Slack | Sends a completion message to Slack | `Filter Valid Properties`, `Save Listings to Google Sheets` |  | ## 4. Save & Notify<br>Append valid listings to Google Sheets and optionally post a Slack summary with listing count and a link to the sheet. |
| `Sticky Note` | Sticky Note | Workspace documentation |  |  | # 🏡 Redfin Property Scraper → Google Sheets + Slack<br><br>This workflow automatically scrapes Redfin property listings on a schedule. It fetches search results via **ScrapeOps Proxy API**, parses them into clean structured data using the **ScrapeOps Redfin Parser API**, filters valid listings, saves them to Google Sheets, and optionally sends a Slack summary.<br><br>### How it works<br>1. ⏰ **Schedule Trigger** fires every 6 hours automatically.<br>2. ⚙️ **Set Search Parameters** defines the Redfin search URL to scrape.<br>3. 🌐 **ScrapeOps: Fetch Redfin Page** loads the listing page with JS rendering and residential proxy via [ScrapeOps Proxy API](https://scrapeops.io/docs/n8n/proxy-api/).<br>4. 🔍 **ScrapeOps: Parse Redfin Listings** extracts clean structured JSON using the [ScrapeOps Parser API](https://scrapeops.io/docs/n8n/parser-api/).<br>5. 🗂️ **Add Search Metadata** lifts summary fields like total listings, region, and price range.<br>6. 📦 **Split Properties Into Items** turns the results array into one item per property.<br>7. 🗺️ **Format Property Fields** normalizes address, price, beds, baths, sqft, and more.<br>8. ✅ **Filter Valid Properties** drops items missing an address or with price = 0.<br>9. 💾 **Save Listings to Google Sheets** appends valid rows to your sheet.<br>10. 📣 **Send Slack Summary** posts an optional summary with listing count and sheet link.<br><br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials to both ScrapeOps nodes. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate the [Google Sheet template](https://docs.google.com/spreadsheets/d/1FYbt_m8nUdlkSmCzwZeBjgp4js6sdIKpiyyTIPBsigQ/edit?gid=0#gid=0) and paste your Sheet ID into **Save Listings to Google Sheets**.<br>- Set your target city URL in **Set Search Parameters**.<br>- Optional: configure the Slack node with your channel and credentials.<br><br>### Customization<br>- Change `redfin_url` to any Redfin city, ZIP, or filtered search page.<br>- Adjust `wait` and `scroll` settings in the Proxy node if results are empty.<br>- Change the schedule interval to daily or hourly as needed. |
| `Sticky Note1` | Sticky Note | Workspace documentation |  |  | ## 1. Trigger & Configuration<br>Fires on a schedule and sets the Redfin search URL to scrape. |
| `Sticky Note2` | Sticky Note | Workspace documentation |  |  | ## 2. Fetch & Parse Listings<br>Load the Redfin page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with JS rendering, then extract structured listing data using the [ScrapeOps Parser API](https://scrapeops.io/docs/n8n/parser-api/). |
| `Sticky Note3` | Sticky Note | Workspace documentation |  |  | ## 3. Transform & Filter<br>Lift search summary fields, split results into individual items, normalize property fields, and drop listings missing an address or valid price. |
| `Sticky Note4` | Sticky Note | Workspace documentation |  |  | ## 4. Save & Notify<br>Append valid listings to Google Sheets and optionally post a Slack summary with listing count and a link to the sheet. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n. It mirrors the current workflow, while also noting places where the original configuration contains inconsistencies that you may want to correct during implementation.

## Prerequisites
1. Install or enable n8n.
2. Install the ScrapeOps n8n node package if it is not already available in your instance.
3. Prepare credentials:
   - **ScrapeOps API credentials**
   - **Google Sheets OAuth2 credentials**
   - **Slack API credentials**
4. Duplicate or create a Google Sheet with columns matching the intended schema.

## Step-by-step build

1. **Create a Schedule Trigger node**
   - Type: `Schedule Trigger`
   - Set interval to every `6 hours`.
   - This is the workflow entry point.

2. **Create a Set node named `Set Search Parameters`**
   - Type: `Set`
   - Add one field:
     - `redfin_url` as string
     - Example value: `https://www.redfin.com/city/21853/MD/California`
   - Connect `Schedule Trigger` → `Set Search Parameters`.

3. **Create a ScrapeOps node named `ScrapeOps: Fetch Redfin Page (Proxy)`**
   - Type: `ScrapeOps`
   - Use ScrapeOps credentials.
   - Configure:
     - URL: `{{$json.redfin_url}}`
     - Wait: `5000`
     - Scroll: `2000`
     - Country: `us`
     - Render JS: enabled
     - Device type: `desktop`
     - Residential proxy: enabled
   - Connect `Set Search Parameters` → `ScrapeOps: Fetch Redfin Page (Proxy)`.

4. **Create another ScrapeOps node named `ScrapeOps: Parse Redfin Listings`**
   - Type: `ScrapeOps`
   - Use the same ScrapeOps credentials.
   - Configure:
     - API Type: `Parser API`
     - Parser Domain: `redfin`
     - Parser URL: `{{ $('Set Search Parameters').item.json.redfin_url }}`
     - Parser HTML: `{{ $json }}`
   - Connect `ScrapeOps: Fetch Redfin Page (Proxy)` → `ScrapeOps: Parse Redfin Listings`.

5. **Create a Set node named `Add Search Metadata`**
   - Type: `Set`
   - Add fields for search metadata. To reproduce the workflow cleanly, use these recommended field names:
     - `timestamp` → `{{ new Date().toISOString() }}`
     - `search_title` → `{{ $json.data.search_information.search_title || 'N/A' }}`
     - `search_type` → `{{ $json.data.search_information.search_type || 'N/A' }}`
     - `total_properties` → `{{ $json.data.search_information.total_count || 0 }}`
     - `region_info` → `{{ $json.data.search_information.region ? $json.data.search_information.region.name + ', ' + $json.data.search_information.region.state : 'N/A' }}`
     - `price_range` → `{{ ($json.data.search_information.min_price || 0) + ' - ' + ($json.data.search_information.max_price || 0) }}`
     - `average_price` → `{{ $json.data.search_information.average_price || 0 }}`
     - `url_scraped` → `{{ $json.url || 'N/A' }}`
     - `search_results` → `{{ $json.data.search_results }}`
   - Connect `ScrapeOps: Parse Redfin Listings` → `Add Search Metadata`.

6. **Create a Code node named `Split Properties Into Items`**
   - Type: `Code`
   - JavaScript:
     ```javascript
     const searchResults = $input.first().json.search_results;

     const propertiesArray = typeof searchResults === 'string'
       ? JSON.parse(searchResults)
       : searchResults;

     if (!Array.isArray(propertiesArray)) {
       throw new Error('search_results is not an array');
     }

     return propertiesArray.map((property, index) => ({
       json: property,
       pairedItem: { item: 0 }
     }));
     ```
   - Connect `Add Search Metadata` → `Split Properties Into Items`.

7. **Create a Set node named `Format Property Fields`**
   - Type: `Set`
   - Add normalized fields:
     - `property_address` → `{{ $json.address || $json.title || 'N/A' }}`
     - `property_price` → `{{ $json.price || 0 }}`
     - `property_type` → `{{ $json.property_type || 'N/A' }}`
     - `bedrooms` → `{{ $json.bedrooms || 0 }}`
     - `bathrooms` → `{{ $json.bathrooms || 0 }}`
     - `bathrooms_full` → `{{ $json.bathrooms_full || 0 }}`
     - `bathrooms_half` → `{{ $json.bathrooms_half || 0 }}`
     - `square_footage` → `{{ $json.area || 0 }}`
     - `lot_area` → `{{ $json.lot_area || 0 }}`
     - `year_built` → `{{ $json.year_built || 'N/A' }}`
     - `listing_status` → `{{ $json.listing_status || $json.status || 'N/A' }}`
     - `days_on_market` → `{{ $json.days_on_market || $json.dom || 'N/A' }}`
     - `mls_id` → `{{ $json.mls_id || 'N/A' }}`
     - `property_url` → `{{ $json.url || 'N/A' }}`
     - `photo_url` → `{{ $json.photo || 'N/A' }}`
     - `location` → `{{ $json.location || 'N/A' }}`
     - `description` → `{{ (($json.description || '').substring(0, 200)) + '...' }}`
     - `hoa_fee` → `{{ $json.hoa || 0 }}`
     - `price_per_sqft` → `{{ $json.price_per_sqft || 0 }}`
     - `sold_date` → `{{ $json.sold_date || 'N/A' }}`
     - `time_zone` → `{{ $json.time_zone || 'N/A' }}`
     - `country` → `{{ $json.country || 'N/A' }}`
     - `badges` → `{{ ($json.badges || []).join(', ') || 'None' }}`
     - `latitude` → `{{ $json.lat_long?.latitude || 0 }}`
     - `longitude` → `{{ $json.lat_long?.longitude || 0 }}`
   - Prefer clean field names without `=` prefixes.
   - Connect `Split Properties Into Items` → `Format Property Fields`.

8. **Create an If node named `Filter Valid Properties`**
   - Type: `If`
   - Set `AND` logic.
   - Conditions:
     - `{{$json.property_address}}` is not equal to `N/A`
     - `{{$json.property_price}}` is not equal to `0`
   - Important: use `property_price`, not `price`, if you want this block to align with the formatting node.
   - Connect `Format Property Fields` → `Filter Valid Properties`.

9. **Create a Google Sheets node named `Save Listings to Google Sheets`**
   - Type: `Google Sheets`
   - Credentials: Google Sheets OAuth2.
   - Operation: `Append`
   - Select your spreadsheet.
   - Select target sheet/tab.
   - Enable manual field mapping.
   - Map columns such as:
     - `Address` → `{{ $json.property_address }}`
     - `Price` → `{{ $json.property_price }}`
     - `Property_Type` → `{{ $json.property_type }}`
     - `Bedrooms` → `{{ $json.bedrooms }}`
     - `Bathrooms` → `{{ $json.bathrooms }}`
     - `Square_Feet` → `{{ $json.square_footage }}`
     - `Lot_Acres` → `{{ $json.lot_area }}`
     - `Year_Built` → `{{ $json.year_built }}`
     - `Status` → `{{ $json.listing_status }}`
     - `MLS_ID` → `{{ $json.mls_id }}`
     - `Price_Per_SqFt` → `{{ $json.price_per_sqft }}`
     - `HOA_Fee` → `{{ $json.hoa_fee }}`
     - `Location` → `{{ $json.location }}`
     - `Description` → `{{ $json.description }}`
     - `Photo_URL` → `{{ $json.photo_url }}`
     - `Property_URL` → `{{ $json.property_url }}`
     - `Coordinates` → `{{ $json.latitude }}, {{ $json.longitude }}`
     - `Scraped_At` → `{{ new Date().toISOString() }}`
     - `Badges` → `{{ $json.badges }}`
   - Connect the **true** output of `Filter Valid Properties` → `Save Listings to Google Sheets`.

10. **Prepare the spreadsheet columns**
    - Ensure your Google Sheet contains the same column names used in the node mapping.
    - The original workflow references:
      - Address
      - Price
      - Property_Type
      - Bedrooms
      - Bathrooms
      - Square_Feet
      - Lot_Acres
      - Year_Built
      - Status
      - MLS_ID
      - Price_Per_SqFt
      - HOA_Fee
      - Has_HOA
      - Location
      - Description
      - Photo_URL
      - Property_URL
      - Coordinates
      - Scraped_At
      - Badges

11. **Create a Slack node named `Send Slack Summary`**
    - Type: `Slack`
    - Credentials: Slack API.
    - Action: send message to channel.
    - Select the target channel.
    - Message example:
      - `🏠 Redfin Scrape Complete! | Sheet: https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID`
    - Enable `Execute Once`.
   - To match the original workflow, connect:
     - `Filter Valid Properties` → `Send Slack Summary`
     - `Save Listings to Google Sheets` → `Send Slack Summary`
   - For a cleaner design, many builders would instead use a Merge or summary node before Slack, but that is not how the provided workflow is wired.

12. **Optionally add sticky notes**
    - Add documentation notes for each major block:
      - Trigger & Configuration
      - Fetch & Parse Listings
      - Transform & Filter
      - Save & Notify

13. **Activate and test**
    - Run manually first.
    - Check:
      - ScrapeOps returns rendered HTML
      - Parser returns `data.search_results`
      - Code node splits items correctly
      - Google Sheets receives rows
      - Slack receives one message

## Credential setup details

### ScrapeOps
- Create a ScrapeOps account: https://scrapeops.io/app/register/n8n
- Add ScrapeOps credentials in n8n.
- Assign the same credential to both ScrapeOps nodes.

### Google Sheets
- Create OAuth2 credentials in n8n for Google Sheets.
- Share the target spreadsheet with the authenticated Google account if needed.
- Use the correct spreadsheet ID and sheet tab.

### Slack
- Create/connect a Slack app or bot credential in n8n.
- Ensure the bot can post to the target channel.
- Invite the bot to the channel if necessary.

## Important implementation cautions
To reproduce the workflow exactly, you could keep the original field mismatches. To reproduce it reliably, correct the following:
1. Use `property_price` consistently instead of switching between `price` and `property_price`.
2. Use `property_address` consistently instead of switching between `address` and `property_address`.
3. Use `photo_url` consistently instead of `photo`.
4. Use `latitude`/`longitude` consistently instead of later referencing `lat_long`.
5. Replace `price_per_sqrf` with `price_per_sqft` if that is the intended source field.
6. Remove malformed Set field names that start with `=` unless intentionally required.
7. Remove the blank field entry in `Format Property Fields`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ScrapeOps registration page for API access | https://scrapeops.io/app/register/n8n |
| ScrapeOps n8n overview documentation | https://scrapeops.io/docs/n8n/overview/ |
| ScrapeOps Proxy API documentation | https://scrapeops.io/docs/n8n/proxy-api/ |
| ScrapeOps Parser API documentation | https://scrapeops.io/docs/n8n/parser-api/ |
| Google Sheet template referenced in the workflow | https://docs.google.com/spreadsheets/d/1FYbt_m8nUdlkSmCzwZeBjgp4js6sdIKpiyyTIPBsigQ/edit?gid=0#gid=0 |
| Workflow note: change `redfin_url` to any Redfin city, ZIP, or filtered search page | Configuration guidance from workspace notes |
| Workflow note: adjust `wait` and `scroll` settings if results are empty | Scrape/render tuning guidance from workspace notes |
| Workflow note: change the schedule interval to daily or hourly as needed | Scheduling customization guidance from workspace notes |