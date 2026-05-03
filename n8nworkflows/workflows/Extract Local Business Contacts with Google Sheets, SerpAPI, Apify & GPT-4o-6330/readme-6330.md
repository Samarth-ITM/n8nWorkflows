Extract Local Business Contacts with Google Sheets, SerpAPI, Apify & GPT-4o

https://n8nworkflows.xyz/workflows/extract-local-business-contacts-with-google-sheets--serpapi--apify---gpt-4o-6330


# Extract Local Business Contacts with Google Sheets, SerpAPI, Apify & GPT-4o

---

### 1. Workflow Overview

This n8n workflow automates the extraction of local business contacts, including email addresses, from Google Maps search results and business websites, leveraging Google Sheets, SerpAPI, Apify, and GPT-4o AI. Its core purpose is to scale the process of finding local businesses based on search terms defined in a Google Sheet, scrape their contact details, extract emails using AI, and record the results back into a Google Sheet.

Logical blocks are structured as follows:

- **1.1 Input Reception:** Manual trigger to start the workflow and reading search terms from Google Sheets.
- **1.2 Search Term Filtering:** Filtering to process only search rows not marked as completed.
- **1.3 Google Maps Search:** Querying SerpAPI’s Google Maps integration to retrieve local business listings.
- **1.4 Data Formatting and Looping:** Cleaning and formatting the raw Google Maps data and iterating over each business item.
- **1.5 Website Scraping:** Using Apify’s Fast Website Content Crawler to scrape business websites for contact information.
- **1.6 Email Extraction with AI:** Employing GPT-4o-based LangChain agents to extract email addresses from scraped website content.
- **1.7 Data Saving and Marking Completion:** Writing the enriched contact data back to a Google Sheet and marking the search terms as completed.
- **1.8 Sub-Workflow Execution:** Invoking an external workflow for additional email extraction purposes (details abstracted).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually and reads search terms and related data from a Google Sheets spreadsheet.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Extract Search Terms (Google Sheets)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually on user command.  
    - Configuration: Default, no parameters set.  
    - Inputs: None  
    - Outputs: Triggers "Extract Search Terms" node.  
    - Edge cases: User forgetting to trigger; no input data if sheet is empty.

  - **Extract Search Terms**  
    - Type: Google Sheets  
    - Role: Reads search input data from a Google Sheet (sheet named "Searches").  
    - Configuration: Reads from a specific Google Sheet document (ID: "1nwgaTyp239ErzFS0IaDnJca-6LkIqvshmiHHs5bX77o") and sheet tab "gid=0". Uses OAuth2 credentials for authentication.  
    - Inputs: Trigger from manual node.  
    - Outputs: Outputs rows containing search terms and related columns such as "Search", "Area", "Area Name", and "Complete".  
    - Edge cases: Authentication failure, invalid sheet ID, no rows found.

#### 2.2 Search Term Filtering

- **Overview:** Filters out rows already marked as completed to avoid redundant processing, takes only the first unprocessed row for sequential processing.
- **Nodes Involved:**  
  - Keep Unprocessed Rows (Filter)  
  - Filter to first row (Code)  

- **Node Details:**

  - **Keep Unprocessed Rows**  
    - Type: Filter  
    - Role: Filters rows where the "Complete" column is empty (i.e., not marked "Yes").  
    - Configuration: Condition checks if "Complete" field is empty string.  
    - Inputs: Rows from "Extract Search Terms".  
    - Outputs: Only rows with incomplete status pass through.  
    - Edge cases: If all rows are marked complete, no output.

  - **Filter to first row**  
    - Type: Code  
    - Role: Reduces the filtered dataset to only the first row, enabling stepwise processing.  
    - Configuration: Runs JavaScript to return only the first item in the array.  
    - Inputs: Filtered rows.  
    - Outputs: Single row object for further processing.  
    - Edge cases: Empty input array, returns no items.

#### 2.3 Mark Row as Completed & Google Maps Search

- **Overview:** Marks the processed search row as completed in the Google Sheet, then queries Google Maps via SerpAPI for local business results.
- **Nodes Involved:**  
  - Mark Row as Completed (Google Sheets)  
  - Search Google Maps (SerpAPI)  

- **Node Details:**

  - **Mark Row as Completed**  
    - Type: Google Sheets  
    - Role: Updates or appends the row corresponding to the current search term, setting "Complete" to "Yes".  
    - Configuration: Matches rows by "Search" column and sets "Complete" = "Yes". Uses same sheet as input, "gid=0" tab.  
    - Inputs: Single row from "Filter to first row".  
    - Outputs: Passes the same row downstream for querying.  
    - Edge cases: Update failures, mismatched keys, API rate limits.

  - **Search Google Maps**  
    - Type: SerpAPI (Google Maps operation)  
    - Role: Queries Google Maps local results for the given search term and area coordinates.  
    - Configuration: Uses SerpAPI credentials and parameters: query = $json.Search, ll = $json.Area (latitude,longitude).  
    - Inputs: Row with search term and area.  
    - Outputs: Raw JSON of local results containing business info.  
    - Edge cases: API quota exceeded, network errors, invalid parameters.

#### 2.4 Format Output and Loop Over Items

- **Overview:** Cleans the Google Maps results for easier processing and iterates over each business entry for detailed scraping.
- **Nodes Involved:**  
  - Format output as table (Code)  
  - Loop Over Items (SplitInBatches)  

- **Node Details:**

  - **Format output as table**  
    - Type: Code  
    - Role: Parses the "local_results" array from SerpAPI output; cleans website URLs by removing tracking prefixes.  
    - Configuration: JavaScript code extracts `title`, `website`, `address`, and `phone` for each business.  
    - Inputs: Raw JSON from "Search Google Maps".  
    - Outputs: Array of cleaned business objects as separate items.  
    - Edge cases: Missing fields, unexpected data formats.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each business item individually through the workflow steps for scraping and extraction.  
    - Configuration: Default batch options, processes all items sequentially.  
    - Inputs: Cleaned array of business objects.  
    - Outputs: Each business item sent downstream one by one.  
    - Edge cases: Large batch causing timeouts or memory usage.

#### 2.5 Website Scraping

- **Overview:** Scrapes the website content of each business using Apify's Fast Website Content Crawler API to obtain raw text for email extraction.
- **Nodes Involved:**  
  - Scrape Web Page (HTTP Request)  

- **Node Details:**

  - **Scrape Web Page**  
    - Type: HTTP Request  
    - Role: Sends POST request to Apify API with the business website URL to scrape site content.  
    - Configuration: URL set to Apify crawler API endpoint; JSON body includes `startUrls` with the current business website. Uses generic HTTP query authentication with Apify API token in query string.  
    - Inputs: Individual business item (contains website URL).  
    - Outputs: Scraped text content from the website.  
    - Edge cases: Website inaccessible, Apify API quota exceeded, invalid or missing website URL.

#### 2.6 Email Extraction with AI

- **Overview:** Uses an AI agent powered by GPT-4o (LangChain agent node) to extract email addresses from the scraped website content.
- **Nodes Involved:**  
  - Extract Email - AI Agent (LangChain agent)  
  - Structured Output Parser (LangChain output parser)  

- **Node Details:**

  - **Extract Email - AI Agent**  
    - Type: LangChain Agent (AI Agent)  
    - Role: Processes scraped website text to find an email address or outputs null if none found.  
    - Configuration: System prompt instructs to extract email addresses from the input text. Output is parsed with a structured parser.  
    - Inputs: Website text from "Scrape Web Page".  
    - Outputs: JSON object containing extracted email or null.  
    - Edge cases: Ambiguous text, no emails present, AI model errors or rate limits.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI agent output to a defined JSON schema with an "email" field.  
    - Configuration: Uses simple JSON schema example with a single "email" string field.  
    - Inputs: AI agent raw output.  
    - Outputs: Structured JSON with email field.  
    - Edge cases: Parsing failures if AI output is not JSON-compliant.

#### 2.7 Save Emails to Sheet & Reduce to 1 Row

- **Overview:** Saves the enriched business contact info including extracted email to a Google Sheet and reduces items for sub-workflow execution.
- **Nodes Involved:**  
  - Save Emails to Sheet (Google Sheets)  
  - Reduce to 1 row (Summarize)  

- **Node Details:**

  - **Save Emails to Sheet**  
    - Type: Google Sheets  
    - Role: Appends or updates the "Results" sheet with detailed business data including area, phone, title, search term, address, website, area name, and the extracted email (manual entry email field filled by AI output).  
    - Configuration: Matches rows by "title" to avoid duplicates, writes to sheet ID "1nwgaTyp239ErzFS0IaDnJca-6LkIqvshmiHHs5bX77o", tab "1470668196" (Results). Uses OAuth2 credentials.  
    - Inputs: Business data enriched with AI-extracted emails.  
    - Outputs: Passes item downstream for summarization.  
    - Edge cases: Sheet API limits, mismatched keys, concurrent writes.

  - **Reduce to 1 row**  
    - Type: Summarize  
    - Role: Reduces multiple processed items into a single summary row, focusing on the "title" field, preparing for sub-workflow execution.  
    - Configuration: Summarizes "title" field, defaults for other parameters.  
    - Inputs: Items from "Loop Over Items".  
    - Outputs: Single summarized item.  
    - Edge cases: No items to summarize.

#### 2.8 Sub-Workflow Execution

- **Overview:** Invokes an external workflow, presumably for advanced email extraction or follow-up processing.
- **Nodes Involved:**  
  - Execute Workflow (Execute Workflow)  

- **Node Details:**

  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Calls another workflow identified by ID "SDMXLfMXOQ6TDfKq" named "Email Extraction Project".  
    - Configuration: Does not wait for sub-workflow completion, no inputs defined.  
    - Inputs: Summarized item from "Reduce to 1 row".  
    - Outputs: Not connected further in this workflow.  
    - Edge cases: Sub-workflow failure, missing workflow ID, timeout.

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                        | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                  |
|-----------------------------|---------------------------------------------|-------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                             | Starts workflow                     | None                             | Extract Search Terms             | ## n8n Workflow: Local Business Contact Finder with Email Extraction  Contact: rbreen@ynteractive.com |
| Extract Search Terms         | Google Sheets                               | Reads search input from sheet       | When clicking ‘Execute workflow’ | Keep Unprocessed Rows            | ## How to Implement This n8n Workflow (see note for setup instructions and links)           |
| Keep Unprocessed Rows        | Filter                                      | Filters rows not marked complete    | Extract Search Terms             | Filter to first row              |                                                                                              |
| Filter to first row          | Code                                        | Takes first unprocessed row         | Keep Unprocessed Rows            | Mark Row as Completed, Search Google Maps |                                                                                              |
| Mark Row as Completed        | Google Sheets                               | Marks search row as complete        | Filter to first row              | Search Google Maps               |                                                                                              |
| Search Google Maps           | SerpAPI (Google Maps)                       | Queries Google Maps local results   | Mark Row as Completed           | Format output as table           |                                                                                              |
| Format output as table       | Code                                        | Cleans and formats local results    | Search Google Maps              | Loop Over Items                 |                                                                                              |
| Loop Over Items              | SplitInBatches                              | Iterates over each business         | Format output as table          | Save Emails to Sheet, Reduce to 1 row, Scrape Web Page |                                                                                              |
| Save Emails to Sheet         | Google Sheets                               | Saves enriched business data        | Loop Over Items                 |                                 |                                                                                              |
| Reduce to 1 row             | Summarize                                   | Summarizes items to one row         | Loop Over Items                 | Execute Workflow                |                                                                                              |
| Scrape Web Page              | HTTP Request                               | Scrapes website content             | Loop Over Items                 | Extract Email - AI Agent        |                                                                                              |
| Extract Email - AI Agent     | LangChain Agent                             | AI extracts email from text         | Scrape Web Page                 | Loop Over Items                 |                                                                                              |
| Structured Output Parser     | LangChain Structured Output Parser          | Parses AI output                    | Extract Email - AI Agent        | Extract Email - AI Agent        |                                                                                              |
| Execute Workflow            | Execute Workflow                            | Calls external workflow             | Reduce to 1 row                 |                                  |                                                                                              |
| Sticky Note                 | Sticky Note                                 | Workflow info and contact           | None                          | None                           | ## n8n Workflow: Local Business Contact Finder with Email Extraction  Contact: rbreen@ynteractive.com |
| Sticky Note1                | Sticky Note                                 | Setup instructions and resource links| None                          | None                           | See https://docs.google.com/spreadsheets/d/1QgcVMlXRlM_5ZFFUHr6bVK-93Tzia9XseTX03ZYnowI/edit?usp=sharing and other links |
| Sticky Note2                | Sticky Note                                 | Empty                              | None                          | None                           |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - No parameters needed.

2. **Add Google Sheets Node ("Extract Search Terms")**  
   - Node Type: Google Sheets  
   - Operation: Read Rows  
   - Document ID: Your Google Sheet ID containing input data  
   - Sheet Name: Tab containing search terms ("Searches")  
   - Credentials: Connect to Google Sheets OAuth2 with your account.

3. **Add Filter Node ("Keep Unprocessed Rows")**  
   - Node Type: Filter  
   - Condition: `$json.Complete` is empty string  
   - Input: Connect from "Extract Search Terms".

4. **Add Code Node ("Filter to first row")**  
   - Node Type: Code  
   - JavaScript: `return [items[0]];`  
   - Input: Connect from "Keep Unprocessed Rows".

5. **Add Google Sheets Node ("Mark Row as Completed")**  
   - Node Type: Google Sheets  
   - Operation: Append or Update Row  
   - Match by "Search" column  
   - Set "Complete" column to "Yes"  
   - Document ID and Sheet Name: Same as "Extract Search Terms"  
   - Credentials: Use same Google Sheets OAuth2  
   - Input: Connect from "Filter to first row".

6. **Add SerpAPI Node ("Search Google Maps")**  
   - Node Type: SerpAPI  
   - Operation: Google Maps Search  
   - Query: `={{ $json.Search }}`  
   - Location: `={{ $json.Area }}` (latitude,longitude)  
   - Credentials: Add SerpAPI API key credentials  
   - Input: Connect from "Mark Row as Completed".

7. **Add Code Node ("Format output as table")**  
   - Node Type: Code  
   - JavaScript: Parse `local_results` from SerpAPI output, clean URLs, map fields `title`, `website`, `address`, `phone`.  
   - Input: Connect from "Search Google Maps".

8. **Add SplitInBatches Node ("Loop Over Items")**  
   - Node Type: SplitInBatches  
   - Default options (process all items sequentially)  
   - Input: Connect from "Format output as table".

9. **Add HTTP Request Node ("Scrape Web Page")**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/6sigmag~fast-website-content-crawler/run-sync-get-dataset-items`  
   - Body (JSON): `{ "startUrls": ["{{ $json.website }}"] }`  
   - Authentication: Generic HTTP Query with Apify API token as query parameter `token=YOUR_API_KEY`  
   - Input: Connect from "Loop Over Items".

10. **Add LangChain Agent Node ("Extract Email - AI Agent")**  
    - Node Type: LangChain Agent (AI Agent)  
    - Text Input: `={{ $json.text }}` (text from scraped webpage)  
    - System Message: "extract the email address from the text. if there is no email address, output null."  
    - Has Output Parser: Enabled  
    - Credentials: OpenAI API key configured  
    - Input: Connect from "Scrape Web Page".

11. **Add Structured Output Parser Node ("Structured Output Parser")**  
    - Node Type: LangChain Structured Output Parser  
    - JSON Schema Example: `{ "email": "emailaddress" }`  
    - Input: Connect AI output from "Extract Email - AI Agent".

12. **Connect Output Parser Back to AI Agent Node**  
    - Connect from "Structured Output Parser" `ai_outputParser` output to "Extract Email - AI Agent" `ai_languageModel` input (for LangChain internal processing).

13. **Add Google Sheets Node ("Save Emails to Sheet")**  
    - Node Type: Google Sheets  
    - Operation: Append or Update Rows  
    - Match by "title" column  
    - Columns to write: `Area`, `phone`, `title`, `Search`, `address`, `website`, `Search Name`, `email (Manual Entry)`  
    - Document ID: Same Google Sheet, tab ID for "Results"  
    - Credentials: Google Sheets OAuth2  
    - Input: Connect from "Loop Over Items".

14. **Add Summarize Node ("Reduce to 1 row")**  
    - Node Type: Summarize  
    - Fields to Summarize: `title`  
    - Input: Connect from "Loop Over Items".

15. **Add Execute Workflow Node ("Execute Workflow")**  
    - Node Type: Execute Workflow  
    - Workflow ID: ID of your external workflow for email extraction (e.g., "SDMXLfMXOQ6TDfKq")  
    - Options: Do not wait for sub-workflow  
    - Input: Connect from "Reduce to 1 row".

16. **Link all nodes as per described connections:**  
    - Manual Trigger → Extract Search Terms → Keep Unprocessed Rows → Filter to first row →  
      → Mark Row as Completed → Search Google Maps → Format output as table → Loop Over Items →  
      → Scrape Web Page → Extract Email - AI Agent → Structured Output Parser (loop for LangChain internal)  
      → Loop Over Items → Save Emails to Sheet  
      → Loop Over Items → Reduce to 1 row → Execute Workflow

17. **Ensure all credentials are set up:**  
    - Google Sheets OAuth2 credentials with proper access to the spreadsheets  
    - SerpAPI API key credentials  
    - Apify API key as query parameter credential for HTTP Request node  
    - OpenAI API key for LangChain AI nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| To implement this workflow, first copy the Google Sheets template for searches and results: https://docs.google.com/spreadsheets/d/1QgcVMlXRlM_5ZFFUHr6bVK-93Tzia9XseTX03ZYnowI/edit?usp=sharing | Setup instructions for the Google Sheets nodes and data structure                                              |
| Sign up and obtain API keys for SerpAPI (https://serpapi.com/dashboard), Apify (https://apify.com), and OpenAI (https://platform.openai.com/) | Required external service accounts and API keys                                                                  |
| Contact the workflow author for implementation help: rbreen@ynteractive.com                                                | Support contact                                                                                                   |
| Workflow executes a stepwise process: reads search terms, queries Google Maps, scrapes websites, extracts emails with AI, and saves results | Workflow execution logic overview                                                                                 |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal or offensive content. All data processed is legal and publicly accessible.