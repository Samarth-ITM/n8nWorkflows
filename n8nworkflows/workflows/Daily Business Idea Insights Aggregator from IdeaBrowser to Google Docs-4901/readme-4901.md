Daily Business Idea Insights Aggregator from IdeaBrowser to Google Docs

https://n8nworkflows.xyz/workflows/daily-business-idea-insights-aggregator-from-ideabrowser-to-google-docs-4901


# Daily Business Idea Insights Aggregator from IdeaBrowser to Google Docs

### 1. Workflow Overview

This workflow automates the daily aggregation of business idea insights from IdeaBrowser.com and documents them into a Google Doc. It is designed to run once per day, retrieve the "Idea of the Day" page, extract multiple insight-related URLs, fetch content from these URLs, and consolidate the information into a formatted Google Document for easy review and archival.

The workflow logic is divided into the following blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Trigger the workflow daily and fetch the raw HTML content of the Idea of the Day page.
- **1.2 Link Extraction & Insight URL Generation:** Parse the raw HTML to extract the base idea path and generate URLs for various insight subpages related to the idea.
- **1.3 Google Document Creation:** Create a new Google Document titled with the current date and the idea’s slug.
- **1.4 Data Aggregation & URL Processing Loop:** Combine initial data, split insight URLs, iterate through each URL in batches, and fetch their HTML content.
- **1.5 Content Formatting & Google Docs Update:** Convert fetched HTML content into Markdown-friendly text and update the Google Document with this content.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

- **Overview:** This block initiates the workflow daily at a specified hour and retrieves the raw HTML content of the "Idea of the Day" page from IdeaBrowser.
- **Nodes Involved:** Schedule Trigger, Get URL data of idea

---

**Node: Schedule Trigger**

- Type: Schedule Trigger  
- Role: Starts the workflow execution daily at 8 AM (Asia/Kolkata timezone).  
- Configuration: Rule set to trigger at hour 8 daily.  
- Inputs: None  
- Outputs: Triggers "Get URL data of idea" node.  
- Failures: Possible issues include scheduler misconfiguration or timezone errors.

---

**Node: Get URL data of idea**

- Type: HTTP Request  
- Role: Fetches the HTML content of the IdeaBrowser "Idea of the Day" page.  
- Configuration:  
  - URL: https://ideabrowser.com/idea-of-the-day  
  - Auth: Uses HTTP Header Authentication with a cookie credential (named "cookie-for-idea-browser") to access the page if required.  
  - No additional request options set.  
- Inputs: Triggered by Schedule Trigger  
- Outputs: Outputs raw HTML in JSON under `data` field.  
- Failures: Network errors, authentication failures due to invalid cookie, or page not accessible.

---

#### 2.2 Link Extraction & Insight URL Generation

- **Overview:** Parses the retrieved HTML to extract the base path of the idea and generates a list of URLs for different insight pages related to the idea. Also cleans the raw HTML to retain only readable text.  
- **Nodes Involved:** Get the links

---

**Node: Get the links**

- Type: Code (JavaScript)  
- Role: Processes raw HTML to:  
  1. Extract base idea path using regex (`/\/idea\/[^\"\/]+/`)  
  2. Generate a list of insight subpage URLs by appending predefined paths.  
  3. Strip all HTML tags from raw data to produce clean text.  
- Configuration: Custom JS code iterating over input items, applying regex and string processing.  
- Inputs: Receives raw HTML data from "Get URL data of idea" node.  
- Outputs: Adds:  
  - `insightLinks`: Array of full URLs for different insight pages.  
  - `ideaPath`: Base idea path string (e.g., `/idea/ai-powered-historical-ad-style-generator`).  
  - `cleanText`: Plain text extracted from HTML.  
- Failures: Regex may fail if HTML structure changes or idea path is missing, resulting in error flag; malformed HTML might affect text cleaning.

---

#### 2.3 Google Document Creation

- **Overview:** Creates a new Google Document named with the current date and the idea slug, stored in a predefined Google Drive folder.  
- **Nodes Involved:** Create google doc

---

**Node: Create google doc**

- Type: Google Docs  
- Role: Creates a new Google Document for the day’s idea insights.  
- Configuration:  
  - Title: Dynamic expression concatenating current date/time string and the idea slug extracted from `ideaPath`.  
  - Folder ID: Fixed Google Drive folder ID for organizing documents.  
- Inputs: Triggered after "Get the links" node.  
- Outputs: Provides newly created document ID/URL for further updates.  
- Credentials: OAuth2 Google Docs with user credential "mvasi".  
- Failures: OAuth token expiration, insufficient permissions, folder ID invalid or inaccessible.

---

#### 2.4 Data Aggregation & URL Processing Loop

- **Overview:** Combines initial data streams, splits the list of insight URLs, processes them in batches, and fetches HTML content from each URL.  
- **Nodes Involved:** Merge the data, Split the url, Loop over URL, Get URL content of each page

---

**Node: Merge the data**

- Type: Merge  
- Role: Combines outputs from two different nodes (Create google doc and Get the links) into one unified data stream.  
- Configuration: Combine mode set to "combineAll".  
- Inputs: Receives data from "Create google doc" (main output 0) and "Get the links" (main output 1).  
- Outputs: Sends combined data to "Split the url".  
- Failures: Can fail if input data formats mismatch or one input is missing.

---

**Node: Split the url**

- Type: SplitOut  
- Role: Splits the array of insight URLs (`insightLinks`) into individual items to process separately.  
- Configuration: Field to split out set to `insightLinks`.  
- Inputs: From "Merge the data".  
- Outputs: One item per URL to "Loop over URL".  
- Failures: Fail if `insightLinks` is missing or empty.

---

**Node: Loop over URL**

- Type: SplitInBatches  
- Role: Iterates over each URL in batches (default batch size) to manage processing load.  
- Configuration: Default options.  
- Inputs: Receives individual URLs from "Split the url".  
- Outputs: Batch output to "Get URL content of each page".  
- Failures: Batch size issues, loop termination errors.

---

**Node: Get URL content of each page**

- Type: HTTP Request  
- Role: Fetches HTML content of each insight URL.  
- Configuration:  
  - URL: Dynamically set from current item’s `insightLinks` field.  
  - Auth: Uses same HTTP Header Authentication cookie as before.  
- Inputs: Received batch URLs from "Loop over URL".  
- Outputs: Raw HTML content per insight URL to "Markdown1".  
- Failures: Network errors, auth failures, 404 from invalid URLs.

---

#### 2.5 Content Formatting & Google Docs Update

- **Overview:** Converts fetched HTML to Markdown text and inserts the content into the previously created Google Document, then loops back to process remaining URLs.  
- **Nodes Involved:** Markdown1, Update the google docs with the data

---

**Node: Markdown1**

- Type: Markdown  
- Role: Converts raw HTML content from each insight page into Markdown-formatted text suitable for insertion into Google Docs.  
- Configuration: Input HTML is set dynamically from `data` field of fetched content.  
- Inputs: HTML from "Get URL content of each page".  
- Outputs: Markdown text to "Update the google docs with the data".  
- Failures: Conversion errors if HTML malformed.

---

**Node: Update the google docs with the data**

- Type: Google Docs  
- Role: Updates the previously created Google Document by inserting the Markdown-formatted insight text.  
- Configuration:  
  - Operation: Update (insert text)  
  - Document URL: Dynamically references document ID from "Create google doc" node output.  
  - Action: Insert the Markdown text content.  
- Inputs: Receives Markdown text from "Markdown1".  
- Outputs: Loops back to "Loop over URL" to continue processing next batch.  
- Credentials: Same OAuth2 Google Docs credential as document creation.  
- Failures: OAuth expiration, document locked or deleted, insertion failures.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                           | Input Node(s)             | Output Node(s)                   | Sticky Note                                                                                     |
|------------------------------|--------------------|-----------------------------------------|---------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger   | Daily trigger at 8 AM                    | None                      | Get URL data of idea             |                                                                                                |
| Get URL data of idea         | HTTP Request       | Fetch Idea of the Day HTML content       | Schedule Trigger          | Get the links                   |                                                                                                |
| Get the links               | Code               | Extract base path, generate insight URLs | Get URL data of idea       | Create google doc, Merge the data |                                                                                                |
| Create google doc           | Google Docs        | Create a new document for the idea       | Get the links             | Merge the data                   |                                                                                                |
| Merge the data              | Merge              | Combine data streams                      | Create google doc, Get the links | Split the url                  |                                                                                                |
| Split the url               | SplitOut           | Split insightLinks array into single URLs | Merge the data             | Loop over URL                   |                                                                                                |
| Loop over URL               | SplitInBatches     | Iterate URLs in batches                   | Split the url             | Get URL content of each page     |                                                                                                |
| Get URL content of each page | HTTP Request       | Fetch HTML content of each insight page  | Loop over URL             | Markdown1                       |                                                                                                |
| Markdown1                  | Markdown           | Convert HTML to markdown text             | Get URL content of each page | Update the google docs with the data |                                                                                                |
| Update the google docs with the data | Google Docs        | Insert markdown content into Google Doc   | Markdown1                 | Loop over URL                   |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger daily at 8:00 AM (Asia/Kolkata timezone).  

2. **Add an HTTP Request node ("Get URL data of idea")**  
   - URL: `https://ideabrowser.com/idea-of-the-day`  
   - Authentication: HTTP Header Auth using a cookie credential named (e.g.) "cookie-for-idea-browser"  
   - Connect Schedule Trigger → Get URL data of idea  

3. **Add a Code node ("Get the links")**  
   - Paste the provided JavaScript code to:  
     - Extract base idea path via regex `/\/idea\/[^\"\/]+/`  
     - Generate insight URLs by appending fixed insight page paths to base path  
     - Strip HTML tags to produce clean text  
   - Connect Get URL data of idea → Get the links  

4. **Add a Google Docs node ("Create google doc")**  
   - Operation: Create Document  
   - Title: Use expression to concatenate current date/time with the idea slug extracted from `ideaPath` (e.g., `{{$today.toLocaleString().concat("-", $json.ideaPath.split("/")[2])}}`)  
   - Folder ID: Set to your desired Google Drive folder ID  
   - Credentials: Configure with Google Docs OAuth2 credentials  
   - Connect Get the links → Create google doc  

5. **Add a Merge node ("Merge the data")**  
   - Mode: Combine (combineAll)  
   - Connect Create google doc (main output 0) and Get the links (main output 1) → Merge the data  

6. **Add a SplitOut node ("Split the url")**  
   - Field to split out: `insightLinks`  
   - Connect Merge the data → Split the url  

7. **Add a SplitInBatches node ("Loop over URL")**  
   - Use default batch size or configure as needed  
   - Connect Split the url → Loop over URL  

8. **Add an HTTP Request node ("Get URL content of each page")**  
   - URL: Expression set to current item’s `insightLinks` value (`={{ $json.insightLinks }}`)  
   - Authentication: Same HTTP Header Auth cookie credential  
   - Connect Loop over URL (batch output) → Get URL content of each page  

9. **Add a Markdown node ("Markdown1")**  
   - Convert input HTML (`{{ $json.data }}`) to Markdown text  
   - Connect Get URL content of each page → Markdown1  

10. **Add a Google Docs node ("Update the google docs with the data")**  
    - Operation: Update document (insert text)  
    - Document URL: Reference document ID from "Create google doc" (`{{ $('Create google doc').item.json.id }}`)  
    - Actions: Insert Markdown text from Markdown1 node (`{{ $json.data }}`)  
    - Credentials: Same Google Docs OAuth2 credentials  
    - Connect Markdown1 → Update the google docs with the data  

11. **Set Update the google docs with the data node output to loop back to Loop over URL**  
    - This enables processing of all batches sequentially  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow requires a valid HTTP Header Authentication cookie credential from IdeaBrowser to access content. | Credential setup for "cookie-for-idea-browser" HTTP Header Auth |
| Google Docs OAuth2 credentials must have write access to the specified Drive folder.                       | Credential setup "mvasi" for Google Docs API                    |
| The workflow is scheduled in the Asia/Kolkata timezone, adjust as per your locale if needed.              | Scheduling context                                              |
| Insight URLs are hardcoded for specific insight pages; adjust in the code node "Get the links" if IdeaBrowser changes URL structure. | Insight URL paths in JS code                                     |

---

**Disclaimer:**  
The supplied text originates exclusively from an automated workflow built using n8n, an integration and automation platform. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and publicly accessible.