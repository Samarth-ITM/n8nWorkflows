Scrape Business Emails from Google Maps Listings to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-business-emails-from-google-maps-listings-to-google-sheets-8760


# Scrape Business Emails from Google Maps Listings to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of business email addresses from Google Maps listings and records them into a Google Sheet. It targets users needing to compile contact data (emails) for businesses such as local dentists, without relying on paid APIs. The workflow is structured into four logical blocks:

- **1.1 Google Maps Data Extraction:** Scrapes Google Maps search results for business listings by sending an HTTP request for a specific query (e.g., "Calgary dentists") and retrieves raw HTML containing business details and URLs.

- **1.2 Website URL Processing:** Extracts website URLs from the HTML, filters out irrelevant or tracking URLs related to Google, removes duplicates, and limits the list size to manage processing load.

- **1.3 Smart Website Scraping:** Processes each filtered website URL individually to scrape their HTML content, using delays to prevent IP blocking and includes error handling to continue despite failures.

- **1.4 Email Extraction & Export:** Extracts email addresses from each website’s HTML, filters out entries without emails, splits multiple emails into single items, removes duplicates again, and appends the clean list into a configured Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Maps Data Extraction

- **Overview:** This block initiates the workflow by manually triggering an HTTP request to Google Maps with a specified search query, returning raw HTML containing business listings and URLs.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Scrape Google Maps (HTTP Request)  
  - Extract URLs (Code)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point; workflow starts manually by user trigger  
    - Configuration: No parameters; activated by “Test workflow” button  
    - Inputs: None  
    - Outputs: Connects to “Scrape Google Maps”  
    - Edge Cases: None inherent; user must start workflow manually  

  - **Scrape Google Maps**  
    - Type: HTTP Request  
    - Role: Fetches Google Maps search results page for “calgary dentists”  
    - Configuration:  
      - URL: `https://www.google.com/maps/search/calgary+dentists` (replaceable for other queries)  
      - Options: Allows unauthorized SSL certificates, returns full HTTP response  
    - Inputs: Manual trigger  
    - Outputs: Raw HTML data to “Extract URLs”  
    - Edge Cases: Potential blocking by Google, captcha, or anti-scraping measures; HTTP errors; network timeouts  

  - **Extract URLs**  
    - Type: Code (JavaScript)  
    - Role: Uses regex to extract all URLs from the raw HTML text  
    - Configuration:  
      - Regex: `/https?:\/\/[^\/\s"'>]+/g` to capture URLs in the HTML  
      - Output: Returns one item per extracted URL in JSON format  
    - Inputs: Raw HTML from previous node  
    - Outputs: List of URLs to “Filter Google URLs”  
    - Edge Cases: Regex might capture irrelevant URLs; empty or malformed HTML causing no matches  

---

#### 2.2 Website URL Processing

- **Overview:** Cleans the extracted URLs by removing Google-related or tracking domains, removing duplicates, and limiting the number processed per run to 10 for practical batch handling.

- **Nodes Involved:**  
  - Filter Google URLs (Filter)  
  - Remove Duplicates (Remove Duplicates)  
  - Limit (Limit)  

- **Node Details:**

  - **Filter Google URLs**  
    - Type: Filter  
    - Role: Removes URLs containing “schema”, “google”, “gg”, or “gstatic” strings, which are irrelevant or tracking URLs  
    - Configuration: Multiple string “does not contain” conditions combined with AND logic  
    - Inputs: List of URLs  
    - Outputs: Filtered URLs to “Remove Duplicates”  
    - Edge Cases: Over-filtering could remove legitimate URLs; case sensitivity enforced  

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Role: Deduplicates URLs to avoid repeated processing  
    - Configuration: Default deduplication of all items  
    - Inputs: Filtered URLs  
    - Outputs: Unique URLs to “Limit”  
    - Edge Cases: None significant; depends on consistent URL formatting  

  - **Limit**  
    - Type: Limit  
    - Role: Restricts processing to maximum 10 URLs per workflow execution to avoid overload  
    - Configuration: `maxItems` set to 10  
    - Inputs: Unique URLs  
    - Outputs: Limited URL batch to “Loop Over Items”  
    - Edge Cases: Limits batch size; adjust for scalability or full scraping  

---

#### 2.3 Smart Website Scraping

- **Overview:** Processes each website URL sequentially to scrape their HTML content. Includes wait nodes to delay requests, preventing IP bans or rate limiting. Designed to be fault-tolerant by continuing on errors.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches)  
  - Scrape Site (HTTP Request)  
  - Wait (Wait)  
  - Wait1 (Wait)  
  - Filter Out Empties (Filter)

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes websites one at a time to control request volume  
    - Configuration: Default batch processing of single items  
    - Inputs: Limited list of URLs  
    - Outputs: Two parallel paths: one to “Scrape Site”, one to “Wait1” (filtering)  
    - Edge Cases: Large input lists require batching for performance  

  - **Scrape Site**  
    - Type: HTTP Request  
    - Role: Downloads HTML content from each business website URL  
    - Configuration:  
      - URL: dynamic from `{{$json.website}}`  
      - Redirects: does not follow redirects (to avoid tracking or errors)  
      - On error: continues workflow without stopping if request fails  
    - Inputs: Single URL per batch  
    - Outputs: Website HTML content to “Wait”  
    - Edge Cases: HTTP errors, timeouts, redirects, or sites blocking scraping  

  - **Wait**  
    - Type: Wait  
    - Role: Short delay (1 second) after scraping before processing emails, reduces rate limiting risk  
    - Configuration: Wait time set to 1 second  
    - Inputs: Website HTML content  
    - Outputs: Passes to “Extract Emails”  
    - Edge Cases: None significant  

  - **Wait1**  
    - Type: Wait  
    - Role: Adds delay between requests (exact time unspecified) to avoid server blocking  
    - Configuration: Default wait, no explicit delay parameter set (implied)  
    - Inputs: Single website URL from batch  
    - Outputs: Filter Out Empties  
    - Edge Cases: Insufficient delay may cause IP blocking; too long delays impact throughput  

  - **Filter Out Empties**  
    - Type: Filter  
    - Role: Passes only items where ‘emails’ field exists and is not empty  
    - Configuration: Checks existence of ‘emails’ array in JSON  
    - Inputs: From “Wait1” node downstream  
    - Outputs: Non-empty email items to “Split Out”  
    - Edge Cases: May exclude valid sites with no visible emails  

---

#### 2.4 Email Extraction & Export

- **Overview:** Extracts emails from website HTML using regex, splits multiple emails into individual items, removes duplicates, and appends the final list to a Google Sheet.

- **Nodes Involved:**  
  - Extract Emails (Code)  
  - Split Out (Split Out)  
  - Remove Duplicates (2) (Remove Duplicates)  
  - Add to Sheet (Google Sheets)

- **Node Details:**

  - **Extract Emails**  
    - Type: Code (JavaScript)  
    - Role: Uses regex to find all email addresses in website HTML content  
    - Configuration:  
      - Regex: `/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g`  
        (Excludes image file extensions to avoid false positives)  
      - Returns JSON object with array of emails  
      - On error: continues output even if parsing fails  
    - Inputs: Website HTML content  
    - Outputs: JSON with emails array to “Filter Out Empties”  
    - Edge Cases: May miss obfuscated emails; regex false positives possible  

  - **Split Out**  
    - Type: Split Out  
    - Role: Converts arrays of emails into individual items (one email per item)  
    - Configuration: Field to split out: `emails`  
    - Inputs: Items with ‘emails’ array  
    - Outputs: Individual email items to “Remove Duplicates (2)”  
    - Edge Cases: Empty arrays filtered previously; no major issues  

  - **Remove Duplicates (2)**  
    - Type: Remove Duplicates  
    - Role: Removes duplicate email addresses from all extracted emails  
    - Configuration: Default deduplication  
    - Inputs: Individual email items  
    - Outputs: Unique emails to “Add to Sheet”  
    - Edge Cases: None significant  

  - **Add to Sheet**  
    - Type: Google Sheets  
    - Role: Appends the extracted unique emails to a specified Google Sheet  
    - Configuration:  
      - Operation: Append  
      - Sheet Name: `gid=0` (first sheet)  
      - Document ID: `"1fcijyZM1oU73i2xUbXYJ4j6RshmVEduOkCJji2SJP68"` (replace with your sheet)  
      - Columns: Maps ‘emails’ field to sheet column ‘emails’  
      - Credential: Google Sheets OAuth2 configured with authorized account  
    - Inputs: Unique email items  
    - Outputs: None  
    - Edge Cases: Google Sheets API quota limits, credential expiration, sheet permission errors  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                          |
|-------------------------|-------------------------|-------------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Entry point – manual start of workflow          | None                        | Scrape Google Maps          | Entry point — manually start the workflow via the Test button.                                     |
| Scrape Google Maps       | HTTP Request            | Fetch Google Maps search results HTML            | When clicking ‘Test workflow’ | Extract URLs                | Fetches the Google Maps search results page for 'calgary dentists'.                               |
| Extract URLs            | Code                    | Extract URLs from HTML response                   | Scrape Google Maps           | Filter Google URLs           | Extracts all URLs from the HTML response and returns one item per URL.                             |
| Filter Google URLs       | Filter                  | Remove Google-related or tracking URLs            | Extract URLs                 | Remove Duplicates            | Removes unwanted Google-related or tracking URLs.                                                 |
| Remove Duplicates        | Remove Duplicates        | Remove duplicate website URLs                      | Filter Google URLs           | Limit                       | Removes duplicate website URLs.                                                                   |
| Limit                   | Limit                   | Limit number of websites processed per run to 10 | Remove Duplicates            | Loop Over Items             | Limits the number of websites processed per run to 10.                                           |
| Loop Over Items          | Split In Batches        | Process websites one by one                        | Limit                       | Scrape Site, Wait1           | Processes websites one at a time (batch processing).                                             |
| Scrape Site             | HTTP Request            | Fetch website HTML without redirects              | Loop Over Items              | Wait                        | Fetches the website HTML (does not follow redirects).                                            |
| Wait                    | Wait                    | Short delay before parsing                         | Scrape Site                  | Extract Emails              | Adds a delay between requests to avoid server blocking.                                          |
| Extract Emails           | Code                    | Extract email addresses from website HTML         | Wait                        | Filter Out Empties           | Extracts email addresses from the website HTML.                                                  |
| Wait1                   | Wait                    | Delay between requests to avoid blocking           | Loop Over Items              | Filter Out Empties           | Adds a delay between requests to avoid server blocking.                                          |
| Filter Out Empties      | Filter                  | Pass through only items where emails exist         | Wait1, Extract Emails        | Split Out                   | Passes through only items where 'emails' exists and is not empty.                                |
| Split Out               | Split Out               | Split multiple emails into individual items        | Filter Out Empties           | Remove Duplicates (2)        | Splits multiple emails into separate items (one per row).                                        |
| Remove Duplicates (2)    | Remove Duplicates        | Remove duplicate emails before saving              | Split Out                   | Add to Sheet                | Removes duplicate emails before saving.                                                         |
| Add to Sheet            | Google Sheets           | Append extracted emails to Google Sheet            | Remove Duplicates (2)        | None                       | Appends the extracted emails into the specified Google Sheet.                                   |
| Sticky Note             | Sticky Note             | Documentation note for Step 1                       | None                        | None                       | ## 🗺️ STEP 1: Google Maps Data Extraction ... Replace search URL with your target location and business type |
| Sticky Note1            | Sticky Note             | Documentation note for Step 2                       | None                        | None                       | ## 🔗 STEP 2: Website URL Processing ... Clean list of actual business websites ready for email extraction |
| Sticky Note2            | Sticky Note             | Documentation note for Step 3                       | None                        | None                       | ## 🔄 STEP 3: Smart Website Scraping ... Batching and delays are essential for reliable operation at scale |
| Sticky Note3            | Sticky Note             | Documentation note for Step 4                       | None                        | None                       | ## 📧 STEP 4: Email Extraction & Export ... Organized database of business emails ready for outreach |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Entry point to manually start the workflow.

2. **Add HTTP Request node**  
   - Name: `Scrape Google Maps`  
   - URL: `https://www.google.com/maps/search/calgary+dentists` (modify for target query)  
   - Options: Allow unauthorized SSL certificates; enable full response output  
   - Connect from manual trigger.

3. **Add Code node**  
   - Name: `Extract URLs`  
   - JavaScript code:  
     ```javascript
     const input = $input.first().json.data;
     const regex = /https?:\/\/[^\/\s"'>]+/g;
     const websites = input?.match?.(regex) || [];
     return websites.map(website => ({ json: { website } }));
     ```  
   - Connect from `Scrape Google Maps`.

4. **Add Filter node**  
   - Name: `Filter Google URLs`  
   - Conditions (all must be true – URLs must NOT contain):  
     - `schema`  
     - `google`  
     - `gg`  
     - `gstatic`  
   - Connect from `Extract URLs`.

5. **Add Remove Duplicates node**  
   - Name: `Remove Duplicates`  
   - Default configuration to remove duplicate URLs  
   - Connect from `Filter Google URLs`.

6. **Add Limit node**  
   - Name: `Limit`  
   - Set `maxItems` to `10` to control batch size  
   - Connect from `Remove Duplicates`.

7. **Add Split In Batches node**  
   - Name: `Loop Over Items`  
   - Default batch size: 1 (process one website at a time)  
   - Connect from `Limit`.

8. **Add HTTP Request node**  
   - Name: `Scrape Site`  
   - URL: `={{ $json.website }}` (dynamic)  
   - Disable redirect following  
   - On error: continue workflow  
   - Connect from `Loop Over Items`.

9. **Add Wait node**  
   - Name: `Wait`  
   - Set wait time to 1 second  
   - Connect from `Scrape Site`.

10. **Add Code node**  
    - Name: `Extract Emails`  
    - JavaScript code:  
      ```javascript
      const input = $input.first().json.data;
      const regex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g;
      const emails = input?.match?.(regex) || [];
      return { json: { emails } };
      ```  
    - On error: continue workflow  
    - Connect from `Wait`.

11. **Add Wait node**  
    - Name: `Wait1`  
    - Use default or set a delay to avoid server blocking (recommended to add a few seconds)  
    - Connect from `Loop Over Items` (second output path).

12. **Add Filter node**  
    - Name: `Filter Out Empties`  
    - Condition: Check if `emails` array exists and is not empty  
    - Connect from `Wait1` and `Extract Emails` (merge both if needed).

13. **Add Split Out node**  
    - Name: `Split Out`  
    - Field to split: `emails`  
    - Connect from `Filter Out Empties`.

14. **Add Remove Duplicates node**  
    - Name: `Remove Duplicates (2)`  
    - Default configuration to remove duplicate emails  
    - Connect from `Split Out`.

15. **Add Google Sheets node**  
    - Name: `Add to Sheet`  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: `gid=0` (first sheet)  
    - Columns: Map `emails` field to sheet column named `emails`  
    - Credentials: Configure Google Sheets OAuth2 credentials  
    - Connect from `Remove Duplicates (2)`.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Replace the Google Maps search URL with your own query to target different locations or business categories.  | Workflow setup step 1, node `Scrape Google Maps`                                                      |
| Use batching and wait delays to avoid IP blocking or rate limiting from websites or Google Maps.               | Critical for scalability and reliability, described in Step 3 sticky note                              |
| The email regex excludes common image file extensions to reduce false positives from URLs ending with image extensions. | Node `Extract Emails`                                                                                   |
| Google Sheets node requires OAuth2 credentials with edit permissions on the target spreadsheet.                | Node `Add to Sheet`                                                                                     |
| Workflow is designed for manual trigger but can be automated by replacing the manual trigger with schedule or webhook triggers. | Node `When clicking ‘Test workflow’`                                                                   |
| Sticky Notes in the workflow provide detailed step descriptions and best practices.                            | Included inside workflow for user reference                                                           |
| Google may block scraping or require CAPTCHA solving; consider using proxies or official APIs for large-scale use. | General scraping consideration                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.