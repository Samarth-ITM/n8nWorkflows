Extract Business Emails from Google Maps Listings for Lead Generation

https://n8nworkflows.xyz/workflows/extract-business-emails-from-google-maps-listings-for-lead-generation-10074


# Extract Business Emails from Google Maps Listings for Lead Generation

### 1. Workflow Overview

This workflow is designed to automate the extraction of business email addresses from Google Maps listings for lead generation purposes. It scrapes a Google Maps search results page for local business websites, filters and cleans the URLs, visits each business website to extract email addresses, and finally stores the collected emails in a Google Sheet for further use.

The workflow is organized into the following logical blocks:

- **1.1 Manual Trigger**  
  Entry point to manually start the workflow.

- **1.2 Google Maps Scraping and URL Extraction**  
  Retrieves the Google Maps search results HTML and extracts all website URLs.

- **1.3 URL Filtering and Deduplication**  
  Removes Google/tracking URLs, eliminates duplicates, and limits the number of URLs processed to prevent overloading.

- **1.4 Website Scraping and Email Extraction**  
  Visits each filtered website, scrapes the HTML content, extracts email addresses, and prepares them for export.

- **1.5 Email Deduplication and Storage**  
  Splits multiple emails into individual entries, removes duplicates, and appends the results into a specified Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger

- **Overview:**  
  Starts the workflow manually via the “Test workflow” button in n8n.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Sticky Note (manual trigger explanation)

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: No parameters; it triggers the workflow manually.  
    - Inputs: None  
    - Outputs: Triggers the next node “Scrape Google Maps”  
    - Potential Failures: None typical, except if workflow is not activated.  
    - Notes: Entry point for manual runs.

  - **Sticky Note**  
    - Purpose: Explains the usage of manual trigger.  
    - Content: “Starts the workflow when you click ‘Test Workflow’. Use this to manually run the scraper whenever you need fresh results.”

---

#### 2.2 Google Maps Scraping and URL Extraction

- **Overview:**  
  Fetches the Google Maps search results page for a specified query and extracts all URLs from the HTML content.

- **Nodes Involved:**  
  - Scrape Google Maps  
  - Extract URLs  
  - Sticky Note1 (block explanation)

- **Node Details:**  
  - **Scrape Google Maps**  
    - Type: HTTP Request  
    - Configuration:  
      - URL hardcoded to a Google Maps search for "carpinteros en Tarragona" (Spanish for "carpenters in Tarragona")  
      - Response: Full HTTP response with body included  
      - Allows unauthorized SSL certificates (for potential local testing or self-signed certs)  
    - Inputs: Trigger from manual start  
    - Outputs: Raw HTML response data  
    - Failure Modes: HTTP errors, Google blocking requests, captchas, network timeouts

  - **Extract URLs**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Uses regex to extract all URLs matching typical HTTP/HTTPS URL patterns from the HTML content  
      - Returns one item per extracted URL  
    - Inputs: HTML data from “Scrape Google Maps”  
    - Outputs: List of extracted URLs as separate items  
    - Failure Modes: Regex might miss malformed URLs or extract unwanted strings

  - **Sticky Note1**  
    - Explains that this block fetches the HTML and extracts website links.

---

#### 2.3 URL Filtering and Deduplication

- **Overview:**  
  Cleans the URL list by removing Google-related or tracking URLs, removes duplicates, and limits the number of URLs to process to 100 to ensure performance and avoid rate limiting.

- **Nodes Involved:**  
  - Filter Google URLs  
  - Remove Duplicates  
  - Limit  
  - Sticky Note2 (block explanation)

- **Node Details:**  
  - **Filter Google URLs**  
    - Type: Filter  
    - Configuration:  
      - Removes URLs containing substrings related to Google infrastructure and tracking domains such as "schema", "google", "gg", "gstatic", and "sentry" domains  
      - Uses string "notContains" operations with strict case-sensitive matching  
    - Inputs: URLs from “Extract URLs”  
    - Outputs: Filtered URLs  
    - Failure Modes: Overly strict filtering might remove legitimate URLs; under-filtering might retain unwanted URLs

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Configuration:  
      - Removes duplicate URLs from the filtered list  
    - Inputs: Filtered URLs  
    - Outputs: Unique URLs  
    - Failure Modes: None typical

  - **Limit**  
    - Type: Limit  
    - Configuration:  
      - Limits output to a maximum of 100 URLs per run  
    - Inputs: Unique URLs  
    - Outputs: Limited list of URLs  
    - Failure Modes: Limits may truncate desired URLs if more than 100 are relevant

  - **Sticky Note2**  
    - Describes the purpose of this block: removing unwanted URLs, deduplicating, and limiting total results to 100.

---

#### 2.4 Website Scraping and Email Extraction

- **Overview:**  
  Processes each website URL one by one with batching, waits between requests to avoid detection or blocking, fetches website HTML without following redirects, extracts email addresses using regex, and filters out empty results.

- **Nodes Involved:**  
  - Loop Over Items  
  - Wait1  
  - Scrape Site  
  - Wait  
  - Extract Emails  
  - Filter Out Empties  
  - Sticky Note3 (block explanation)

- **Node Details:**  
  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration:  
      - Processes one URL at a time (default batch size 1) to avoid overloading target servers  
    - Inputs: Limited URLs from previous block  
    - Outputs: Single URL per execution cycle  
    - Failure Modes: If batch size changed, might cause rate limiting

  - **Wait1**  
    - Type: Wait  
    - Configuration: Default (no explicit duration noted, but implied delay)  
    - Inputs: URL from batch  
    - Outputs: Passes to next node after waiting  
    - Failure Modes: None typical

  - **Scrape Site**  
    - Type: HTTP Request  
    - Configuration:  
      - Requests the URL from current item  
      - Does not follow redirects (to prevent being taken to irrelevant pages)  
      - On error, continues workflow (prevents workflow halt on request failure)  
    - Inputs: URL from Wait1  
    - Outputs: HTML content of the business website or error  
    - Failure Modes: Network errors, redirects not followed, site protections (captcha, bot detection)

  - **Wait**  
    - Type: Wait  
    - Configuration: Waits 1 second before continuing to prevent rate limits or detection  
    - Inputs: Website HTML from Scrape Site  
    - Outputs: Passes HTML content forward  
    - Failure Modes: None typical

  - **Extract Emails**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Uses regex to find email addresses in the HTML content  
      - Excludes common image file extensions in matches to reduce false positives (e.g., .jpg, .png)  
      - Returns found emails as an array in a single item  
      - On error, continues without stopping workflow  
    - Inputs: HTML content from Wait  
    - Outputs: JSON object with "emails" array  
    - Failure Modes: Regex might miss obfuscated emails or catch invalid strings

  - **Filter Out Empties**  
    - Type: Filter  
    - Configuration:  
      - Passes only items where the "emails" field exists and is not empty  
    - Inputs: Emails array from Extract Emails  
    - Outputs: Items with at least one email  
    - Failure Modes: Potentially filters out items with empty or missing fields

  - **Sticky Note3**  
    - Clarifies this block visits each website, extracts visible emails, and prepares for export.

---

#### 2.5 Email Deduplication and Storage

- **Overview:**  
  Splits multiple emails into individual items, removes duplicate emails, and appends the unique emails into a configured Google Sheet.

- **Nodes Involved:**  
  - Split Out  
  - Remove Duplicates (2)  
  - Add to Sheet  
  - Sticky Note4 (block explanation)

- **Node Details:**  
  - **Split Out**  
    - Type: Split Out  
    - Configuration:  
      - Splits the emails array into individual items, one email per item  
      - Field to split: "emails"  
    - Inputs: Items with emails from Filter Out Empties  
    - Outputs: Individual email items  
    - Failure Modes: None typical

  - **Remove Duplicates (2)**  
    - Type: Remove Duplicates  
    - Configuration:  
      - Removes duplicate email addresses to ensure uniqueness before saving  
    - Inputs: Individual email items  
    - Outputs: Unique email items  
    - Failure Modes: None typical

  - **Add to Sheet**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append  
      - Sheet name: Default “gid=0” (first sheet)  
      - Document ID: Specific Google Sheet ID provided  
      - Columns: Uses predefined columns, though only relevant column for emails is assumed (other columns present but not mapped explicitly)  
      - Credentials: Google Sheets OAuth2 credential set up (named "Google Sheets account")  
    - Inputs: Unique email items  
    - Outputs: Confirmation of append operation  
    - Failure Modes: Google API errors, credential permission errors, rate limits

  - **Sticky Note4**  
    - Describes this block as saving all extracted emails directly into Google Sheet (optional step).

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                                  | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                  |
|--------------------------|---------------------|-------------------------------------------------|----------------------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual start entry point                          | None                             | Scrape Google Maps             | Starts the workflow when you click “Test Workflow”. Use this to manually run the scraper whenever you need fresh results. |
| Scrape Google Maps        | HTTP Request        | Fetches Google Maps search HTML                   | When clicking ‘Test workflow’    | Extract URLs                  | Fetches the HTML from your chosen Google Maps search, then extracts all website links from the page. |
| Extract URLs              | Code                | Extracts all URLs from HTML                        | Scrape Google Maps               | Filter Google URLs            |                                                                                             |
| Filter Google URLs        | Filter              | Removes Google/tracking URLs                       | Extract URLs                    | Remove Duplicates             | Removes unwanted Google/tracking links, keeps only unique websites, and limits total results to 100 for safe execution. |
| Remove Duplicates         | Remove Duplicates   | Removes duplicated URLs                            | Filter Google URLs              | Limit                        |                                                                                             |
| Limit                    | Limit               | Limits number of URLs to 100                       | Remove Duplicates               | Loop Over Items              |                                                                                             |
| Loop Over Items           | Split In Batches    | Processes websites one at a time                   | Limit                          | Wait1, Scrape Site           |                                                                                             |
| Wait1                    | Wait                | Delay before scraping to avoid blocking           | Loop Over Items                 | Filter Out Empties           |                                                                                             |
| Scrape Site               | HTTP Request        | Fetches website HTML without following redirects | Loop Over Items                 | Wait                        | Visits each website, extracts visible email addresses, and prepares them for export.        |
| Wait                     | Wait                | Short wait before parsing                          | Scrape Site                    | Extract Emails              |                                                                                             |
| Extract Emails            | Code                | Extracts email addresses via regex                 | Wait                           | Filter Out Empties           |                                                                                             |
| Filter Out Empties        | Filter              | Passes items where emails exist and are non-empty | Wait1                          | Split Out                   |                                                                                             |
| Split Out                 | Split Out           | Splits email arrays into individual items         | Filter Out Empties             | Remove Duplicates (2)        |                                                                                             |
| Remove Duplicates (2)     | Remove Duplicates   | Removes duplicate emails                            | Split Out                     | Add to Sheet                |                                                                                             |
| Add to Sheet              | Google Sheets       | Appends emails into Google Sheet                   | Remove Duplicates (2)          | None                        | Saves all extracted emails directly into your Google Sheet (optional step).                 |
| Sticky Note               | Sticky Note         | Manual Trigger explanation                         | None                          | None                        | Starts the workflow when you click “Test Workflow”. Use this to manually run the scraper whenever you need fresh results. |
| Sticky Note1              | Sticky Note         | Scrape Google Maps + Extract URLs explanation      | None                          | None                        | Fetches the HTML from your chosen Google Maps search, then extracts all website links from the page. |
| Sticky Note2              | Sticky Note         | Filter, Remove Duplicates, Limit explanation       | None                          | None                        | Removes unwanted Google/tracking links, keeps only unique websites, and limits total results to 100 for safe execution. |
| Sticky Note3              | Sticky Note         | Scrape Site + Extract Emails explanation           | None                          | None                        | Visits each website, extracts visible email addresses, and prepares them for export.        |
| Sticky Note4              | Sticky Note         | Add to Google Sheet explanation                     | None                          | None                        | Saves all extracted emails directly into your Google Sheet (optional step).                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Manual Trigger Node**  
   - Add **Manual Trigger** node named “When clicking ‘Test workflow’”.  
   - No special configuration needed. This is the entry point.

2. **HTTP Request: Scrape Google Maps**  
   - Add **HTTP Request** node named “Scrape Google Maps”.  
   - Set method to GET.  
   - Set URL to the Google Maps search page URL for your target query (e.g., https://www.google.com/maps/search/carpinteros+en+tarragona/@41.1402229,1.0832014,56063m/data=!3m1!1e3?entry=ttu&g_ep=EgoyMDI1MTAwMS4wIKXMDSoASAFQAw%3D%3D).  
   - Enable “Full Response” in options.  
   - Allow unauthorized SSL certificates if needed.  
   - Connect the output of the manual trigger to this node.

3. **Code Node: Extract URLs**  
   - Add a **Code** node named “Extract URLs”.  
   - Use JavaScript code to extract all http/https URLs from the HTML content using regex.  
   - Code snippet:  
     ```js
     const input = $input.first().json.data;
     const regex = /https?:\/\/[^\/\s"'>]+/g;
     const websites = input?.match?.(regex) || [];
     return websites.map(website => ({ json: { website } }));
     ```  
   - Connect “Scrape Google Maps” node output to this node.

4. **Filter Node: Filter Google URLs**  
   - Add a **Filter** node named “Filter Google URLs”.  
   - Add multiple conditions with “not contains” operators on the “website” field for strings: “schema”, “google”, “gg”, “gstatic”, “sentry.wixpress.com”, “sentry.io”, “sentry-next.wixpress.com”.  
   - Connect output of “Extract URLs” to this node.

5. **Remove Duplicates Node: Remove Duplicates**  
   - Add a **Remove Duplicates** node named “Remove Duplicates”.  
   - Configure to remove duplicates based on the “website” field.  
   - Connect “Filter Google URLs” output here.

6. **Limit Node: Limit**  
   - Add a **Limit** node named “Limit”.  
   - Set max items to 100 to restrict processing volume.  
   - Connect output of “Remove Duplicates” here.

7. **Split In Batches Node: Loop Over Items**  
   - Add **Split In Batches** node named “Loop Over Items”.  
   - Batch size set to 1 (default) to process URLs individually.  
   - Connect “Limit” output here.

8. **Wait Node: Wait1**  
   - Add **Wait** node named “Wait1”.  
   - No explicit delay configured (optional: add a short delay if needed).  
   - Connect primary output of “Loop Over Items” to this node.

9. **Filter Node: Filter Out Empties**  
   - Add **Filter** node named “Filter Out Empties”.  
   - Condition: Pass only if “emails” field exists and is not empty.  
   - Connect “Wait1” output to this node.

10. **HTTP Request Node: Scrape Site**  
    - Add **HTTP Request** node named “Scrape Site”.  
    - Set method to GET.  
    - Set URL to “={{ $json.website }}” to dynamically fetch each site.  
    - Disable “Follow Redirects”.  
    - Configure to continue on error (“onError”: “continueRegularOutput”).  
    - Connect secondary output of “Loop Over Items” to this node.

11. **Wait Node: Wait**  
    - Add **Wait** node named “Wait”.  
    - Set a 1-second delay.  
    - Connect output of “Scrape Site” to this node.

12. **Code Node: Extract Emails**  
    - Add **Code** node named “Extract Emails”.  
    - Use JavaScript regex to extract email addresses excluding common image extensions:  
      ```js
      const input = $input.first().json.data;
      const regex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g;
      const emails = input?.match?.(regex) || [];
      return { json: { emails } };
      ```  
    - Set “Always Output Data” to true.  
    - Configure to continue on error.  
    - Connect output of “Wait” to this node.

13. **Split Out Node: Split Out**  
    - Add **Split Out** node named “Split Out”.  
    - Set field to split: “emails”.  
    - Connect output of “Filter Out Empties” to this node.

14. **Remove Duplicates Node: Remove Duplicates (2)**  
    - Add **Remove Duplicates** node named “Remove Duplicates (2)”.  
    - Configure to remove duplicates by email value.  
    - Connect “Split Out” output here.

15. **Google Sheets Node: Add to Sheet**  
    - Add **Google Sheets** node named “Add to Sheet”.  
    - Set operation to “Append”.  
    - Configure to append to the desired Google Sheet and worksheet by specifying document ID and sheet name (e.g., gid=0).  
    - Map the email data appropriately to a column (ensure at least one column holds the email).  
    - Set up Google Sheets OAuth2 credentials for API access.  
    - Connect output of “Remove Duplicates (2)” here.

16. (Optional) Add Sticky Notes at relevant places as reminders or explanations.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                                            |
|----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger starts the workflow when you click “Test Workflow”. Use to run scraper manually. | Sticky Note covering “When clicking ‘Test workflow’” node.                                                                  |
| Fetches the HTML from your chosen Google Maps search, then extracts all website links.        | Sticky Note covering “Scrape Google Maps” and “Extract URLs” nodes.                                                          |
| Removes unwanted Google/tracking links, keeps only unique websites, and limits total results to 100 for safe execution. | Sticky Note covering “Filter Google URLs”, “Remove Duplicates”, and “Limit” nodes.                                           |
| Visits each website, extracts visible email addresses, and prepares them for export.          | Sticky Note covering “Scrape Site” and “Extract Emails” nodes.                                                               |
| Saves all extracted emails directly into your Google Sheet (optional step).                    | Sticky Note covering “Add to Sheet” node.                                                                                     |

---

This documentation provides a full structural and functional understanding of the “Extract Business Emails from Google Maps Listings for Lead Generation” workflow, enabling developers and AI agents to reproduce, modify, or troubleshoot it effectively.