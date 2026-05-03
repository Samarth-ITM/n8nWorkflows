Scrape Crunchbase recent funding rounds

https://n8nworkflows.xyz/workflows/scrape-crunchbase-recent-funding-rounds-2076


# Scrape Crunchbase recent funding rounds

### 1. Workflow Overview

This n8n workflow automates the retrieval of recent funding rounds data from Crunchbase via the Piloterr API and records enriched company information into a Google Sheets spreadsheet. It targets users interested in tracking startup fundraising activities, enriching lead databases with company metadata, and automating lead engagement or alerting workflows.

The logical flow is organized into these blocks:

- **1.1 Schedule Trigger & Data Retrieval:**  
  Triggered daily, this block queries the Piloterr API to fetch recent funding rounds filtered by investment types (Seed, Series A, Series B).

- **1.2 Data Splitting & Preparation:**  
  Raw API responses are split into individual funding round records and minimally prepared with key fields extracted.

- **1.3 Company Data Enrichment:**  
  This block enriches each company by querying additional company details via Piloterr API and extracts LinkedIn URLs from social network data.

- **1.4 Final Data Formatting & Google Sheets Update:**  
  The enriched data is formatted into the required schema and appended or updated into Google Sheets for downstream use.

- **1.5 Workflow Metadata & Documentation:**  
  Sticky notes provide contextual guidance and usage instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Data Retrieval

**Overview:**  
This block initiates the workflow daily at 8 AM and performs three separate API calls to Piloterr to fetch recent funding rounds for Seed, Series A, and Series B investment types, each limited to events announced in the last day.

**Nodes Involved:**  
- Schedule Trigger - Run Workflow Every Day  
- Piloterr - Get Recent Fundraise - Seed  
- Piloterr - Get Recent Fundraise - Serie A  
- Piloterr - Get Recent Fundraise - Serie B

**Node Details:**

- **Schedule Trigger - Run Workflow Every Day**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 08:00 hours.  
  - Configuration: Interval set to trigger at hour 8 every day.  
  - Inputs: None (trigger node).  
  - Outputs: Three parallel output connections to the three Piloterr API request nodes.  
  - Edge Cases: Workflow will not run if n8n is down or paused at trigger time.

- **Piloterr - Get Recent Fundraise - Seed / Serie A / Serie B**  
  - Type: HTTP Request  
  - Role: Fetch recent funding rounds from Piloterr API filtered by investment type (seed, series_a, series_b) and days_since_announcement = 1.  
  - Configuration:  
    - URL: `https://piloterr.com/api/v2/crunchbase/funding_rounds`  
    - Query parameters: `days_since_announcement=1`, `investment_type` set per node.  
    - Authentication: HTTP Header Auth with Piloterr API key credential.  
  - Inputs: Trigger node output.  
  - Outputs: Raw JSON response containing funding rounds data.  
  - Edge Cases: Possible API auth failure, rate limiting, or network timeouts. No retry logic configured.  
  - Notes: Users can remove unused investment type nodes if not needed.

#### 1.2 Data Splitting & Preparation

**Overview:**  
This block splits the funding rounds array from each API response into individual items for processing and extracts key fields into a normalized format.

**Nodes Involved:**  
- Split results  
- Prepare data

**Node Details:**

- **Split results**  
  - Type: Item Lists  
  - Role: Splits the `results` array from API response into separate workflow items.  
  - Configuration: Field to split out: `results`.  
  - Inputs: Each Piloterr API response node output.  
  - Outputs: One item per funding round.  
  - Edge Cases: If `results` field is empty or missing, no items are created.

- **Prepare data**  
  - Type: Set  
  - Role: Extracts and sets key funding round fields for downstream enrichment.  
  - Configuration: Keeps only set fields. Sets:  
    - `type` = investment_type from funding round  
    - `money_raised` = money_raised.value_usd  
    - `announced_on` = announced_on date  
    - `company_name` = funded_organization_identifier.value (company name)  
    - `link` = funded_organization_identifier.permalink (Crunchbase permalink)  
    - `event_link` = identifier.permalink (funding round permalink)  
  - Inputs: Split results node output.  
  - Outputs: Normalized funding round data items.  
  - Edge Cases: Missing fields in raw data may cause empty or null values.

#### 1.3 Company Data Enrichment

**Overview:**  
This block enriches each funding round’s company data by fetching detailed company information from Piloterr and extracts the LinkedIn URL from social networks.

**Nodes Involved:**  
- Piloterr - Enrich company  
- Get Linkedin URL from object

**Node Details:**

- **Piloterr - Enrich company**  
  - Type: HTTP Request  
  - Role: Retrieves detailed company info from Piloterr API using company Crunchbase permalink.  
  - Configuration:  
    - URL: `https://piloterr.com/api/v2/crunchbase/company/info`  
    - Query parameter: `query` set dynamically to `https://www.crunchbase.com/organization/{{ $json["link"] }}`  
    - Batching enabled with batch size 3 to limit concurrent requests.  
    - Authentication: Piloterr HTTP Header Auth.  
    - Continue on fail: true (to avoid workflow stop on enrichment failure).  
  - Inputs: Prepared funding round data.  
  - Outputs: Enriched company data including website, traffic, funding totals, social networks, employee count, location, etc.  
  - Edge Cases: API failures, empty enrichment data, partial data; continueOnFail avoids workflow break.

- **Get Linkedin URL from object**  
  - Type: Code (JavaScript)  
  - Role: Extracts LinkedIn URL from `social_networks` array inside enriched company data.  
  - Configuration:  
    - Runs once per item.  
    - Searches array for object where `name === 'linkedin'`.  
    - Sets `linkedin_url` property with found URL or null if missing.  
    - Logs error if LinkedIn URL not found (does not halt workflow).  
  - Inputs: Enriched company data.  
  - Outputs: Company data with added `linkedin_url` field.  
  - Edge Cases: Missing or empty social_networks array, no LinkedIn entry found.

#### 1.4 Final Data Formatting & Google Sheets Update

**Overview:**  
This block formats the enriched data to match Google Sheets columns and appends or updates rows in the designated spreadsheet.

**Nodes Involved:**  
- Prepare data before importing to Gsheets  
- Merge  
- Google Sheets

**Node Details:**

- **Prepare data before importing to Gsheets**  
  - Type: Set  
  - Role: Extracts final fields for spreadsheet insertion.  
  - Configuration: Keeps only set fields. Sets:  
    - `website_url` = domain extracted from company website URL (regex).  
    - `monthly_traffic_semrush` = latest monthly visits from SEMrush summary.  
    - `funding_total` = total funding value from funding rounds headline.  
    - `linkedin_url`, `employee_count`, `country`, `founded_date` from enriched data.  
  - Inputs: Output from LinkedIn URL extraction node.  
  - Outputs: Final structured item for spreadsheet.  
  - Edge Cases: Missing or malformed website URLs may cause regex failure; missing SEMrush or location data.

- **Merge**  
  - Type: Merge  
  - Role: Combines data from the three investment type branches into a single stream for the Google Sheets node.  
  - Configuration:  
    - Mode: Combine  
    - Combination Mode: Merge by position (aligns items by index).  
  - Inputs: Two inputs:  
    - Input 0 from `Prepare data before importing to Gsheets` (Seed, Series A, Series B branch combined)  
    - Input 1 from `Prepare data` node output after enrichment (used for merging enriched and prepared data).  
  - Outputs: Merged data stream.  
  - Edge Cases: Misaligned data length between merge inputs could cause data mismatches.

- **Google Sheets**  
  - Type: Google Sheets  
  - Role: Append or update rows in Google Sheets with the final enriched funding round data.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Document: URL for target Google Sheet provided.  
    - Sheet: Sheet1 (gid=0)  
    - Columns mapped: company_name, website_url, type, money_raised, linkedin_url, announced_on, funding_total, link, monthly_traffic_semrush, event_link, employee_count, country, founded_date  
    - Matching columns for update: event_link (ensures no duplicates)  
    - Credentials: Google Sheets OAuth2 account configured.  
  - Inputs: Merged enriched and prepared data.  
  - Outputs: Updated Google Sheet rows.  
  - Edge Cases: API quota limits, Google Sheets API errors, invalid spreadsheet URLs, permission errors.

#### 1.5 Workflow Metadata & Documentation

**Overview:**  
Sticky notes provide contextual information and usage instructions for workflow users.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - Contains a summary and link to full user guide:  
    "This workflow will scrape recent fundraising events from Crunchbase, and add them in Google Sheets.  
    Full guide here: https://lempire.notion.site/Get-recent-fundraising-in-Google-Sheets-dafbbda2635544b4925c4fb04abac8f5?pvs=74"

- **Sticky Note1**  
  - Contains usage notes:  
    "Create an account at piloterr.com to get your API key  
    Feel free to delete the node that are not useful to you. For instance 'Serie B' and 'Seed' if you want only to scrape Serie A events"

---

### 3. Summary Table

| Node Name                                  | Node Type         | Functional Role                               | Input Node(s)                                | Output Node(s)                         | Sticky Note                                                                                     |
|--------------------------------------------|-------------------|----------------------------------------------|----------------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger - Run Workflow Every Day  | Schedule Trigger  | Triggers workflow daily at 8 AM               | None                                         | Piloterr - Get Recent Fundraise - Seed, Serie A, Serie B |                                                                                                |
| Piloterr - Get Recent Fundraise - Seed     | HTTP Request      | Fetch recent Seed funding rounds from Piloterr| Schedule Trigger                             | Split results                        | "Create an account at piloterr.com to get your API key. Feel free to delete nodes not useful."  |
| Piloterr - Get Recent Fundraise - Serie A  | HTTP Request      | Fetch recent Series A funding rounds from Piloterr | Schedule Trigger                             | Split results                        | "Create an account at piloterr.com to get your API key. Feel free to delete nodes not useful."  |
| Piloterr - Get Recent Fundraise - Serie B  | HTTP Request      | Fetch recent Series B funding rounds from Piloterr | Schedule Trigger                             | Split results                        | "Create an account at piloterr.com to get your API key. Feel free to delete nodes not useful."  |
| Split results                              | Item Lists        | Splits API response array into individual items | Piloterr API nodes (Seed, Serie A, Serie B)  | Prepare data                        |                                                                                                |
| Prepare data                              | Set               | Extracts key funding round fields             | Split results                                | Piloterr - Enrich company            |                                                                                                |
| Piloterr - Enrich company                 | HTTP Request      | Enriches company info from Piloterr           | Prepare data                                 | Get Linkedin URL from object         |                                                                                                |
| Get Linkedin URL from object              | Code (JavaScript) | Extracts LinkedIn URL from company social networks | Piloterr - Enrich company                    | Prepare data before importing to Gsheets |                                                                                                |
| Prepare data before importing to Gsheets | Set               | Formats enriched data for Google Sheets        | Get Linkedin URL from object                  | Merge                               |                                                                                                |
| Merge                                    | Merge             | Combines enriched data streams                  | Prepare data before importing to Gsheets, Prepare data | Google Sheets                      |                                                                                                |
| Google Sheets                            | Google Sheets     | Appends or updates rows in Google Sheets        | Merge                                        | None                                |                                                                                                |
| Sticky Note                             | Sticky Note       | Provides workflow description and guide link   | None                                         | None                                | "This workflow will scrape recent fundraising events from Crunchbase, and add them in Google Sheets. Full guide here: https://lempire.notion.site/Get-recent-fundraising-in-Google-Sheets-dafbbda2635544b4925c4fb04abac8f5?pvs=74" |
| Sticky Note1                            | Sticky Note       | Provides usage instructions                      | None                                         | None                                | "Create an account at piloterr.com to get your API key. Feel free to delete nodes not useful. For instance 'Serie B' and 'Seed' if you want only to scrape Serie A events" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger once daily at 08:00 hours.

2. **Create Three HTTP Request Nodes to Piloterr API:**  
   - Name them: "Piloterr - Get Recent Fundraise - Seed", "Piloterr - Get Recent Fundraise - Serie A", "Piloterr - Get Recent Fundraise - Serie B"  
   - URL: `https://piloterr.com/api/v2/crunchbase/funding_rounds`  
   - Query Parameters:  
     - `days_since_announcement` = 1  
     - `investment_type` = `seed`, `series_a`, or `series_b` respectively  
   - Authentication: Configure HTTP Header Auth credentials with your Piloterr API key  
   - Connect output of Schedule Trigger to all three HTTP Request nodes.

3. **Create Item Lists Node 'Split results':**  
   - Used to split the `results` array from each Piloterr API node into individual items  
   - Parameter: Field to Split Out = `results`  
   - Connect each Piloterr HTTP Request node output to this node (can be multiple connections or set up separate Split nodes per branch, then merged).

4. **Create Set Node 'Prepare data':**  
   - Extract and set these fields from each funding round item:  
     - `type` = `investment_type`  
     - `money_raised` = `money_raised.value_usd`  
     - `announced_on` = `announced_on`  
     - `company_name` = `funded_organization_identifier.value`  
     - `link` = `funded_organization_identifier.permalink`  
     - `event_link` = `identifier.permalink`  
   - Keep only these fields  
   - Connect output of Split results node to this node.

5. **Create HTTP Request Node 'Piloterr - Enrich company':**  
   - URL: `https://piloterr.com/api/v2/crunchbase/company/info`  
   - Query Parameter: `query` = `https://www.crunchbase.com/organization/{{ $json["link"] }}` (dynamic expression)  
   - Enable batching with batch size 3 to limit concurrency  
   - Authentication: Use Piloterr HTTP Header Auth credentials  
   - Set "Continue On Fail" to true  
   - Connect output of Prepare data node to this node.

6. **Create Code Node 'Get Linkedin URL from object':**  
   - JavaScript code to extract LinkedIn URL from `social_networks` array:  
     ```javascript
     let linkedinObject = $json.social_networks.find(e => e.name === 'linkedin');
     $input.item.json.linkedin_url = linkedinObject ? linkedinObject.url : null;
     if (!$input.item.json.linkedin_url) {
         console.error('No LinkedIn URL found!');
     }
     return $input.item;
     ```  
   - Connect output of Piloterr - Enrich company node to this node.

7. **Create Set Node 'Prepare data before importing to Gsheets':**  
   - Extract and set these fields:  
     - `website_url` = Extract domain from `$json.website` using regex `https?:\/\/(?:www\.)?([^\/]+)`  
     - `monthly_traffic_semrush` = `$json.semrush_summary.semrush_visits_latest_month`  
     - `funding_total` = `$json.funding_rounds_headline.funding_total.value`  
     - `linkedin_url`, `employee_count`, `country` (from `$json.location[2].name`), `founded_date` (from `$json.founded`)  
   - Keep only these fields  
   - Connect output of Get Linkedin URL from object node to this node.

8. **Create Merge Node 'Merge':**  
   - Mode: Combine  
   - Combination Mode: Merge by position (align items by index)  
   - Connect output of Prepare data before importing to Gsheets node to input 0 of Merge node.  
   - Connect output of Prepare data node (from step 4) to input 1 of Merge node.  
   - This merges enriched data with base prepared data for final insertion.

9. **Create Google Sheets Node:**  
   - Operation: appendOrUpdate  
   - Document ID: Use the URL of your target Google Sheet (e.g., `https://docs.google.com/spreadsheets/d/your-sheet-id/edit#gid=0`)  
   - Sheet Name: `Sheet1` or your target sheet  
   - Map columns exactly: company_name, website_url, type, money_raised, linkedin_url, announced_on, funding_total, link, monthly_traffic_semrush, event_link, employee_count, country, founded_date  
   - Set Matching Column: `event_link` to avoid duplicate rows  
   - Credentials: Set up Google Sheets OAuth2 credentials with access to your spreadsheet  
   - Connect output of Merge node to this node.

10. **Add Sticky Notes (Optional):**  
    - One with workflow description and link to guide:  
      "This workflow will scrape recent fundraising events from Crunchbase, and add them in Google Sheets. Full guide here: https://lempire.notion.site/Get-recent-fundraising-in-Google-Sheets-dafbbda2635544b4925c4fb04abac8f5?pvs=74"  
    - Another advising to create Piloterr account and remove unused investment types.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Full guide and explanation of this workflow usage can be found here: https://lempire.notion.site/Get-recent-fundraising-in-Google-Sheets-dafbbda2635544b4925c4fb04abac8f5?pvs=74 | Official user guide for this workflow                                                                             |
| Create a free account at piloterr.com to obtain your API key for Crunchbase data access                        | Piloterr API platform for Crunchbase data: https://piloterr.com                                                    |
| You can remove nodes for Series B and Seed if only Series A funding rounds are of interest                     | Workflow customization tip                                                                                          |
| Google Sheets API quota and permission restrictions may affect workflow execution                              | Ensure your Google Sheets OAuth2 credentials have edit access and quota limits are monitored                       |
| Batch size of 3 in company enrichment node balances API rate limits and processing speed                      | Adjust batch size in 'Piloterr - Enrich company' node if needed                                                    |
| LinkedIn URL extraction relies on presence of LinkedIn social network entry in Piloterr company enrichment data| Missing LinkedIn URLs will be logged as errors but do not fail workflow                                            |