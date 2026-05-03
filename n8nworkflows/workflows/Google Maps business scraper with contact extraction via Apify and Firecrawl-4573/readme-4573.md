Google Maps business scraper with contact extraction via Apify and Firecrawl

https://n8nworkflows.xyz/workflows/google-maps-business-scraper-with-contact-extraction-via-apify-and-firecrawl-4573


# Google Maps business scraper with contact extraction via Apify and Firecrawl

### 1. Workflow Overview

This workflow automates the process of scraping Google Maps business data (specifically restaurants) and extracting contact information from their websites. It leverages Apify's Google Places crawler to obtain business listings, then filters those with valid websites, scrapes website content via Firecrawl, extracts contact details including emails and social media profiles, and saves all data into structured Google Sheets. The workflow runs on a schedule, processes data in batches, and includes looping mechanisms to wait for job completion and to iterate through website scraping tasks.

The workflow is logically divided into three main blocks:

- **1.1 Initialization Phase**: Triggering the workflow periodically, reading unprocessed queries from Google Sheets, and starting the Apify scraping job.
- **1.2 Data Collection Phase**: Monitoring the Apify job status until completion, fetching scraped business data, and saving raw business info.
- **1.3 Website Processing Phase**: Filtering businesses with websites, batch processing each website by scraping content, extracting contact info, saving details, and marking records as processed.

Each block contains dedicated nodes with specific roles, including HTTP requests, Google Sheets integration, filtering, looping, and custom code execution.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Initialization Phase

**Overview:**  
This block triggers the workflow every 30 minutes, reads pending (unprocessed) search queries from a Google Sheet, and initiates a new Apify Google Places scraping job for restaurant data in New York, USA.

**Nodes Involved:**  
- Schedule Trigger  
- Read Pending Queries  
- Start Apify Scraping Job  
- Wait for Job Succeed (entry point for next block)

**Node Details:**  

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Starts workflow execution every 30 minutes.  
  - Configuration: Interval trigger set to every 30 minutes.  
  - Input: None (trigger node)  
  - Output: Initiates flow to Read Pending Queries node.  
  - Edge Cases: Trigger failure unlikely; check server uptime and workflow activation status.

- **Read Pending Queries**  
  - Type: Google Sheets  
  - Role: Reads rows from Google Sheet where "Status" is false (unprocessed queries).  
  - Configuration: Reads from sheet with Document ID referencing "Google Maps Scraper" spreadsheet, filtering rows where Status column equals false.  
  - Key Expressions: Filter `Status = false`  
  - Input: Trigger output  
  - Output: Passes query data to Start Apify Scraping Job  
  - Edge Cases: Google API rate limits, authentication errors, empty result sets.

- **Start Apify Scraping Job**  
  - Type: HTTP Request  
  - Role: Starts an Apify actor run for the Google Places scraper act with parameters targeting restaurants in New York.  
  - Configuration: POST request to Apify API endpoint `https://api.apify.com/v2/acts/compass~crawler-google-places/runs`  
  - Request Body: JSON with searchStringsArray containing "restaurant", locationQuery "New York, USA", maxCrawledPlacesPerSearch 15, language "en", no lead enrichment, no images.  
  - Authentication: HTTP Query Auth and Header Auth for Apify API.  
  - Input: Queries from Google Sheets  
  - Output: Job run data forwarded to Wait for Job Succeed node.  
  - Edge Cases: API rate limits, invalid credentials, network timeouts, malformed responses.

- **Wait for Job Succeed**  
  - Type: Wait  
  - Role: Waits for webhook or a timing mechanism (webhookId present) before checking job status.  
  - Configuration: Default wait (likely webhook-based) to pause workflow until notified or timeout.  
  - Input: Job run data from Start Apify Scraping Job  
  - Output: Triggers Check Scraping Status node.  
  - Edge Cases: Webhook not triggered, timeout, connectivity failures.

---

#### 1.2 Data Collection Phase

**Overview:**  
This block monitors the Apify scraping job status, loops until the job is complete, then fetches the scraped restaurant data and saves it into the "Data" sheet of the Google Sheet.

**Nodes Involved:**  
- Check Scraping Status  
- Loop Until Complete (If node)  
- Fetch Scraped Results  
- Save Business Data

**Node Details:**  

- **Check Scraping Status**  
  - Type: HTTP Request  
  - Role: Queries Apify API for current status of the actor run by job ID.  
  - Configuration: GET request to `https://api.apify.com/v2/actor-runs/{{ $json.data.id }}` with headers `Content-Type: application/json` and `Accept: application/json`.  
  - Authentication: HTTP Query Auth and Header Auth for Apify API.  
  - Input: Job run data from Wait for Job Succeed  
  - Output: Status data to Loop Until Complete node.  
  - Edge Cases: API unavailability, invalid job ID, timeouts.

- **Loop Until Complete (If node)**  
  - Type: If  
  - Role: Checks if job status is not "SUCCEEDED" to continue looping.  
  - Configuration: Condition checks that `data.status` is not equal to "SUCCEEDED".  
  - Input: Scraper status JSON  
  - Output:  
    - True: Loops back to Wait for Job Succeed node (wait and re-check)  
    - False: Proceeds to Fetch Scraped Results node.  
  - Edge Cases: Infinite loops if status never updates, malformed status field.

- **Fetch Scraped Results**  
  - Type: HTTP Request  
  - Role: Retrieves scraped data results from Apify dataset using the dataset ID from job data.  
  - Configuration: GET request to `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`  
  - Authentication: HTTP Query Auth and Header Auth for Apify API.  
  - Input: Job completion trigger from Loop Until Complete node  
  - Output: Raw business data to Save Business Data node.  
  - Edge Cases: Dataset ID missing, empty results, API rate limits.

- **Save Business Data**  
  - Type: Google Sheets  
  - Role: Appends scraped business data to "Data" sheet.  
  - Configuration: Maps fields such as phone, title, status (false), address, website, categoryName, and searchString to corresponding columns.  
  - Matching Columns: searchString (used for potential matching)  
  - Input: Scraped data array from Fetch Scraped Results  
  - Output: Passes data to Filter Businesses with Websites node.  
  - Edge Cases: API limits, data format mismatches, authentication errors.

---

#### 1.3 Website Processing Phase

**Overview:**  
This block filters businesses with valid websites, processes each website in batches to scrape content using Firecrawl, extracts contact information via custom code, saves the contact details, and marks businesses as processed in the Google Sheet.

**Nodes Involved:**  
- Filter Businesses with Websites  
- Batch Processing Logic (SplitInBatches)  
- Scrape Website Content  
- Extract Contact Information (Code node)  
- Save Contact Details  
- Mark as Processed

**Node Details:**  

- **Filter Businesses with Websites**  
  - Type: Filter  
  - Role: Filters business records where website is not empty and status is "false" (unprocessed).  
  - Configuration: Conditions require website field to be non-empty and status equal to "false".  
  - Input: Data from Save Business Data node  
  - Output: Businesses with websites to Batch Processing Logic  
  - Edge Cases: Missing or malformed website URLs, status field inconsistencies.

- **Batch Processing Logic**  
  - Type: SplitInBatches  
  - Role: Splits input data into batches to process websites sequentially.  
  - Configuration: Default batch size (not explicitly set)  
  - Input: Filtered businesses  
  - Output: Each batch triggers Scrape Website Content node, empty output triggers no further action.  
  - Edge Cases: Large data sets causing long processing times, batch size configuration.

- **Scrape Website Content**  
  - Type: HTTP Request  
  - Role: Sends POST request to Firecrawl API to scrape HTML content of each business website.  
  - Configuration: URL `https://api.firecrawl.dev/v1/scrape`, JSON body with `url` set to the business website and format set to "html".  
  - Authentication: Bearer token (Firecrawl) and Header Auth (Apify) configured.  
  - Input: Website URL per batch from Batch Processing Logic  
  - Output: HTML content to Extract Contact Information node.  
  - Edge Cases: Website unreachable, rate limits, malformed URLs, API errors.

- **Extract Contact Information**  
  - Type: Code (JavaScript)  
  - Role: Extracts emails and social media profile URLs (LinkedIn, Facebook, Instagram, Twitter/X) from scraped HTML content.  
  - Configuration: Uses multiple regex patterns to find emails and social URLs, cleans results, deduplicates emails, falls back on @username handles for Instagram and Twitter if full URLs are missing.  
  - Key Expressions: Regex for email and social media; fallback logic for handles; URL cleanup.  
  - Input: HTML content JSON from Scrape Website Content  
  - Output: Object with extracted emails, linkedin, facebook, instagram, twitter, and website URL.  
  - Edge Cases: No matches found, false positives, non-standard social URLs, malformed HTML.

- **Save Contact Details**  
  - Type: Google Sheets  
  - Role: Appends extracted contact details to "Details" sheet in the Google Sheet.  
  - Configuration: Columns mapped for emails, twitter, website, facebook, linkedin, instagram.  
  - Input: Extracted contact info from Code node  
  - Output: Passes website URL to Mark as Processed node.  
  - Edge Cases: Google Sheets API errors, data type mismatches.

- **Mark as Processed**  
  - Type: Google Sheets  
  - Role: Updates the "Data" sheet row corresponding to the website, setting status to "true" to mark it processed.  
  - Configuration: Uses website as matching column to update status field to "true".  
  - Input: Website URL from Save Contact Details  
  - Output: Loops back to Batch Processing Logic for next batch.  
  - Edge Cases: Matching failures if website URLs differ, concurrent updates, API limits.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                                           | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                  |
|---------------------------|--------------------|-----------------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Trigger            | Triggers workflow every 30 minutes                         | None                        | Read Pending Queries         | ## 🚀 INITIALIZATION PHASE<br>- Triggers every 30 minutes<br>- Reads unprocessed records from [Google Sheet](https://docs.google.com/spreadsheets/d/1DHezdcetT0c3Ie1xB3z3jDc5WElsLN87K4J9EQDef9g/edit?usp=sharing)<br>- Starts Google Places scraper for restaurants<br>- Waits for completion |
| Read Pending Queries      | Google Sheets      | Reads unprocessed queries from Google Sheet                | Schedule Trigger            | Start Apify Scraping Job     | (See above)                                                                                                  |
| Start Apify Scraping Job  | HTTP Request       | Starts Google Places scraping job on Apify                 | Read Pending Queries        | Wait for Job Succeed         | (See above)                                                                                                  |
| Wait for Job Succeed      | Wait               | Waits for Apify job completion                              | Start Apify Scraping Job    | Check Scraping Status        | (See above)                                                                                                  |
| Check Scraping Status     | HTTP Request       | Checks Apify job status                                     | Wait for Job Succeed        | Loop Until Complete          | ## 📊 DATA COLLECTION PHASE<br>- Monitors scraper job status<br>- Loops until job completes<br>- Fetches scraped restaurant data<br>- Saves to "Data" sheet in [Google Sheets](https://docs.google.com/spreadsheets/d/1DHezdcetT0c3Ie1xB3z3jDc5WElsLN87K4J9EQDef9g/edit?usp=sharing) |
| Loop Until Complete       | If                 | Loops until Apify scraping job status is "SUCCEEDED"       | Check Scraping Status       | Wait for Job Succeed / Fetch Scraped Results | (See above)                                                                                                  |
| Fetch Scraped Results     | HTTP Request       | Retrieves scraped business data from Apify dataset         | Loop Until Complete         | Save Business Data           | (See above)                                                                                                  |
| Save Business Data        | Google Sheets      | Appends scraped business info to "Data" sheet              | Fetch Scraped Results       | Filter Businesses with Websites | (See above)                                                                                                  |
| Filter Businesses with Websites | Filter         | Filters businesses with websites and unprocessed status    | Save Business Data          | Batch Processing Logic       | ## 🌐 WEBSITE PROCESSING PHASE<br>- Filters restaurants with valid websites<br>- Loops through each website<br>- Scrapes website content with Firecrawl<br>- Extracts contact information (emails, social media)<br>- Saves to "Details" sheet and marks as processed |
| Batch Processing Logic    | SplitInBatches     | Processes businesses in batches for website scraping       | Filter Businesses with Websites | Scrape Website Content / End of batch | (See above)                                                                                                  |
| Scrape Website Content    | HTTP Request       | Scrapes website HTML content via Firecrawl API             | Batch Processing Logic      | Extract Contact Information  | (See above)                                                                                                  |
| Extract Contact Information | Code (JavaScript) | Extracts emails and social media profiles from HTML content | Scrape Website Content      | Save Contact Details         | (See above)                                                                                                  |
| Save Contact Details      | Google Sheets      | Saves extracted contact information to "Details" sheet     | Extract Contact Information | Mark as Processed            | (See above)                                                                                                  |
| Mark as Processed         | Google Sheets      | Updates business status to processed in "Data" sheet       | Save Contact Details        | Batch Processing Logic       | (See above)                                                                                                  |
| Sticky Note               | Sticky Note        | Notes for Initialization Phase                             | None                        | None                        | See content in Initialization Phase                                                                         |
| Sticky Note1              | Sticky Note        | Notes for Data Collection Phase                            | None                        | None                        | See content in Data Collection Phase                                                                         |
| Sticky Note2              | Sticky Note        | Notes for Website Processing Phase                         | None                        | None                        | See content in Website Processing Phase                                                                      |
| Sticky Note3              | Sticky Note        | Notes on looping mechanisms                                | None                        | None                        | ## 🔄 LOOPS<br>**Status Check Loop**: Continues until scraper job is complete<br>**Website Processing Loop**: Processes each restaurant website individually |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new n8n workflow.**

2. **Add a "Schedule Trigger" node:**  
   - Set to trigger every 30 minutes (minutes interval = 30).

3. **Add "Read Pending Queries" node (Google Sheets):**  
   - Connect it to Schedule Trigger output.  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Set document ID to your spreadsheet ID (e.g., "1DHezdcetT0c3Ie1xB3z3jDc5WElsLN87K4J9EQDef9g").  
   - Sheet name: default sheet with `gid=0`.  
   - Set filter to read only rows where "Status" column is "false".

4. **Add "Start Apify Scraping Job" node (HTTP Request):**  
   - Connect to Read Pending Queries output.  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/compass~crawler-google-places/runs`  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "searchStringsArray": ["restaurant"],
       "locationQuery": "New York, USA",
       "maxCrawledPlacesPerSearch": 15,
       "language": "en",
       "maximumLeadsEnrichmentRecords": 0,
       "maxImages": 0
     }
     ```  
   - Authentication: Use HTTP Query Auth and HTTP Header Auth with your Apify credentials.  
   - Headers: Content-Type and Accept set to application/json.

5. **Add "Wait for Job Succeed" node (Wait):**  
   - Connect to Start Apify Scraping Job output.  
   - Configure with webhookId or default wait settings.

6. **Add "Check Scraping Status" node (HTTP Request):**  
   - Connect Wait for Job Succeed output to this node.  
   - Method: GET  
   - URL: `https://api.apify.com/v2/actor-runs/{{ $json.data.id }}` (use expression for dynamic ID).  
   - Headers: Content-Type and Accept set to application/json.  
   - Authentication: same Apify credentials as before.

7. **Add "Loop Until Complete" node (If):**  
   - Connect Check Scraping Status output to "If" node.  
   - Condition: `$json.data.status` not equal to "SUCCEEDED".  
   - True output connects back to "Wait for Job Succeed" to recheck status.  
   - False output proceeds to Fetch Scraped Results.

8. **Add "Fetch Scraped Results" node (HTTP Request):**  
   - Connect False output of If node.  
   - Method: GET  
   - URL: `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`  
   - Headers and Auth: same as previous Apify nodes.

9. **Add "Save Business Data" node (Google Sheets):**  
   - Connect Fetch Scraped Results output.  
   - Authenticate with Google Sheets.  
   - Document ID: same spreadsheet.  
   - Sheet name: "Data" (use gid=1948906848 or sheet name).  
   - Operation: Append  
   - Map columns: phone, title, status (set to "false"), address, website, categoryName, searchString.

10. **Add "Filter Businesses with Websites" node (Filter):**  
    - Connect Save Business Data output.  
    - Conditions: website not empty AND status equals "false".  

11. **Add "Batch Processing Logic" node (SplitInBatches):**  
    - Connect Filter Businesses with Websites output.  
    - Leave batch size default or configure as needed.

12. **Add "Scrape Website Content" node (HTTP Request):**  
    - Connect Batch Processing Logic output (first output).  
    - Method: POST  
    - URL: `https://api.firecrawl.dev/v1/scrape`  
    - Body: JSON with url set to `{{ $json.website }}` and formats as ["html"].  
    - Authentication: HTTP Bearer Auth with Firecrawl token.  
    - Headers: optionally add Apify Header Auth if required.

13. **Add "Extract Contact Information" node (Code):**  
    - Connect Scrape Website Content output.  
    - Paste the provided JavaScript code that extracts emails and social profiles from HTML content.

14. **Add "Save Contact Details" node (Google Sheets):**  
    - Connect Extract Contact Information output.  
    - Authenticate with Google Sheets.  
    - Document ID: same spreadsheet.  
    - Sheet name: "Details" (gid=2056137853 or sheet name).  
    - Operation: Append  
    - Map columns: emails, twitter, website, facebook, linkedin, instagram.

15. **Add "Mark as Processed" node (Google Sheets):**  
    - Connect Save Contact Details output.  
    - Authenticate with Google Sheets.  
    - Document ID: same spreadsheet.  
    - Sheet name: "Data"  
    - Operation: Update  
    - Matching columns: website  
    - Set status field to "true".

16. **Connect "Mark as Processed" output back to "Batch Processing Logic" (second output) to continue batch processing.**

17. **Add Sticky Notes (optional) to document phases and loops.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow relies on Apify's Google Places scraper act for restaurant data.                                  | Apify actor: https://apify.com/compass/crawler-google-places                                    |
| Firecrawl API used for website content scraping and contact info extraction.                              | Firecrawl API docs: https://firecrawl.dev/docs                                                  |
| Google Sheets used extensively for data storage and query tracking.                                      | Google Sheets API: https://developers.google.com/sheets/api                                     |
| The workflow includes two main loops: one for waiting on Apify job completion and one for batch website scraping. | Sticky Note3 explains looping logic.                                                            |
| Regex patterns in code node include email and social media extraction with fallback for @username patterns. | Useful for robust contact extraction from semi-structured HTML content.                         |
| Ensure all API credentials (Apify, Firecrawl, Google Sheets OAuth2) are properly configured before running. | Credential errors are common failure points.                                                    |
| This workflow is designed to be scalable and can be modified for other search strings or locations.       | Modify JSON body in Start Apify Scraping Job node accordingly.                                  |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.