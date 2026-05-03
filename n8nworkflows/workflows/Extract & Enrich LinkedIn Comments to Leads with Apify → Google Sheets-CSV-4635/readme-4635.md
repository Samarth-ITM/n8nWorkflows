Extract & Enrich LinkedIn Comments to Leads with Apify → Google Sheets/CSV

https://n8nworkflows.xyz/workflows/extract---enrich-linkedin-comments-to-leads-with-apify---google-sheets-csv-4635


# Extract & Enrich LinkedIn Comments to Leads with Apify → Google Sheets/CSV

---

### 1. Workflow Overview

This workflow automates the extraction and enrichment of leads from LinkedIn post comments using Apify scrapers, then exports the enriched data to Google Sheets or CSV format. It targets LinkedIn posts by their IDs or URLs, scrapes comments and engagement data without requiring LinkedIn login credentials, enriches commenters’ profiles with detailed information, and outputs the results for sales or marketing lead generation.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Initialization**: Accepts LinkedIn post IDs/URLs via a form or manual input, sets API tokens and limits.
- **1.2 Apify Comments Scraping**: Runs Apify’s LinkedIn Post Comments Scraper to fetch comments in paginated batches.
- **1.3 Comments Data Aggregation & Unique Lead Extraction**: Aggregates multiple scrape runs, deduplicates commenters to create a unique leads list.
- **1.4 Profile Enrichment**: Uses Apify’s LinkedIn Profile Batch Scraper to enrich lead profiles in batches.
- **1.5 Data Preparation for Export**: Flattens enriched data structures for spreadsheet compatibility.
- **1.6 Export to Google Sheets or CSV**: Creates a new Google Sheet and appends enriched leads data or optionally converts data to CSV.

Additional elements include user guidance notes, configuration hints, and error handling suggestions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

**Overview:**  
This block captures user inputs (LinkedIn post IDs or URLs, comment scrape limit) either from a web form or manual trigger, and sets necessary variables including the Apify API token.

**Nodes Involved:**  
- Trigger manually  
- On form submission  
- Set fields from the form  
- Set manual fields (disabled by default)  
- Set APIFY Token  
- Sticky Notes 1, 2, 3, 4, 10

**Node Details:**

- **Trigger manually**  
  - Type: Manual trigger node  
  - Role: Enables manual start of the workflow (disabled by default)  
  - Input: None  
  - Output: Triggers "Set manual fields" node  
  - Edge Cases: If enabled, it requires manual input of post IDs and limit in "Set manual fields"  

- **On form submission**  
  - Type: Form trigger node  
  - Role: Accepts LinkedIn post IDs/URLs and comment limit via form submission  
  - Configuration:  
    - Ignores bots, custom submit button label  
    - Responds with redirect to Google Sheets documentation URL upon completion  
    - Form fields: Textarea for post IDs/URLs (required), number input for limit (default 100)  
  - Input: HTTP webhook form data  
  - Output: Passes form data to "Set fields from the form"  
  - Edge Cases: Invalid or empty post IDs, malformed URLs, no limit specified  

- **Set fields from the form**  
  - Type: Set node  
  - Role: Parses and normalizes form input into array of post IDs and numeric limit  
  - Configuration:  
    - Processes input string by removing line breaks, splitting by commas, filtering out empty entries  
    - Sets default limit to 100 if not specified  
  - Input: Form submission JSON  
  - Output: JSON with `postIds` (array) and `limit` (number)  
  - Edge Cases: Empty or improperly formatted input results in empty array or default limit  

- **Set manual fields** (disabled)  
  - Type: Set node  
  - Role: Provides hardcoded post IDs and limit for manual workflow runs  
  - Configuration:  
    - Sample post URLs array  
    - Limit set to 100  
  - Input: Trigger manually  
  - Output: JSON with `postIds` and `limit`  
  - Edge Cases: Disabled by default; must be enabled for manual runs  

- **Set APIFY Token**  
  - Type: Set node  
  - Role: Assigns Apify API token, post IDs, and limit to workflow variables  
  - Configuration:  
    - `APIFY_TOKEN` string (to be filled by user)  
    - `postIds` passed as array  
    - `limit` passed as number  
  - Input: From either "Set fields from the form" or "Set manual fields"  
  - Output: JSON with token and parameters for API calls  
  - Edge Cases: Missing or invalid `APIFY_TOKEN` will cause API authentication failures  

- **Sticky Notes**  
  - Provide detailed instructions, prerequisites, and usage tips related to input methods, token setup, and workflow modes.

---

#### 1.2 Apify Comments Scraping

**Overview:**  
Executes the Apify LinkedIn Post Comments Scraper API to retrieve comments data. Handles pagination if the number of comments exceeds a single run’s limit.

**Nodes Involved:**  
- Run Apify Comments Scraper  
- More runs needed? (IF node)  
- Set pagination (Code node)  
- Split Out  
- Loop Over Comments (Split In Batches)  
- Run Apify Comments Scraper Loop  
- Aggregate  
- Aggregate Comments  
- Aggregate All Comments  
- Gather All Comments  
- Sticky Note (🛠 Running Apify LinkedIn Comments Scrapers and processing data)

**Node Details:**

- **Run Apify Comments Scraper**  
  - Type: HTTP Request  
  - Role: Calls Apify API to scrape comments for given post IDs, page 1 only  
  - Configuration:  
    - POST request with body parameters: postIds, page_number=1, sortOrder="most recent", limit from input  
    - URL constructed with Apify token  
  - Input: JSON with `APIFY_TOKEN`, `postIds`, `limit`  
  - Output: JSON dataset of comments and summary data  
  - Edge Cases: API errors, token invalid, rate limits, empty results  

- **More runs needed?** (IF node)  
  - Type: Conditional (IF) node  
  - Role: Checks if total available comments and requested limit exceed 100 (per run max)  
  - Condition: totalComments > 100 AND limit > 100  
  - Input: Output of "Aggregate" (summary.totalComments) and "Set APIFY Token" (limit)  
  - Outputs:  
    - True: Proceed to pagination to fetch additional pages  
    - False: Proceed to unique leads extraction  
  - Edge Cases: Incorrect summary data could cause logic errors  

- **Set pagination** (Code node)  
  - Type: Code (JavaScript)  
  - Role: Calculates how many additional pages of 100 comments to fetch based on limit and total available comments  
  - Output: Array of pagination objects with page numbers starting from 2  
  - Edge Cases: Division by zero or missing data might cause errors  

- **Split Out**  
  - Type: Split Out node  
  - Role: Splits pagination array into individual runs for looping calls  
  - Input: Pagination array from previous node  
  - Output: Individual page objects for API calls  

- **Loop Over Comments** (Split In Batches)  
  - Type: Split In Batches  
  - Role: Batches pagination calls to avoid overloading API or workflow  
  - Input: Individual page objects  
  - Output: Batched API calls for each page  

- **Run Apify Comments Scraper Loop**  
  - Type: HTTP Request  
  - Role: Calls Apify API for each paginated page to collect comments beyond the first page  
  - Configuration: Same as initial scraper but with dynamic page_number parameter  
  - Input: Pagination page number, post IDs, limit, API token  
  - Output: Comments dataset for each page  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates all comment data from multiple API calls into a single array  
  - Input: Multiple comment datasets  
  - Output: Unified comments array  
  - Edge Cases: Large data volumes might impact memory  

- **Aggregate Comments** and **Aggregate All Comments**  
  - Both aggregate all comment batches progressively, merging lists for comprehensive data  

- **Gather All Comments** (Set node)  
  - Combines aggregated comments into a single unified list for further processing  

- **Sticky Note (🛠 Running Apify LinkedIn Comments Scrapers and processing data)**  
  - Provides context on running scrapers once for validation and looping if needed  

---

#### 1.3 Comments Data Aggregation & Unique Lead Extraction

**Overview:**  
Processes the gathered comments to filter unique commenters (leads) based on their LinkedIn profile URLs, preparing for profile enrichment.

**Nodes Involved:**  
- Create Unique List of Leads (Code node)  
- Sticky Note (Create a unique list of Leads, ✨ enrich ✨ it with more data)  

**Node Details:**

- **Create Unique List of Leads**  
  - Type: Code (JavaScript)  
  - Role: Iterates over all comment items, extracts and deduplicates lead profiles by `profile_url`  
  - Logic: Uses array reduce to accumulate unique authors; ignores duplicates  
  - Input: Aggregated comments array  
  - Output: Array of unique author profiles for enrichment  
  - Edge Cases: Missing or malformed author data, empty input arrays, console logs present which might clutter logs  

- **Sticky Note**  
  - Explains the importance of creating a deduplicated lead list and next step of enrichment  

---

#### 1.4 Profile Enrichment

**Overview:**  
Enriches the unique lead profiles by calling Apify’s LinkedIn Profile Batch Scraper API in batches of up to 500 profiles.

**Nodes Involved:**  
- Split in batches (Code node)  
- Split Out Batches  
- Loop Over Profiles (Split In Batches)  
- Run Apify Profile Enrichment (HTTP Request)  
- Aggregate Profiles  
- Aggregate All Profiles  

**Node Details:**

- **Split in batches**  
  - Type: Code (JavaScript)  
  - Role: Splits the unique profile URLs into batches of 500 for API processing limits  
  - Output: JSON object with `batches` array, each containing up to 500 profile URLs  
  - Edge Cases: Profiles count less than 500 results in single batch  

- **Split Out Batches**  
  - Type: Split Out node  
  - Role: Emits each batch individually for sequential processing  

- **Loop Over Profiles** (Split In Batches)  
  - Type: Split In Batches  
  - Role: Processes batches in manageable chunks to avoid overload  

- **Run Apify Profile Enrichment**  
  - Type: HTTP Request  
  - Role: Calls Apify API with batch of profile URLs to retrieve enriched profile data  
  - Configuration:  
    - POST request with `usernames` parameter set to batch of profiles  
    - URL includes Apify token  
  - Input: Batch of profile URLs, Apify token  
  - Output: Enriched profile details dataset  
  - Edge Cases: API limits, token expiry, empty batches  

- **Aggregate Profiles** and **Aggregate All Profiles**  
  - Aggregate all enriched profile data from batch API calls into a single array for export  

---

#### 1.5 Data Preparation for Export

**Overview:**  
Flattens the enriched nested JSON data structures into a flat key-value format suitable for spreadsheet columns.

**Nodes Involved:**  
- Prepare the list for export (Code node)  

**Node Details:**

- **Prepare the list for export**  
  - Type: Code (JavaScript)  
  - Role: Recursively flattens nested objects into single-level keys with underscore concatenation  
  - Input: Aggregated enriched profile items  
  - Output: Array of flattened JSON objects  
  - Edge Cases: Deeply nested or missing fields handled via recursion; large datasets may impact performance  

---

#### 1.6 Export to Google Sheets or CSV

**Overview:**  
Creates a new Google Sheet document and appends the prepared lead data, or optionally converts the data to a CSV file for download.

**Nodes Involved:**  
- Create Google Sheet (Google Sheets node)  
- Add Leads (Google Sheets node)  
- Set the list for sheets (Code node)  
- Convert to File (Convert to file node, disabled by default)  
- Done (NoOp node)  
- Sticky Notes 6 and 7  

**Node Details:**

- **Create Google Sheet**  
  - Type: Google Sheets node  
  - Role: Creates a new spreadsheet titled with current date-time, with default "Leads" sheet  
  - Credentials: Google Sheets OAuth2 (configured separately)  
  - Output: Spreadsheet and sheet IDs for downstream use  
  - Edge Cases: OAuth failure, quota limits  

- **Add Leads**  
  - Type: Google Sheets node  
  - Role: Appends the flattened lead data to the created sheet, auto-mapping columns  
  - Configuration:  
    - Append mode, USER_ENTERED cell format  
    - Maps multiple profile and enrichment fields automatically to columns  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Data mismatch, schema changes, quota limits  

- **Set the list for sheets**  
  - Type: Code node  
  - Role: Passes the flattened array from preparation node to Google Sheets append node  
  - Edge Cases: Empty input array leads to no data appended  

- **Convert to File**  
  - Type: Convert to File (CSV) node, disabled by default  
  - Role: Allows exporting data as CSV file instead of Google Sheets  
  - Configuration: Includes header row  
  - Edge Cases: Large files might slow workflow, disabled to prevent conflict with Sheets export  

- **Done**  
  - Type: No Operation node  
  - Role: Marks workflow completion  

- **Sticky Notes 6 and 7**  
  - Provide instructions on manual mode, CSV export, and Google Sheets credentials setup  

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                          | Input Node(s)                    | Output Node(s)                        | Sticky Note                                                                                                                                                                                                                   |
|-------------------------------|-------------------------|----------------------------------------|---------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger manually              | Manual Trigger          | Manual workflow start                   | None                            | Set manual fields                   | _(or if run manually ENABLE this)_  \n### 4. Set Post ID/URL\n\n\n\n\n\n\n\n\n\n\n\n\n\n### _(CLICK ME)_ and Connect                                                                                                          |
| On form submission            | Form Trigger            | Receive LinkedIn post URLs and limit   | None                            | Set fields from the form            | ## 3. You can run through the form (check Form URL)\n\n\n\n\n\n\n\n\n\n\n\n\n\n### _(CLICK ME)_ \nIf run manually DISABLE this                                                                                              |
| Set fields from the form      | Set                     | Parse form input to array and number   | On form submission              | Set APIFY Token                    |                                                                                                                                                                                                                              |
| Set manual fields             | Set (disabled)          | Hardcoded post IDs and limit for manual run | Trigger manually               | None                              | _(or if run manually ENABLE this)_  \n### 4. Set Post ID/URL\n\n\n\n\n\n\n\n\n\n\n\n\n\n### _(CLICK ME)_ and Connect                                                                                                          |
| Set APIFY Token               | Set                     | Assign API token, post IDs, limit      | Set fields from the form / manual fields | Run Apify Comments Scraper         | ## 1. Set APIFY TOKEN\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n### _(CLICK ME)_                                                                                                                                                         |
| Run Apify Comments Scraper    | HTTP Request            | Scrape comments page 1                  | Set APIFY Token                 | Aggregate                         | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Aggregate                    | Aggregate               | Merge initial comment datasets          | Run Apify Comments Scraper      | More runs needed?                  | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| More runs needed?             | IF                      | Check if pagination needed               | Aggregate, Set APIFY Token      | Set pagination (true), Create Unique List of Leads (false) | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Set pagination               | Code                    | Calculate pagination pages               | More runs needed?               | Split Out                        | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Split Out                   | Split Out                | Emit each page for looping               | Set pagination                 | Loop Over Comments               | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Loop Over Comments           | Split In Batches        | Batch API calls for comment pages        | Split Out                     | Aggregate All Comments, Run Apify Comments Scraper Loop | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Run Apify Comments Scraper Loop | HTTP Request            | Scrape paginated comment pages          | Loop Over Comments             | Aggregate Comments              | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Aggregate Comments          | Aggregate               | Merge paginated comment datasets         | Run Apify Comments Scraper Loop | Loop Over Comments             | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Aggregate All Comments       | Aggregate               | Merge all collected comments             | Loop Over Comments             | Gather All Comments             | ## 🛠 Running Apify LinkedIn Comments Scrapers and processing data\nWe want to run an Apify scraper once to validate the number of available comments and then run it multiple times if necessary.                           |
| Gather All Comments          | Set                     | Combine all comments into one list        | Aggregate All Comments          | Create Unique List of Leads      | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Create Unique List of Leads  | Code                    | Deduplicate commenters by profile URL    | Gather All Comments             | Split in batches               | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Split in batches            | Code                    | Split unique profiles into batches of 500 | Create Unique List of Leads    | Split Out Batches              | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Split Out Batches            | Split Out                | Emit profile batches individually          | Split in batches               | Loop Over Profiles             | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Loop Over Profiles           | Split In Batches        | Batch process profile enrichment API calls | Split Out Batches              | Aggregate All Profiles, Run Apify Profile Enrichment | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Run Apify Profile Enrichment | HTTP Request            | Call Apify Profile Batch Scraper API       | Loop Over Profiles             | Aggregate Profiles             | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Aggregate Profiles          | Aggregate               | Merge enriched profile batch data          | Run Apify Profile Enrichment   | Loop Over Profiles             | ## Create a unique list of Leads, ✨ enrich ✨ it with more data\nOnce we have all of the comment authors, we need to enrich them with more information from LinkedIn                                                   |
| Aggregate All Profiles       | Aggregate               | Merge all enriched profiles                 | Loop Over Profiles             | Prepare the list for export    |                                                                                                                                                                                                                              |
| Prepare the list for export  | Code                    | Flatten nested JSON for spreadsheet export | Aggregate All Profiles         | Create Google Sheet, Convert to File |                                                                                                                                                                                                                              |
| Create Google Sheet          | Google Sheets           | Create new spreadsheet document             | Prepare the list for export    | Set the list for sheets        | ## 2. Add credentials to Google Sheets\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n### _(CLICK ME)_                                                                                                                             |
| Set the list for sheets      | Code                    | Pass flattened data to Google Sheets node   | Create Google Sheet            | Add Leads                     | ## You can either run it manually (not through form) and download the CSV\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\nYou need to activate "Trigger manual" nodes first\n\n### _(CLICK ME) and Connect_                        |
| Add Leads                   | Google Sheets           | Append lead data to spreadsheet              | Set the list for sheets        | Done                         |                                                                                                                                                                                                                              |
| Convert to File             | Convert To File         | Convert data to CSV (disabled by default)    | Prepare the list for export    | None                         | ## You can either run it manually (not through form) and download the CSV\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\nYou need to activate "Trigger manual" nodes first\n\n### _(CLICK ME) and Connect_                        |
| Done                        | No Operation            | Marks end of workflow                         | Add Leads                     | None                         |                                                                                                                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create two entry points:**  
   - Add a **Manual Trigger** node (disabled by default) for manual runs.  
   - Add a **Form Trigger** node configured with:  
     - Form title: "Linkedin Posts Comments Leads Scraper"  
     - Fields:  
       - Textarea labeled "Post IDs/URLs" (required)  
       - Number input labeled "How many comments you want to scrape?" (default 100)  
     - Button label: "Submit"  
     - Ignore bot submissions enabled  
     - Response: Redirect to Google Sheets documentation URL  

2. **Input Parsing:**  
   - Add a **Set** node ("Set fields from the form") connected from the Form Trigger:  
     - Assign `postIds` by splitting the textarea input on commas, removing whitespace and empty entries.  
     - Assign `limit` defaulting to 100 if empty.  
   - Add a **Set** node ("Set manual fields", disabled by default) connected from Manual Trigger:  
     - Hardcode an array of post URLs and a comment limit (e.g., 100).  

3. **Set API Token and parameters:**  
   - Add a **Set** node ("Set APIFY Token") connected from either input Set node:  
     - Add string variable `APIFY_TOKEN` with your Apify API token.  
     - Pass `postIds` and `limit` from input.  

4. **Initial Comment Scrape:**  
   - Add an **HTTP Request** node ("Run Apify Comments Scraper"):  
     - POST to `https://api.apify.com/v2/acts/apimaestro~linkedin-post-comments-replies-engagements-scraper-no-cookies/run-sync-get-dataset-items?token={{ $json.APIFY_TOKEN }}`  
     - Body parameters:  
       - `postIds`: array of post IDs  
       - `page_number`: 1  
       - `sortOrder`: "most recent"  
       - `limit`: numeric limit  
   - Connect from "Set APIFY Token"  

5. **Aggregate initial results:**  
   - Add an **Aggregate** node to merge items from the initial scrape.  

6. **Check if pagination needed:**  
   - Add an **IF** node ("More runs needed?") to evaluate:  
     - If total comments and limit both > 100, paginate; else proceed.  

7. **Pagination setup (if needed):**  
   - Add a **Code** node ("Set pagination") to calculate total pages based on limit and total comments.  
   - Add a **Split Out** node to emit each page as separate data.  
   - Add a **Split In Batches** node ("Loop Over Comments") to batch API calls.  
   - Add an **HTTP Request** node ("Run Apify Comments Scraper Loop") for paginated calls with dynamic `page_number`.  
   - Add **Aggregate** nodes to combine paginated results progressively.  

8. **Combine all comments:**  
   - Add a **Set** node ("Gather All Comments") to concatenate aggregated comment arrays.  

9. **Extract unique leads:**  
   - Add a **Code** node ("Create Unique List of Leads") that iterates over all comments and extracts unique commenter profiles by `profile_url`.  

10. **Batch unique profiles for enrichment:**  
    - Add a **Code** node ("Split in batches") to split profiles into chunks of 500.  
    - Add a **Split Out** node ("Split Out Batches") to emit each batch.  
    - Add a **Split In Batches** node ("Loop Over Profiles") to process batches sequentially.  

11. **Run Profile Enrichment:**  
    - Add an **HTTP Request** node ("Run Apify Profile Enrichment"):  
      - POST to `https://api.apify.com/v2/acts/apimaestro~linkedin-profile-batch-scraper-no-cookies-required/run-sync-get-dataset-items?token={{ $json.APIFY_TOKEN }}`  
      - Body parameter: `usernames` set to batch array.  
    - Add **Aggregate** nodes to merge enriched profile results.  

12. **Prepare data for export:**  
    - Add a **Code** node ("Prepare the list for export") to flatten nested JSON into a flat structure suitable for Google Sheets or CSV.  

13. **Export options:**  

    - **Google Sheets export:**  
      - Add a **Google Sheets** node ("Create Google Sheet"):  
        - Set spreadsheet title with current date-time pattern.  
        - Create a sheet named "Leads".  
        - Use OAuth2 credentials configured in n8n.  
      - Add a **Code** node ("Set the list for sheets") to pass flattened data.  
      - Add a **Google Sheets** node ("Add Leads") to append the data to the sheet with automatic column mapping.  
      - Connect nodes accordingly, ensuring "Add Leads" executes once and uses spreadsheet and sheet IDs from "Create Google Sheet".  

    - **CSV export (optional):**  
      - Add a **Convert to File** node configured to export CSV with header row (disabled by default).  
      - Connect the flatten data node to this node if CSV export is preferred.  
      - Disable Google Sheets nodes in this mode.  

14. **Workflow finalization:**  
    - Add a **No Operation** node ("Done") as workflow endpoint connected from "Add Leads".  

15. **Add Sticky Notes:**  
    - Add descriptive sticky notes at logical sections for user guidance, including prerequisites, API token setup, form usage, export options, and troubleshooting tips.  

16. **Credential Setup:**  
    - Configure Apify API token as a secured variable or directly in the "Set APIFY Token" node (recommended as credential).  
    - Configure Google Sheets OAuth2 credentials via n8n credentials manager.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                            |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Automate LinkedIn lead generation by scraping comments from targeted posts and enriching profiles without LinkedIn login. First 1,000 comments free with Apify. Export results to Google Sheets or CSV.                                                                                                                                               | Overview sticky note content                                                                                               |
| Apify Scrapers used: LinkedIn Post Comments Scraper (no cookies, $5/1,000 results), LinkedIn Profile Batch Scraper (no cookies, $5/1,000 results).                                                                                                                                                             | Apify scrapers section in sticky note                                                                                      |
| Prerequisites include Apify API token ([Get your token](https://apify.com/account#/integrations)) and Google Sheets OAuth2 credentials ([Google Sheets setup](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal)) | Sticky Note 10                                                                                                             |
| Free tier of Apify allows 1,000 scraped comments before incurring cost.                                                                                                                                                                                                                                           | Overview sticky note                                                                                                       |
| Pro Tips: Target high-quality posts, monitor Apify usage costs, maintain data hygiene, comply with LinkedIn terms of service. Troubleshoot common issues like authentication errors, empty results, or export failures. Contact [Saverflow.ai](https://saverflow.ai) for support or custom workflow development.         | Sticky Note 9                                                                                                              |
| For manual CSV exports, enable manual trigger node, disable form trigger and Google Sheets nodes, enable CSV node, then run workflow and download CSV file.                                                                                                                                                      | Sticky Note 6                                                                                                              |
| Google Sheets export requires proper OAuth2 credentials; ensure quota and API limits are respected.                                                                                                                                                                                                               | Sticky Note 7                                                                                                              |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. All data processed are legal and public. The workflow adheres to relevant content policies and does not contain illegal or protected elements.

---