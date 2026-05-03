Scrape & Track Dentist Leads from Google Maps with ScrapeOps, Sheets & Notifications

https://n8nworkflows.xyz/workflows/scrape---track-dentist-leads-from-google-maps-with-scrapeops--sheets---notifications-11456


# Scrape & Track Dentist Leads from Google Maps with ScrapeOps, Sheets & Notifications

### 1. Workflow Overview

This n8n workflow automates the process of finding and tracking dentist leads in any specified city by scraping Google Maps data using ScrapeOps, analyzing the scraped data, deduplicating against existing leads stored in a Google Sheet, saving new leads back to the sheet, and sending notifications of new leads via Gmail and Slack.

The workflow is logically divided into four main blocks:

- **1.1 Input & Configuration:** Captures city input from a form and sets up the Google Maps search URL with configuration parameters.
- **2. Deep Scraping & Data Extraction:** Performs the Google Maps search scrape, extracts business listings, then scrapes individual business pages to parse detailed information.
- **3. Deduplication:** Reads existing lead data from Google Sheets, compares with freshly scraped data to identify new leads.
- **4. Save & Alert:** Saves the new leads into Google Sheets and sends notifications via Gmail and Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Configuration

- **Overview:**  
  This block obtains the city name input from a user via a web form trigger and sets up the search parameters and URLs needed for the Google Maps scraping.

- **Nodes Involved:**  
  - Form - City Input  
  - Set Google Maps Configuration

- **Node Details:**

  - **Form - City Input**  
    - Type: Form Trigger  
    - Role: Captures user input for the city name via a webhook form.  
    - Configuration: Single required field "City Name". Webhook path is `form-city-input-webhook`.  
    - Input/Output: No input; outputs form submission JSON containing the city.  
    - Failure Modes: Missing or invalid city input will later trigger error handling.  
    - Notes: Entry point for the workflow.

  - **Set Google Maps Configuration**  
    - Type: Set Node  
    - Role: Prepares search parameters including the city, fixed keyword "dentist", base URL, and the final Google Maps search URL by combining keyword and city.  
    - Key Expressions: Builds `searchUrl` as `"https://www.google.com/maps/search/dentist+in+<city>"` with spaces replaced by `+`.  
    - Input: Receives city from previous node.  
    - Output: JSON with keys: city, keyword, baseUrl, searchUrl.  
    - Failure Modes: If city is empty or malformed, subsequent scraping may fail or return empty results.

---

#### 2. Deep Scraping & Data Extraction

- **Overview:**  
  This block scrapes the Google Maps search results page HTML using ScrapeOps, then parses and extracts a list of dentists with partial info. It then scrapes each dentist’s individual Google Maps page to extract detailed info such as phone, website, ratings, reviews, address, and LGBTQ+ friendly status.

- **Nodes Involved:**  
  - ScrapeOps - Google Maps Search  
  - Parse Business Details (Code)  
  - ScrapeOps - Business Details  
  - Parse Full Business Info (Code)

- **Node Details:**

  - **ScrapeOps - Google Maps Search**  
    - Type: ScrapeOps (Scraper)  
    - Role: Fetches the HTML of the Google Maps search results for dentists in the specified city.  
    - Configuration: Uses the dynamic `searchUrl` from config, waits 12 seconds for JS rendering, no residential proxy.  
    - Input: Receives search URL JSON.  
    - Output: HTML response containing search results page.  
    - Failure Modes: Network errors, rate limits, or incomplete rendering may cause missing data.

  - **Parse Business Details**  
    - Type: Code Node (JavaScript)  
    - Role: Parses the HTML search results page to extract individual business listings, capturing business names, addresses, map URLs, phone numbers, websites, ratings, reviews count, and categories.  
    - Configuration: Complex regex and string extraction logic to handle multiple HTML patterns and fallback strategies.  
    - Key Variables: Uses `htmlContent` from ScrapeOps response, extracts unique map URLs (Google Maps place IDs).  
    - Input: HTML from previous scrape node.  
    - Output: Array of JSON objects with partial business details for each dentist found.  
    - Failure Modes: Changes in Google Maps HTML structure or incomplete HTML content may cause parsing errors or empty results.  
    - Edge Case: Returns error JSON if city invalid or no businesses found.

  - **ScrapeOps - Business Details**  
    - Type: ScrapeOps (Scraper)  
    - Role: For each business found in the search results, scrapes the individual Google Maps detail page HTML.  
    - Configuration: Uses each business’s `mapUrl` to fetch detail page, waits 12 seconds, no residential proxy.  
    - Input: List of `mapUrl` from previous node.  
    - Output: HTML response per business detail page.  
    - Failure Modes: Same as above; network or rendering issues can cause incomplete data.

  - **Parse Full Business Info**  
    - Type: Code Node (JavaScript)  
    - Role: Parses detailed business HTML pages to extract enriched info, including phone number, website, address, ratings, reviews (top 3), and LGBTQ+ friendly status. It merges this data carefully with the partial data from the search results.  
    - Key Logic: Matches each detail page to the corresponding partial data via map URLs or index fallback. Uses multiple regex patterns and fallback methods to robustly extract fields. Cleans and normalizes scraped text.  
    - Input: HTML detail pages + partial business data.  
    - Output: Fully detailed JSON objects per business.  
    - Failure Modes: Parsing failures or mismatches result in a "MATCHING_ERROR" entry with error details in JSON.  
    - Edge Cases: Handles missing or malformed data gracefully, limits large HTML sizes to prevent memory issues.

---

#### 3. Deduplication

- **Overview:**  
  This block reads previously saved leads from Google Sheets, compares the current scraped results against these to detect duplicates, and marks records as either "new" or "old" accordingly.

- **Nodes Involved:**  
  - Read Previous Entries from Sheet (Google Sheets)  
  - Compare With Previous Run (Code)  
  - Filter New Leads (If)

- **Node Details:**

  - **Read Previous Entries from Sheet**  
    - Type: Google Sheets  
    - Role: Reads all rows from the configured Google Sheet that stores previously saved leads.  
    - Configuration: Uses OAuth2 credentials, reads from the specified document and sheet by URL and gid.  
    - Input: Receives none (triggered by previous node).  
    - Output: Array of previous lead records as JSON.  
    - Failure Modes: Credential issues, sheet permission errors, or network failures.  
    - Continues on failure but outputs empty data if error occurs.

  - **Compare With Previous Run**  
    - Type: Code Node (JavaScript)  
    - Role: Compares current scraped leads against previous leads from sheet. Normalizes business names and phone numbers to detect duplicates. Assigns status "new" or "old" to each lead accordingly.  
    - Key Logic: Normalizes phone numbers (keeping + prefix, digits only) and business names (lowercase, trimmed). Uses exact and partial matching strategies, including name-only matches if phone missing. Skips duplicates in the current run.  
    - Input: Current full business info and previous entries from sheet.  
    - Output: List of all leads with added `status` field.  
    - Failure Modes: Errors reading previous data or empty current data return fallback "No data" or error JSON.

  - **Filter New Leads**  
    - Type: If Node  
    - Role: Filters the leads to pass only those marked as `status = "new"` for further processing (saving and alerting).  
    - Input: Leads from comparison node.  
    - Output: Two branches — "true" for new leads, "false" for old leads.  
    - Failure Modes: None significant; empty outputs if no new leads.

---

#### 4. Save & Alert

- **Overview:**  
  This block saves new leads to Google Sheets and sends notifications via email and Slack to alert stakeholders of new dentists found.

- **Nodes Involved:**  
  - Save to Google Sheets  
  - Send Gmail Alert for New Leads  
  - Send a message (Slack)

- **Node Details:**

  - **Save to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends new lead records to the configured Google Sheet to maintain updated lead lists.  
    - Configuration: Defines explicit columns mapping to lead JSON fields (e.g., businessName, phone, website, rating, address, city, status, etc.). Uses OAuth2 credentials.  
    - Input: New leads from filter node.  
    - Output: Confirmation of append operation.  
    - Failure Modes: Sheet permission errors, credential issues, or quota limits.  
    - Continues on failure to avoid workflow break.

  - **Send Gmail Alert for New Leads**  
    - Type: Gmail  
    - Role: Sends an email alert to a configured recipient notifying them of new dentist leads found in the specified city, including a link to the Google Sheet.  
    - Configuration: Uses OAuth2 Gmail credentials; subject includes dynamic city name; plain text email body includes sheet URL.  
    - Input: Triggered once after saving new leads.  
    - Output: Email sent confirmation.  
    - Failure Modes: Credential issues, email quota, or network problems.

  - **Send a message (Slack)**  
    - Type: Slack  
    - Role: Sends a Slack message to a specified user notifying them of new dentist leads with a link to the Google Sheet.  
    - Configuration: Uses Slack API credentials; sends message to user by ID; includes link to sheet; configured to execute once per run.  
    - Input: Triggered after Gmail alert.  
    - Output: Slack message confirmation.  
    - Failure Modes: API token issues, user ID invalid, or network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                           | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                          |
|-------------------------------|--------------------------------|------------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Form - City Input              | Form Trigger                   | Capture city input from user              | None                            | Set Google Maps Configuration  | ## 1. Input & Config Capture city input and set search terms.                                                        |
| Set Google Maps Configuration  | Set                           | Build search URL and parameters           | Form - City Input               | ScrapeOps - Google Maps Search  | ## 1. Input & Config Capture city input and set search terms.                                                        |
| ScrapeOps - Google Maps Search | ScrapeOps Scraper             | Scrape Google Maps search results         | Set Google Maps Configuration   | Parse Business Details          | ## 2. Deep Scraping Search Maps & extract full dentist details.                                                      |
| Parse Business Details         | Code                          | Parse search HTML for business listings   | ScrapeOps - Google Maps Search  | ScrapeOps - Business Details    | ## 2. Deep Scraping Search Maps & extract full dentist details.                                                      |
| ScrapeOps - Business Details   | ScrapeOps Scraper             | Scrape individual business detail pages   | Parse Business Details          | Parse Full Business Info        | ## 2. Deep Scraping Search Maps & extract full dentist details.                                                      |
| Parse Full Business Info       | Code                          | Parse detailed business pages & enrich    | ScrapeOps - Business Details    | Read Previous Entries from Sheet | ## 3. Deduplication Filter out existing businesses from Sheet.                                                      |
| Read Previous Entries from Sheet| Google Sheets                 | Read historical leads to deduplicate      | Parse Full Business Info        | Compare With Previous Run       | ## 3. Deduplication Filter out existing businesses from Sheet.                                                      |
| Compare With Previous Run      | Code                          | Deduplicate current leads vs sheet data   | Read Previous Entries from Sheet| Filter New Leads                | ## 3. Deduplication Filter out existing businesses from Sheet.                                                      |
| Filter New Leads              | If                             | Filter leads marked as "new"               | Compare With Previous Run       | Save to Google Sheets           | ## 4. Save & Alert Store data and notify via Email/Slack.                                                             |
| Save to Google Sheets          | Google Sheets                 | Append new leads to Google Sheet           | Filter New Leads                | Send Gmail Alert for New Leads  | ## 4. Save & Alert Store data and notify via Email/Slack.                                                             |
| Send Gmail Alert for New Leads | Gmail                         | Send email notification of new leads      | Save to Google Sheets           | Send a message (Slack)          | ## 4. Save & Alert Store data and notify via Email/Slack.                                                             |
| Send a message                | Slack                         | Send Slack notification of new leads      | Send Gmail Alert for New Leads  | None                          | ## 4. Save & Alert Store data and notify via Email/Slack.                                                             |
| Sticky Note                  | Sticky Note                   | Documentation Summary                      | None                          | None                          | # 🗺️ Google Maps Dentist Finder ... Setup steps, customization notes, and links to template and ScrapeOps signup.    |
| Sticky Note1                 | Sticky Note                   | Section label for Input & Config           | None                          | None                          | ## 1. Input & Config Capture city input and set search terms.                                                        |
| Sticky Note2                 | Sticky Note                   | Section label for Deep Scraping             | None                          | None                          | ## 2. Deep Scraping Search Maps & extract full dentist details.                                                      |
| Sticky Note3                 | Sticky Note                   | Section label for Deduplication             | None                          | None                          | ## 3. Deduplication Filter out existing businesses from Sheet.                                                      |
| Sticky Note4                 | Sticky Note                   | Section label for Save & Alert              | None                          | None                          | ## 4. Save & Alert Store data and notify via Email/Slack.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "Form - City Input"  
   - Webhook Path: `form-city-input-webhook`  
   - Form Title: "Tell Which City"  
   - Add one required text field labeled "City Name"  
   - This node captures user input for the city.

2. **Add Set Node for Search Configuration**  
   - Type: Set  
   - Name: "Set Google Maps Configuration"  
   - Add fields:  
     - city: expression `{{$json.city || $json["City Name"] || ""}}`  
     - keyword: string `"dentist"` (default, can be changed)  
     - baseUrl: string `"https://www.google.com/maps/search/"`  
     - searchUrl: expression `"https://www.google.com/maps/search/dentist+in+" + String($json.city || $json["City Name"] || "").replace(/\s+/g, '+')`  
   - Connect output of Form Trigger to this node.

3. **Add ScrapeOps Node for Google Maps Search**  
   - Type: ScrapeOps (custom n8n node)  
   - Name: "ScrapeOps - Google Maps Search"  
   - URL parameter: expression `{{$json.searchUrl}}`  
   - Return Type: `htmlResponse`  
   - Advanced Options:  
     - Wait: 12000 ms (12 seconds)  
     - Render JS: true  
     - Residential proxy: false  
   - Add your ScrapeOps API credentials.  
   - Connect output of Set node to this node.

4. **Add Code Node to Parse Business Details**  
   - Type: Code  
   - Name: "Parse Business Details"  
   - Paste the provided JavaScript code that extracts business listings from the HTML page.  
   - Connect output of ScrapeOps Search node to this node.

5. **Add ScrapeOps Node for Individual Business Details**  
   - Type: ScrapeOps  
   - Name: "ScrapeOps - Business Details"  
   - URL parameter: expression `{{$json.mapUrl}}` (iterates over each business)  
   - Same advanced options as previous ScrapeOps node.  
   - Use same ScrapeOps credentials.  
   - Connect output of Parse Business Details node to this node.

6. **Add Code Node to Parse Full Business Info**  
   - Type: Code  
   - Name: "Parse Full Business Info"  
   - Paste the provided JavaScript code that extracts enriched details (phone, website, reviews, etc.) from detail pages and merges with partial data.  
   - Connect output of ScrapeOps Business Details node to this node.

7. **Add Google Sheets Node to Read Previous Entries**  
   - Type: Google Sheets  
   - Name: "Read Previous Entries from Sheet"  
   - Operation: Read rows from your lead tracking Google Sheet.  
   - Configure OAuth2 credentials for Google Sheets.  
   - Specify your spreadsheet URL and sheet name (`gid=0` or your specific sheet).  
   - Connect output of Parse Full Business Info node to this node.

8. **Add Code Node to Compare With Previous Run**  
   - Type: Code  
   - Name: "Compare With Previous Run"  
   - Paste the provided JavaScript code that normalizes and compares current leads to previous sheet entries, marking each as "new" or "old".  
   - Connect output of Read Previous Entries node to this node.

9. **Add If Node to Filter New Leads**  
   - Type: If  
   - Name: "Filter New Leads"  
   - Condition: Check if `{{$json.status}}` equals `"new"`  
   - Connect output of Compare With Previous Run node to this node.

10. **Add Google Sheets Node to Save New Leads**  
    - Type: Google Sheets  
    - Name: "Save to Google Sheets"  
    - Operation: Append rows  
    - Configure columns to map all relevant fields from lead JSON: businessName, phone, website, rating, totalReviews, address, city, category, mapUrl, status, checkedAt, lgbtqFriendly, review1, review2, review3.  
    - Use same Google Sheets OAuth2 credentials and specify your target sheet.  
    - Connect the "true" output of Filter New Leads node to this node.

11. **Add Gmail Node to Send Email Alert**  
    - Type: Gmail  
    - Name: "Send Gmail Alert for New Leads"  
    - Recipient: Your email (e.g. noorsimar.singh@gmail.com)  
    - Subject: `📍 New Businesses Found in {{ $json.city }} (Dentist)` via expression referencing the Set Google Maps Configuration node city field.  
    - Message: Plain text with a link to your Google Sheet.  
    - Use Gmail OAuth2 credentials.  
    - Connect output of Save to Google Sheets node to this node.  
    - Set to execute once per workflow run.

12. **Add Slack Node to Send Slack Message**  
    - Type: Slack  
    - Name: "Send a message"  
    - Configure to send message to a specific user by user ID.  
    - Message text includes notification and Google Sheet link.  
    - Use Slack API credentials.  
    - Connect output of Gmail node to this node.  
    - Set to execute once per workflow run.

13. **Optional: Add Sticky Notes**  
    - Add sticky notes for documentation and section labeling as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates scraping Google Maps for local dentists, deduplicates leads, saves to Google Sheets, and sends email and Slack notifications.                                                                            | Main project purpose                                                                                                         |
| Setup requires ScrapeOps account: https://scrapeops.io/app/register/n8n                                                                                                                                                          | ScrapeOps API credentials                                                                                                    |
| Google Sheet template with headers is available: https://docs.google.com/spreadsheets/d/1HYO6pw9PigmNKzrnmE9s9JcNVSvZAcIHGF1ZJQ9kfks/edit?gid=0                                                                                     | Template Sheet                                                                                                               |
| To customize business type, change the `keyword` in "Set Google Maps Configuration" node (e.g., plumber, gym).                                                                                                                   | Customization hint                                                                                                           |
| For scheduled runs, replace the Form Trigger with a Schedule Trigger.                                                                                                                                                            | Scheduling advice                                                                                                            |
| Parsing uses extensive regex and fallback logic to handle frequent changes in Google Maps HTML structure and data variations, but failures may occur if Google updates page layouts heavily.                                      | Parsing robustness note                                                                                                      |
| The workflow limits HTML data size to avoid memory issues with large pages.                                                                                                                                                      | Performance safeguard                                                                                                        |
| Deduplication logic normalizes names and phones carefully to avoid duplicates and false positives.                                                                                                                              | Deduplication detail                                                                                                         |
| Email and Slack notifications help keep stakeholders informed of new leads found.                                                                                                                                               | Alerting purpose                                                                                                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation platform. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.