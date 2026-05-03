Web Site Scraper for LLMs with Airtop

https://n8nworkflows.xyz/workflows/web-site-scraper-for-llms-with-airtop-4252


# Web Site Scraper for LLMs with Airtop

### 1. Workflow Overview

This workflow automates recursive web scraping targeting content extraction from websites using Airtop’s AI scraping capabilities combined with Google Sheets and Google Docs for data management and documentation. The primary use case is to start from a seed URL, scrape the page content, extract internal links matching specific criteria, and recursively scrape linked pages up to a defined depth. It is designed for content aggregation, research, or lead generation where multiple layers of linked pages need to be crawled automatically.

The workflow is logically grouped as follows:

- **1.1 Input Reception and Parameter Normalization:** Accepts input parameters via form submission or external workflow call, unifies them for consistent use.
- **1.2 Initialization and Setup:** Creates a Google Sheet to track URLs and a Google Doc to store scraped content.
- **1.3 Initial Webpage Scraping:** Scrapes the seed URL content, saves it to the document, and records the URL in the spreadsheet.
- **1.4 Recursive Scraping Loop:** Reads URLs from the sheet, extracts internal links, filters new links, appends them to the sheet, scrapes their content, updates the document, and flags processed URLs.
- **1.5 Control Flow and Termination:** Manages depth-based recursion, deciding whether to repeat the scraping cycle or terminate.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Normalization

- **Overview:**  
Receives user input either via a web form or from another workflow trigger and standardizes parameters for downstream nodes.

- **Nodes Involved:**  
  - On form submission  
  - When Executed by Another Workflow  
  - Unify params  
  - Sticky Note5 (Documentation)

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point for manual user input.  
    - Config: Form named "Website scraper" with three fields: Seed URL (required), Links must contain (required), Depth (number, required).  
    - Input: HTTP webhook trigger on form submission.  
    - Output: JSON with form field values.  
    - Edge cases: Missing required fields, invalid URL format, negative or zero depth values.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for programmatic triggering with workflow inputs.  
    - Config: Expects inputs named "Seed url", "Links must contain", "Depth".  
    - Input: Workflow call with parameters.  
    - Output: JSON with parameters.  
    - Edge cases: Missing or improperly typed parameters.

  - **Unify params**  
    - Type: Set  
    - Role: Normalizes input fields from either entry point into unified JSON keys: "Seed url", "Links must contain", "Depth".  
    - Config: Maps incoming JSON to consistent names and types.  
    - Input: From either entry node.  
    - Output: Structured parameters for downstream use.

  - **Sticky Note5**  
    - Content: "## Input Parameters\nRun this workflow using a form or from another workflow"

#### 2.2 Initialization and Setup

- **Overview:**  
Creates a Google Sheets spreadsheet to track URLs and a Google Docs document to store scraped content.

- **Nodes Involved:**  
  - Create Spreadsheet  
  - Info to upload into spreadsheet  
  - Load info to spreadsheet  
  - Create Google Docs  
  - Sticky Note (Create Spreadsheet)  
  - Sticky Note2 (Create Doc)

- **Node Details:**  
  - **Create Spreadsheet**  
    - Type: Google Sheets (resource: spreadsheet)  
    - Role: Creates a new spreadsheet titled dynamically using the seed URL and depth.  
    - Config: Title format `Site map - [Seed url] (Depth - [Depth])`.  
    - Output: Spreadsheet metadata including spreadsheetId and sheet title.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: API quota exceeded, permission errors.

  - **Info to upload into spreadsheet**  
    - Type: Set  
    - Role: Prepares the initial data row with the Seed URL and an empty "Scraped" flag.  
    - Config: Sets fields "URL" to seed URL, "Scraped" to empty string.  
    - Input: Unified parameters.  
    - Output: JSON for appending to sheet.

  - **Load info to spreadsheet**  
    - Type: Google Sheets (operation: append)  
    - Role: Appends the initial seed URL record to the newly created spreadsheet.  
    - Config: Auto-mapped columns, targets sheet name and document ID from "Create Spreadsheet".  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Duplicate entries, API errors.

  - **Create Google Docs**  
    - Type: Google Docs  
    - Role: Creates a new Google Doc titled `Site to File - [Seed url]` to store scraped content.  
    - Config: Default folder, dynamic title.  
    - Credentials: Google Docs OAuth2.  
    - Edge cases: API rate limits, permission issues.

  - **Sticky Notes**  
    - "## Create Spreadsheet" and "## Create Doc" provide visual documentation.

#### 2.3 Initial Webpage Scraping

- **Overview:**  
Scrapes the seed URL content using Airtop and writes the initial scraped content to the Google Doc.

- **Nodes Involved:**  
  - Scrape webpage  
  - Write scraped content  
  - Sticky Note1 (Scrape webpage)  
  - Sticky Note2 (Create Doc)

- **Node Details:**  
  - **Scrape webpage**  
    - Type: Airtop Node (extraction/scrape)  
    - Role: Uses Airtop's AI extraction to scrape the webpage content at the seed URL.  
    - Config: URL from "Info to upload into spreadsheet". New session mode.  
    - Credentials: Airtop API key.  
    - Output: JSON containing scraped content text and metadata.  
    - Edge cases: Invalid URLs, network errors, scraping limits, session timeouts.

  - **Write scraped content**  
    - Type: Google Docs (update)  
    - Role: Inserts the entire scraped content text into the created Google Doc with a header indicating depth.  
    - Config: Inserts text with header: "The entire content from [URL] up to [Depth] levels deep" followed by scraped text. Uses document URL from "Create Google Docs".  
    - Credentials: Google Docs OAuth2.  
    - Edge cases: Document access issues, text insertion failures.

  - Sticky notes reinforce the scraping and document creation steps visually.

#### 2.4 Recursive Scraping Loop

- **Overview:**  
Recursively reads URLs from the spreadsheet, uses Airtop to extract internal links, filters and deduplicates new links, appends them to the spreadsheet, scrapes their content, updates the Google Doc, and flags each URL as scraped. This loop continues until the specified depth is reached.

- **Nodes Involved:**  
  - Should scrape more? (If)  
  - Read scraped webpages  
  - Retrieve links to scrape  
  - Filter links to insert to Sheets (Code)  
  - Insert new links  
  - Scrape webpage1  
  - Update with new scraped content  
  - Flag scraped link  
  - Insert flag  
  - Sticky Note3 (Recursive Scraping Process)

- **Node Details:**  
  - **Should scrape more?**  
    - Type: If  
    - Role: Controls recursion based on current run index vs Depth parameter.  
    - Config: Conditions check if current run index is less than Depth - 1 and Depth > 1.  
    - Output: True branch continues recursion; false branch ends it.  
    - Edge cases: Depth=1 disables recursion; invalid Depth values.

  - **Read scraped webpages**  
    - Type: Google Sheets (read)  
    - Role: Reads all URLs from the spreadsheet for processing.  
    - Config: Uses document ID and sheet name from "Create Spreadsheet".  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Empty sheets, read errors.

  - **Retrieve links to scrape**  
    - Type: Airtop (extraction/query)  
    - Role: Extracts internal links from each URL’s webpage content by querying Airtop with a prompt.  
    - Config: URL from sheet; prompt requests extraction of internal domain links; output schema enforces an array of strings under "internal_links". New session mode.  
    - Credentials: Airtop API key.  
    - Edge cases: Schema mismatch, no links found, API errors.

  - **Filter links to insert to Sheets**  
    - Type: Code (JavaScript)  
    - Role: Filters the extracted links to only those containing the user-defined substring; deduplicates against existing sheet URLs and within the new links.  
    - Key expressions:  
      - Uses "Links must contain" parameter.  
      - Compares against URLs already in the sheet.  
      - Removes URL query parameters for cleanliness.  
    - Output: List of new unique links to append.  
    - Edge cases: Empty "Links must contain" means no filtering; malformed URLs; deduplication logic errors.

  - **Insert new links**  
    - Type: Google Sheets (append)  
    - Role: Appends filtered new links to the spreadsheet, initializing "Scraped" flag empty.  
    - Config: Maps "URL" column with new links.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Duplicate inserts, API limits.

  - **Scrape webpage1**  
    - Type: Airtop (extraction/scrape)  
    - Role: Scrapes the content of each newly inserted link.  
    - Config: URL from new links; new session mode.  
    - Credentials: Airtop API key.  
    - Edge cases: Scraping failures or invalid URLs.

  - **Update with new scraped content**  
    - Type: Google Docs (update)  
    - Role: Inserts scraped content from each link into the Google Doc with separators for clarity.  
    - Config: Inserts text with delimiters before and after scraped text.  
    - Credentials: Google Docs OAuth2.  
    - Edge cases: Document update errors.

  - **Flag scraped link**  
    - Type: Set  
    - Role: Marks each processed URL as scraped with the current run index.  
    - Config: Sets "Scraped" field to current run index for the URL.  
    - Edge cases: Synchronization or concurrency issues.

  - **Insert flag**  
    - Type: Google Sheets (update)  
    - Role: Updates the "Scraped" flag in the spreadsheet for the processed URLs.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Update conflicts.

  - **Sticky Note3** explains the recursive scraping logic visually.

#### 2.5 Control Flow and Termination

- **Overview:**  
Manages workflow execution based on depth and recursive iteration count, deciding whether to continue scraping or terminate.

- **Nodes Involved:**  
  - Should scrape more? (revisited in control flow)  
  - Connections to loop or end the workflow.

- **Node Details:**  
  - The "Should scrape more?" node’s true branch leads to reading sheet and continuing scraping; false branch ends the recursion.  
  - Edge cases: Infinite loops prevented by depth logic; premature termination if depth or input is misconfigured.

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                               | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                              |
|----------------------------|-------------------------------|----------------------------------------------|-------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                  | Receives user input via form                   | —                             | Unify params                 | ## Input Parameters<br>Run this workflow using a form or from another workflow                          |
| When Executed by Another Workflow | Execute Workflow Trigger      | Receives inputs programmatically               | —                             | Unify params                 | ## Input Parameters<br>Run this workflow using a form or from another workflow                          |
| Unify params              | Set                           | Normalizes input parameters                    | On form submission, When Executed by Another Workflow | Create Spreadsheet            | ## Input Parameters<br>Run this workflow using a form or from another workflow                          |
| Create Spreadsheet         | Google Sheets (spreadsheet)   | Creates spreadsheet to track URLs              | Unify params                  | Info to upload into spreadsheet | ## Create Spreadsheet                                                                                   |
| Info to upload into spreadsheet | Set                           | Prepares initial URL data for sheet            | Create Spreadsheet            | Load info to spreadsheet     | ## Create Spreadsheet                                                                                   |
| Load info to spreadsheet   | Google Sheets (append)        | Inserts initial seed URL entry                  | Info to upload into spreadsheet | Scrape webpage               | ## Create Spreadsheet                                                                                   |
| Scrape webpage             | Airtop (scrape)               | Scrapes seed URL content                        | Load info to spreadsheet      | Create Google Docs           | ## Scrape webpage                                                                                       |
| Create Google Docs          | Google Docs                   | Creates document to store scraped content      | Scrape webpage                | Write scraped content        | ## Create Doc                                                                                           |
| Write scraped content       | Google Docs (update)          | Inserts scraped content into document          | Create Google Docs            | Should scrape more?          | ## Create Doc                                                                                           |
| Should scrape more?         | If                           | Controls recursive scraping based on depth     | Write scraped content, Flag scraped link | Read scraped webpages, (end flow) | ## Recursive Scraping Process                                                                            |
| Read scraped webpages       | Google Sheets (read)          | Reads URLs from spreadsheet                      | Should scrape more?           | Retrieve links to scrape     | ## Recursive Scraping Process                                                                            |
| Retrieve links to scrape    | Airtop (query)                | Extracts internal links from webpage            | Read scraped webpages         | Filter links to insert to Sheets | ## Recursive Scraping Process                                                                            |
| Filter links to insert to Sheets | Code                          | Filters and deduplicates new links               | Retrieve links to scrape      | Insert new links             | ## Recursive Scraping Process                                                                            |
| Insert new links            | Google Sheets (append)        | Appends filtered new links to spreadsheet       | Filter links to insert to Sheets | Scrape webpage1             | ## Recursive Scraping Process                                                                            |
| Scrape webpage1             | Airtop (scrape)               | Scrapes content of new links                     | Insert new links              | Update with new scraped content | ## Recursive Scraping Process                                                                            |
| Update with new scraped content | Google Docs (update)          | Updates document with new scraped content        | Scrape webpage1               | Flag scraped link            | ## Recursive Scraping Process                                                                            |
| Flag scraped link           | Set                           | Marks URL as scraped                              | Update with new scraped content | Insert flag                 | ## Recursive Scraping Process                                                                            |
| Insert flag                 | Google Sheets (update)        | Updates scraped flag in spreadsheet               | Flag scraped link             | Should scrape more?          | ## Recursive Scraping Process                                                                            |
| Sticky Note                 | Sticky Note                   | Documentation for spreadsheet creation           | —                             | —                            | ## Create Spreadsheet                                                                                   |
| Sticky Note1                | Sticky Note                   | Documentation for scraping step                   | —                             | —                            | ## Scrape webpage                                                                                       |
| Sticky Note2                | Sticky Note                   | Documentation for Google Docs creation             | —                             | —                            | ## Create Doc                                                                                           |
| Sticky Note3                | Sticky Note                   | Documentation for recursive scraping process       | —                             | —                            | ## Recursive Scraping Process                                                                            |
| Sticky Note4                | Sticky Note                   | README with detailed use case, setup, and next steps | —                             | —                            | README<br># Recursive Web Scraping ... [full detailed content]                                         |
| Sticky Note5                | Sticky Note                   | Documentation about input parameter methods        | —                             | —                            | ## Input Parameters<br>Run this workflow using a form or from another workflow                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Trigger Nodes:**

   - Add a **Form Trigger** node named "On form submission":
     - Form title: "Website scraper"
     - Fields (all required):  
       - Seed url (string)  
       - Links must contain (string)  
       - Depth (number)
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow":
     - Parameters: inputs named "Seed url", "Links must contain", "Depth" (number).

2. **Add a Set Node "Unify params":**

   - Purpose: Normalize input fields from either trigger.
   - Map inputs into three fields:  
     - Seed url (string) = `{{$json["Seed url"]}}`  
     - Links must contain (string) = `{{$json["Links must contain"]}}`  
     - Depth (number) = `{{$json.Depth}}`

3. **Add a Google Sheets Node "Create Spreadsheet":**

   - Resource: Spreadsheet  
   - Operation: Create new spreadsheet  
   - Title: `Site map - {{$json["Seed url"]}} (Depth - {{$json.Depth}})`  
   - Credentials: Configure Google Sheets OAuth2 credentials.

4. **Add a Set Node "Info to upload into spreadsheet":**

   - Assignments:  
     - URL = `={{ $('Unify params').item.json["Seed url"] }}`  
     - Scraped = `""` (empty string)

5. **Add a Google Sheets Node "Load info to spreadsheet":**

   - Operation: Append  
   - Document ID: `={{ $('Create Spreadsheet').item.json.spreadsheetId }}`  
   - Sheet name: `={{ $('Create Spreadsheet').item.json.sheets[0].properties.title }}`  
   - Columns: Auto-map input data, at least "URL" column.  
   - Credentials: Google Sheets OAuth2.

6. **Add an Airtop Node "Scrape webpage":**

   - Operation: Scrape  
   - URL: `={{ $json.URL }}` (from "Info to upload into spreadsheet")  
   - Session Mode: New  
   - Credentials: Airtop API key.

7. **Add a Google Docs Node "Create Google Docs":**

   - Operation: Create document  
   - Title: `Site to File - {{$('Unify params').item.json["Seed url"]}}`  
   - Folder ID: Default  
   - Credentials: Google Docs OAuth2.

8. **Add a Google Docs Node "Write scraped content":**

   - Operation: Update document (insert text)  
   - Document URL: `={{ $json.id }}` (from "Create Google Docs")  
   - Insert text:  
     ```
     The entire content from {{ $('Info to upload into spreadsheet').item.json.URL }} up to {{ $('Unify params').item.json.Depth }} levels deep.
     ---------------------------------------------
     {{ $('Scrape webpage').item.json.data.modelResponse.scrapedContent.text }}
     ---------------------------------------------
     ```
   - Credentials: Google Docs OAuth2.

9. **Add an If Node "Should scrape more?":**

   - Conditions:  
     - `$runIndex` < `{{$('Unify params').first().json.Depth - 1}}`  
     - AND  
     - `Depth` > 1  
   - True branch continues recursion; false branch ends.

10. **Add a Google Sheets Node "Read scraped webpages":**

    - Operation: Read rows  
    - Document ID and Sheet name from "Create Spreadsheet".  
    - Credentials: Google Sheets OAuth2.

11. **Add an Airtop Node "Retrieve links to scrape":**

    - Operation: Query  
    - URL: `={{ $json.URL }}` (from read sheet rows)  
    - Prompt: Extract links leading to other pages in the same domain.  
    - Session Mode: New  
    - Output Schema: Enforce array of strings under "internal_links".  
    - Credentials: Airtop API key.

12. **Add a Code Node "Filter links to insert to Sheets":**

    - JavaScript logic:  
      - Parse Airtop response for links.  
      - Filter links containing "Links must contain" parameter (if non-empty).  
      - Remove duplicates between model response and existing sheet URLs.  
      - Output deduplicated list of new links.

13. **Add a Google Sheets Node "Insert new links":**

    - Operation: Append  
    - Columns: Map "URL" from filtered links.  
    - Document ID and Sheet name from "Create Spreadsheet".  
    - Credentials: Google Sheets OAuth2.

14. **Add an Airtop Node "Scrape webpage1":**

    - Operation: Scrape  
    - URL: `={{ $json.URL }}` (newly inserted links)  
    - Session Mode: New  
    - Credentials: Airtop API key.

15. **Add a Google Docs Node "Update with new scraped content":**

    - Operation: Update (insert text)  
    - Document URL: from "Create Google Docs"  
    - Text to insert:  
      ```
      -------------------------
      {{ $json.data.modelResponse.scrapedContent.text }}
      -------------------------
      ```
    - Credentials: Google Docs OAuth2.

16. **Add a Set Node "Flag scraped link":**

    - Assignments:  
      - URL = `={{ $('Insert new links').item.json.URL }}`  
      - Scraped = `={{ $runIndex }}`

17. **Add a Google Sheets Node "Insert flag":**

    - Operation: Update rows  
    - Columns: Map URL and updated "Scraped" flag.  
    - Document ID and Sheet name from "Create Spreadsheet".  
    - Credentials: Google Sheets OAuth2.

18. **Connect the nodes in this order:**

    - Entry nodes → Unify params → Create Spreadsheet → Info to upload into spreadsheet → Load info to spreadsheet → Scrape webpage → Create Google Docs → Write scraped content → Should scrape more?  
    - True branch: Should scrape more? → Read scraped webpages → Retrieve links to scrape → Filter links to insert to Sheets → Insert new links → Scrape webpage1 → Update with new scraped content → Flag scraped link → Insert flag → Should scrape more? (loop)  
    - False branch: Should scrape more? → End workflow.

19. **Add Sticky Notes as per original workflow for documentation at relevant locations.**

20. **Set workflow settings:** Execution order as v1, activate workflow when ready.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| README describing recursive web scraping automation, use cases, input parameters, setup requirements, and next steps. Highlights the need for Airtop API key, Google Docs and Sheets credentials, and suggests improvements like filtering rules, scheduler integration, and exporting structured data.                                                                                                                                                                                                                                                                                                                                                                                                                | Embedded in Sticky Note4 in workflow.                                                              |
| Airtop API key is required for scraping and querying. API keys are free to generate at https://portal.airtop.ai/api-keys                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | https://portal.airtop.ai/api-keys                                                                    |
| Google Docs and Sheets OAuth2 credentials must be created via Google Cloud Console, following n8n documentation instructions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | https://docs.n8n.io/integrations/builtin/credentials/google/                                        |
| The workflow is designed to be manually triggered through a form or programmatically invoked from other workflows, providing flexible integration options.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Sticky Note5                                                                                        |
| The recursive scraping depth parameter controls the number of iterations beyond the initial seed URL scraping, preventing infinite loops.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Workflow logic                                                                                      |
| The filtering of links to scrape is based on the "Links must contain" string, enabling domain or path based crawling restrictions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Filter links to insert to Sheets (Code node)                                                       |

---

**Disclaimer:** The provided content originates exclusively from an n8n automated workflow, adhering strictly to content policies, containing no illegal or protected elements. All manipulated data is legal and public.