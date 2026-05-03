Lead Generation System: Google Maps to Email Scraper with Google Sheets Export

https://n8nworkflows.xyz/workflows/lead-generation-system--google-maps-to-email-scraper-with-google-sheets-export-5385


# Lead Generation System: Google Maps to Email Scraper with Google Sheets Export

### 1. Workflow Overview

This workflow automates lead generation by scraping Google Maps business listings and extracting emails from their websites, then exporting the clean email list into Google Sheets. It is designed for marketers and sales professionals targeting specific business categories within a geographic area without relying on paid APIs.

The workflow consists of four logical blocks:

- **1.1 Google Maps Data Extraction:** Scrapes Google Maps search results for business listings using HTTP requests, retrieving raw HTML containing business names and website URLs.

- **1.2 Website URL Processing:** Extracts website URLs from the raw HTML, filters out irrelevant Google domains, removes duplicates, and limits the batch size for controlled processing.

- **1.3 Smart Website Scraping:** Iteratively downloads each business website’s HTML content with built-in delays and error handling to avoid IP blocking and rate limits.

- **1.4 Email Extraction & Export:** Extracts emails from website HTML using regex, filters entries without emails, deduplicates the final email list, and appends the results to a Google Sheets spreadsheet for outreach.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Google Maps Data Extraction

**Overview:**  
Initiates the workflow by scraping Google Maps search results for a specific query (e.g., "Calgary dentists") using an HTTP request node. The node fetches raw HTML containing business listings and website URLs.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Scrape Google Maps (HTTP Request)  
- Extract URLs (Code)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution manually for testing or live runs.  
  - Configuration: No parameters needed.  
  - Input: None  
  - Output: Triggers next node (Scrape Google Maps).  
  - Edge Cases: None typical; user must trigger manually.

- **Scrape Google Maps**  
  - Type: HTTP Request  
  - Role: Fetches Google Maps search results page HTML for a specified query URL.  
  - Configuration:  
    - URL set to "https://www.google.com/maps/search/calgary+dentists" (replaceable for other queries).  
    - Full HTTP response captured.  
    - Allows unauthorized certificates (for robustness).  
  - Input: Trigger from Manual Trigger node.  
  - Output: Full HTML response passed to Extract URLs node.  
  - Edge Cases:  
    - Potential IP blocking by Google if overused.  
    - HTTP errors if URL changes or is blocked.  
  - Version: 4.2 required for fullResponse option.

- **Extract URLs**  
  - Type: Code (JavaScript)  
  - Role: Parses raw HTML to extract all website URLs using regex.  
  - Configuration:  
    - Uses regex matching for URLs starting with http(s).  
    - Outputs each URL as a separate JSON item with key `website`.  
  - Input: Raw HTML from Scrape Google Maps.  
  - Output: Array of website URLs for filtering.  
  - Edge Cases:  
    - Regex may capture irrelevant URLs if HTML structure changes.  
    - Potential empty or malformed URLs.

---

#### 1.2 Website URL Processing

**Overview:**  
Cleans the extracted URLs by filtering out Google-owned or irrelevant domains, removes duplicates, and limits the number of URLs to process for testing or controlled runs.

**Nodes Involved:**  
- Filter Google URLs (Filter)  
- Remove Duplicates (RemoveDuplicates)  
- Limit (Limit)

**Node Details:**

- **Filter Google URLs**  
  - Type: Filter  
  - Role: Excludes URLs containing domains like "google", "gstatic", "schema", or "gg" to remove non-business websites.  
  - Configuration:  
    - Conditions set to exclude strings: "schema", "google", "gg", "gstatic" within `website` field.  
  - Input: URLs from Extract URLs node.  
  - Output: Filtered list of business websites.  
  - Edge Cases:  
    - If new Google domains appear, may need updates.  
    - False negatives possible if businesses have URLs containing these strings.

- **Remove Duplicates**  
  - Type: RemoveDuplicates  
  - Role: Removes duplicate website URLs to avoid redundant scraping.  
  - Configuration: Default (removes duplicates based on JSON content).  
  - Input: Filtered URLs from Filter Google URLs.  
  - Output: Unique list of websites.  
  - Edge Cases: None significant.

- **Limit**  
  - Type: Limit  
  - Role: Restricts the number of URLs to process (default maxItems=10), useful for testing or avoiding overload.  
  - Configuration: maxItems set to 10 by default.  
  - Input: Unique URLs from Remove Duplicates.  
  - Output: Limited subset of URLs forwarded for scraping.  
  - Edge Cases:  
    - Need to increase or remove limit in production for full results.

---

#### 1.3 Smart Website Scraping

**Overview:**  
Processes each website URL sequentially by downloading its HTML content with built-in waits and error handling to prevent IP blocking and rate limiting. This block uses batching and delays to scrape safely.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Wait1 (Wait)  
- Scrape Site (HTTP Request)  
- Wait (Wait)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes websites one at a time to avoid simultaneous requests that can trigger IP blocking.  
  - Configuration: Defaults, batch size of 1 (implied by sequential processing).  
  - Input: Limited list of websites from Limit node.  
  - Output: One website item per batch.  
  - Edge Cases:  
    - Batch size change may affect rate limits.

- **Wait1**  
  - Type: Wait  
  - Role: Adds delay before each website scrape to avoid rapid requests.  
  - Configuration: Default delay (not explicitly set, implies minimal delay).  
  - Input: Each website item from Loop Over Items.  
  - Output: Passes to Scrape Site node.  
  - Edge Cases:  
    - Insufficient delay may cause blocking.

- **Scrape Site**  
  - Type: HTTP Request  
  - Role: Downloads the HTML content of each website URL.  
  - Configuration:  
    - URL dynamically set to current website URL (`={{ $json.website }}`).  
    - Redirect following disabled to avoid unnecessary redirects that may complicate scraping.  
    - On error: continue workflow output to avoid halting on failed sites.  
  - Input: Website URL item from Wait1.  
  - Output: Website HTML content to Wait node.  
  - Edge Cases:  
    - Site may block scraping or respond with errors.  
    - Redirects may be missed due to disabled followRedirects.

- **Wait**  
  - Type: Wait  
  - Role: Adds a 1-second delay after scraping each site to further reduce IP blocking risk.  
  - Configuration: amount=1 second.  
  - Input: HTML content from Scrape Site.  
  - Output: Passes to Extract Emails node.  
  - Edge Cases: None significant.

---

#### 1.4 Email Extraction & Export

**Overview:**  
Extracts email addresses from the scraped website HTML, filters out entries with no emails, splits email arrays into individual items, removes duplicates, and exports the clean list to a Google Sheets spreadsheet.

**Nodes Involved:**  
- Extract Emails (Code)  
- Loop Over Items (SplitInBatches) [reused]  
- Filter Out Empties (Filter)  
- Split Out (SplitOut)  
- Remove Duplicates (RemoveDuplicates) [second instance]  
- Add to Sheet (or whatever you want!) (Google Sheets)

**Node Details:**

- **Extract Emails**  
  - Type: Code (JavaScript)  
  - Role: Uses regex to find all email addresses in website HTML content.  
  - Configuration:  
    - Regex matches standard email patterns, excludes image file extensions.  
    - Returns JSON with `emails` key containing array of found emails or null.  
    - On error: continues without stopping workflow.  
  - Input: Website HTML content from Wait node.  
  - Output: JSON with array of emails.  
  - Edge Cases:  
    - Regex may miss obfuscated or javascript-generated emails.  
    - Empty or null emails handled downstream.

- **Loop Over Items (reused)**  
  - Processes each email extraction result individually, feeding into filter.

- **Filter Out Empties**  
  - Type: Filter  
  - Role: Removes items where no emails are found (empty or null email arrays).  
  - Configuration: Checks existence of `emails` array.  
  - Input: Email extraction results.  
  - Output: Only items with emails proceed.  
  - Edge Cases: None significant.

- **Split Out**  
  - Type: SplitOut  
  - Role: Converts each email array into individual items (one email per item).  
  - Configuration: Field to split out is `emails`.  
  - Input: Filtered email arrays.  
  - Output: Individual email items for deduplication.  
  - Edge Cases: None significant.

- **Remove Duplicates (2)**  
  - Type: RemoveDuplicates  
  - Role: Removes duplicate emails across all processed websites.  
  - Configuration: Default deduplication.  
  - Input: Individual email items.  
  - Output: Unique email list for export.  
  - Edge Cases: None significant.

- **Add to Sheet (or whatever you want!)**  
  - Type: Google Sheets  
  - Role: Appends the final email list to a specified Google Sheets spreadsheet for storage and outreach.  
  - Configuration:  
    - Appends to sheet with gid=0 in spreadsheet ID "1fcijyZM1oU73i2xUbXYJ4j6RshmVEduOkCJji2SJP68".  
    - Mapping column: "emails".  
    - Uses Google Sheets OAuth2 credentials named "YouTube".  
  - Input: Deduplicated email items.  
  - Output: None (final node).  
  - Edge Cases:  
    - Credential expiry or permission issues.  
    - Spreadsheet or sheet not found or access revoked.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                             | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                                                |
|----------------------------|---------------------|---------------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts workflow manually                     | None                         | Scrape Google Maps              |                                                                                                                                             |
| Scrape Google Maps          | HTTP Request        | Fetches Google Maps search result HTML      | When clicking ‘Test workflow’ | Extract URLs                   | Step 1: Google Maps Data Extraction. Uses HTTP requests to scrape business listings without API. Replace URL for target location/business.  |
| Extract URLs               | Code (JavaScript)    | Extracts all URLs from raw HTML              | Scrape Google Maps            | Filter Google URLs              | Step 2: Website URL Processing. Extracts URLs with regex for further filtering.                                                              |
| Filter Google URLs          | Filter              | Removes Google and irrelevant domains       | Extract URLs                  | Remove Duplicates               | Step 2: Filters out google.com, gstatic, and similar domains to keep only business websites.                                                |
| Remove Duplicates           | RemoveDuplicates     | Removes duplicate website URLs               | Filter Google URLs            | Limit                         | Step 2: Ensures unique website URLs for scraping.                                                                                           |
| Limit                      | Limit               | Limits number of URLs to process             | Remove Duplicates             | Loop Over Items                | Step 2: Controls batch size for testing or production.                                                                                      |
| Loop Over Items            | SplitInBatches      | Processes websites one by one                 | Limit                        | Wait1, Scrape Site             | Step 3: Smart Website Scraping. Loops over URLs with delays to avoid blocking.                                                              |
| Wait1                      | Wait                | Delay before scraping each website           | Loop Over Items               | Scrape Site                   | Step 3: Adds delay to prevent IP blocking and rate limits.                                                                                  |
| Scrape Site                | HTTP Request        | Downloads website HTML content                | Wait1                        | Wait                         | Step 3: Fetches each business website HTML for email extraction. Continues on error.                                                        |
| Wait                       | Wait                | Delay after scraping each site                | Scrape Site                  | Extract Emails                | Step 3: Adds 1 second wait to pace requests safely.                                                                                        |
| Extract Emails             | Code (JavaScript)    | Extracts emails from website HTML             | Wait                         | Loop Over Items (for emails)  | Step 4: Uses regex to find emails; excludes image file extensions. Continues on error.                                                      |
| Loop Over Items            | SplitInBatches      | Processes extracted email arrays              | Extract Emails               | Filter Out Empties             | Step 4: Iterates over each extracted array of emails.                                                                                      |
| Filter Out Empties          | Filter              | Removes entries with no emails                 | Loop Over Items (emails)      | Split Out                    | Step 4: Filters out websites without any extracted emails.                                                                                  |
| Split Out                  | SplitOut            | Splits email arrays into individual email items | Filter Out Empties            | Remove Duplicates (2)          | Step 4: Converts arrays to individual email items for deduplication.                                                                        |
| Remove Duplicates (2)       | RemoveDuplicates     | Removes duplicate emails                       | Split Out                    | Add to Sheet (or whatever you want!) | Step 4: Final deduplication to ensure unique emails before export.                                                                          |
| Add to Sheet (or whatever you want!) | Google Sheets       | Appends emails to Google Sheets spreadsheet    | Remove Duplicates (2)         | None                          | Step 4: Exports clean email list to Google Sheets for outreach. Requires Google Sheets OAuth2 credentials.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters needed.

2. **Add an HTTP Request node to scrape Google Maps**  
   - Name: "Scrape Google Maps"  
   - URL: Set to your target search, e.g., `https://www.google.com/maps/search/calgary+dentists`  
   - Options: Enable fullResponse to get full HTTP response details; allow unauthorized certificates.  
   - Connect "When clicking ‘Test workflow’" node output to this node.

3. **Add a Code node to extract URLs from the HTML**  
   - Name: "Extract URLs"  
   - Language: JavaScript  
   - Code: Use regex to extract all URLs beginning with http or https from the HTML text in `data` field.  
   - Output: Map each matched URL to `{json: {website: url}}`.  
   - Connect "Scrape Google Maps" node output to this node.

4. **Add a Filter node to exclude Google and irrelevant URLs**  
   - Name: "Filter Google URLs"  
   - Condition: Exclude if `website` contains any of "schema", "google", "gg", or "gstatic".  
   - Connect "Extract URLs" node output to this node.

5. **Add a RemoveDuplicates node**  
   - Name: "Remove Duplicates"  
   - Default configuration to remove duplicate URLs.  
   - Connect "Filter Google URLs" node output to this node.

6. **Add a Limit node to control batch size**  
   - Name: "Limit"  
   - Set maxItems to 10 (adjustable for production).  
   - Connect "Remove Duplicates" node output to this node.

7. **Add a SplitInBatches node to loop over URLs one by one**  
   - Name: "Loop Over Items"  
   - Default batch size (1).  
   - Connect "Limit" node output to this node.

8. **Add a Wait node before scraping each site**  
   - Name: "Wait1"  
   - Default delay (optional to increase).  
   - Connect "Loop Over Items" node output (main) to this node.

9. **Add an HTTP Request node to scrape each business website**  
   - Name: "Scrape Site"  
   - URL: Set to expression `{{$json["website"]}}` to dynamically use each URL.  
   - Options: Disable followRedirects to false; enable "continue on error" to avoid workflow halt.  
   - Connect "Wait1" node output to this node.

10. **Add a Wait node after scraping site**  
    - Name: "Wait"  
    - Set amount to 1 second.  
    - Connect "Scrape Site" node output to this node.

11. **Add a Code node to extract emails from site HTML**  
    - Name: "Extract Emails"  
    - Language: JavaScript  
    - Code: Use regex to match email addresses excluding image extensions, return `{json:{emails: [...]}}`.  
    - Set node to continue on error.  
    - Connect "Wait" node output to this node.

12. **Add another SplitInBatches node to loop over extracted email arrays**  
    - Name: "Loop Over Items" (reuse or create new)  
    - Connect "Extract Emails" node output to this node.

13. **Add a Filter node to remove empty email arrays**  
    - Name: "Filter Out Empties"  
    - Condition: Check if `emails` field exists and is non-empty.  
    - Connect "Loop Over Items" node output to this node.

14. **Add a SplitOut node to convert arrays into single email items**  
    - Name: "Split Out"  
    - Field to split out: `emails`  
    - Connect "Filter Out Empties" node output to this node.

15. **Add a RemoveDuplicates node to deduplicate emails**  
    - Name: "Remove Duplicates (2)"  
    - Default settings.  
    - Connect "Split Out" node output to this node.

16. **Add a Google Sheets node to append emails to a sheet**  
    - Name: "Add to Sheet (or whatever you want!)"  
    - Operation: Append  
    - Sheet Name: Set to the target sheet (e.g., gid=0)  
    - Document ID: Your Google Sheets document ID  
    - Mapping Mode: Define below, map `emails` column to `{{$json["emails"]}}`  
    - Credentials: Configure with Google Sheets OAuth2 credentials (e.g., named "YouTube")  
    - Connect "Remove Duplicates (2)" node output to this node.

17. **Ensure connections are made as per the workflow:**  
    - Manual Trigger → Scrape Google Maps → Extract URLs → Filter Google URLs → Remove Duplicates → Limit → Loop Over Items → Wait1 → Scrape Site → Wait → Extract Emails → Loop Over Items → Filter Out Empties → Split Out → Remove Duplicates (2) → Add to Sheet.

18. **Test the workflow with small batch size first to verify proper execution and adjust delays or batch sizes to avoid blocking.**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow scrapes Google Maps without using official APIs, relying on HTML scraping only. | Be aware of Google’s terms of use and potential legal or technical restrictions on scraping.                     |
| Batch size and delays are critical to prevent IP blocking. Adjust 'Limit' and 'Wait' nodes accordingly. | Sticky note in Step 3 emphasizes this for reliable operation at scale.                                          |
| Google Sheets OAuth2 credentials are required with proper permissions for the target spreadsheet. | Configure credentials in n8n’s credential manager before running the workflow.                                   |
| Regex in "Extract Emails" excludes common image extensions to avoid false positives.           | Code node uses `/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g`.               |
| Replace the Google Maps search URL in "Scrape Google Maps" node to target different locations/businesses. | See Sticky Note 1 for customization instructions.                                                               |

---

**Disclaimer:** This documentation is generated solely from an n8n workflow and complies strictly with content policies. All data processed is legal and public.