Generate Business Leads with OpenStreetMap Data and Save to Google Sheets

https://n8nworkflows.xyz/workflows/generate-business-leads-with-openstreetmap-data-and-save-to-google-sheets-3827


# Generate Business Leads with OpenStreetMap Data and Save to Google Sheets

### 1. Workflow Overview

This workflow, titled **"Generate Business Leads with OpenStreetMap Data and Save to Google Sheets"**, is designed to automate the generation of targeted business leads by leveraging free OpenStreetMap data accessed through the Overpass API. It is ideal for business owners, marketers, sales professionals, and entrepreneurs who want to build comprehensive lead lists without incurring costs from paid APIs or services.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Query Looping:** Accepts a list of search queries (keywords combined with geographic areas) and iteratively processes each query to avoid API timeouts.
- **1.2 Data Extraction via Overpass API:** Executes the Overpass API queries to retrieve business data matching the search criteria.
- **1.3 Data Organization and Filtering:** Cleans, organizes, and filters the raw data to remove duplicates and irrelevant entries.
- **1.4 Website Scraping for Missing Emails:** For businesses with websites but missing email addresses, scrapes the homepage to extract additional contact emails.
- **1.5 Final Data Aggregation and Export:** Merges all data, applies final filters, and saves the results to Google Sheets for easy access and management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Query Looping

**Overview:**  
This block receives the input list of queries, each specifying an outer area, inner area, and keyword. It loops over these queries in batches to manage API rate limits and prevent timeouts, triggering a sub-workflow for each query.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Loop Over Items1 (SplitInBatches)  
- Execute Workflow  
- Wait

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution, providing a predefined list of queries with fields: `outarea`, `inarea`, and `keyword`.  
  - *Configuration:* Preloaded with many city and state combinations targeting "Restaurant" keywords.  
  - *Connections:* Outputs to Loop Over Items1.  
  - *Edge Cases:* No input means no queries processed; ensure input list is populated.

- **Loop Over Items1**  
  - *Type:* SplitInBatches  
  - *Role:* Processes queries one batch at a time to avoid API overload.  
  - *Configuration:* Default batch size (likely 1) to process queries sequentially.  
  - *Connections:* Outputs to Execute Workflow on batch completion; also has a second output with no connections (unused).  
  - *Edge Cases:* Large input lists may slow processing; batch size can be adjusted.

- **Execute Workflow**  
  - *Type:* Execute Workflow  
  - *Role:* Invokes a sub-workflow that handles data extraction and processing for each query.  
  - *Configuration:* Retry enabled with 3-second wait between attempts; on failure, continues regular output to avoid halting the main workflow.  
  - *Connections:* Outputs to Wait node.  
  - *Edge Cases:* Sub-workflow failures handled gracefully; ensure sub-workflow is accessible and correctly configured.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Introduces a delay between query executions to respect API rate limits and avoid timeouts.  
  - *Configuration:* Default wait time (not explicitly set).  
  - *Connections:* Outputs back to Loop Over Items1 to continue processing next batch.  
  - *Edge Cases:* Too short wait times may cause API throttling; too long increases total runtime.

---

#### 2.2 Data Extraction via Overpass API

**Overview:**  
This block executes the Overpass API queries to retrieve business data matching the input keyword and geographic areas. It parses the API response and prepares the data for further processing.

**Nodes Involved:**  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- HTTP Request  
- Extract List  
- No Operation, do nothing

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for the sub-workflow called by the main Execute Workflow node.  
  - *Configuration:* No parameters; triggers on external execution.  
  - *Connections:* Outputs to HTTP Request node.  
  - *Edge Cases:* Must be triggered by main workflow; standalone execution will not proceed.

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Sends a query to the Overpass API to fetch business data based on the current query parameters.  
  - *Configuration:*  
    - Uses Overpass API endpoint.  
    - Query built dynamically using input parameters (outer area, inner area, keyword).  
    - On error, continues with error output to avoid workflow halt.  
  - *Connections:* Outputs to Extract List on success, No Operation on error.  
  - *Edge Cases:* API timeouts, malformed queries, or rate limiting may cause errors; handled gracefully.

- **Extract List**  
  - *Type:* Set  
  - *Role:* Extracts and formats the list of businesses from the raw API response.  
  - *Configuration:* Uses expressions to parse JSON response and flatten data.  
  - *Connections:* Outputs to Split Out node.  
  - *Edge Cases:* Empty or malformed responses may yield empty lists.

- **No Operation, do nothing**  
  - *Type:* NoOp  
  - *Role:* Placeholder node to handle error branch from HTTP Request without action.  
  - *Configuration:* None.  
  - *Connections:* None.  
  - *Edge Cases:* None.

---

#### 2.3 Data Organization and Filtering

**Overview:**  
This block organizes the raw data, checks for presence of websites and emails, filters out entries lacking contact info, and prepares data for website scraping if needed.

**Nodes Involved:**  
- Split Out  
- Set Output Fields  
- Organize Data  
- Has No Website? (If)  
- Has Email? (If)  
- Append Items (No Website and/not Email)  
- Loop Over Items  
- Merge Items with HTML  
- Loop Over Items (second instance)  
- Get Website HTML

**Node Details:**

- **Split Out**  
  - *Type:* SplitOut  
  - *Role:* Splits the list of businesses into individual items for processing.  
  - *Connections:* Outputs to Set Output Fields.

- **Set Output Fields**  
  - *Type:* Set  
  - *Role:* Defines and standardizes output fields such as business name, address, phone, email, website, etc.  
  - *Connections:* Outputs to Organize Data.

- **Organize Data**  
  - *Type:* Code  
  - *Role:* Custom JavaScript code to clean and organize data fields, remove duplicates, and standardize formats.  
  - *Connections:* Outputs to Has No Website? node.

- **Has No Website?**  
  - *Type:* If  
  - *Role:* Checks if the business entry lacks a website URL.  
  - *Connections:*  
    - True branch outputs to Append Items (No Website and/not Email).  
    - False branch outputs to Has Email? node.

- **Has Email?**  
  - *Type:* If  
  - *Role:* Checks if the business entry has an email address.  
  - *Connections:*  
    - True branch outputs to Append Items (No Website and/not Email) (second input).  
    - False branch outputs to Merge Items with HTML and Loop Over Items.

- **Append Items (No Website and/not Email)**  
  - *Type:* Merge  
  - *Role:* Collects items that either have no website or no email for further processing or filtering.  
  - *Connections:* Outputs to Append Items.

- **Append Items**  
  - *Type:* Merge  
  - *Role:* Aggregates items from different branches for final filtering.  
  - *Connections:* Outputs to Filter Away Items With No Contact Info.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes items with websites but missing emails one by one for website scraping.  
  - *Connections:*  
    - First output to Get Website HTML.  
    - Second output to Merge Items with HTML.

- **Merge Items with HTML**  
  - *Type:* Merge  
  - *Role:* Combines items after website HTML is fetched for email scraping.  
  - *Connections:* Outputs to Scrape HomePage.

- **Get Website HTML**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the homepage HTML of the business website to extract emails.  
  - *Configuration:* Continues on error to avoid workflow halt if website is unreachable.  
  - *Connections:* Outputs back to Loop Over Items for iterative processing.

---

#### 2.4 Website Scraping for Missing Emails

**Overview:**  
This block scrapes the fetched website HTML to extract email addresses using regular expressions and filters out irrelevant emails.

**Nodes Involved:**  
- Scrape HomePage  
- Clean Emails

**Node Details:**

- **Scrape HomePage**  
  - *Type:* Code  
  - *Role:* Custom JavaScript code that parses the website HTML content to extract email addresses using regex patterns.  
  - *Configuration:* Continues on error to avoid workflow interruption.  
  - *Connections:* Outputs to Clean Emails.

- **Clean Emails**  
  - *Type:* Code  
  - *Role:* Cleans and filters extracted emails, removing duplicates and emails matching unwanted patterns (e.g., generic or no-reply addresses).  
  - *Connections:* Outputs to Append Items for final aggregation.

---

#### 2.5 Final Data Aggregation and Export

**Overview:**  
This block filters out items lacking any contact information and saves the cleaned, enriched lead data to Google Sheets.

**Nodes Involved:**  
- Filter Away Items With No Contact Info  
- Google Sheets

**Node Details:**

- **Filter Away Items With No Contact Info**  
  - *Type:* Filter  
  - *Role:* Removes any business entries that do not contain at least one form of contact information (email, phone, website, etc.).  
  - *Connections:* Outputs to Google Sheets.

- **Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Saves the final list of business leads into a configured Google Sheets spreadsheet for easy access and management.  
  - *Configuration:* Requires Google OAuth2 credentials authorized for the target spreadsheet.  
  - *Connections:* None (end node).  
  - *Edge Cases:* Authentication errors, quota limits, or spreadsheet access issues.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                                  | Input Node(s)                     | Output Node(s)                      | Sticky Note                           |
|----------------------------------|---------------------------|-------------------------------------------------|----------------------------------|-----------------------------------|-------------------------------------|
| When clicking ‘Test workflow’    | Manual Trigger            | Entry point with predefined queries             | -                                | Loop Over Items1                   |                                     |
| Loop Over Items1                 | SplitInBatches            | Batch processing of queries                      | When clicking ‘Test workflow’    | Execute Workflow, (unused output) |                                     |
| Execute Workflow                 | Execute Workflow          | Calls sub-workflow for each query                | Loop Over Items1                 | Wait                              |                                     |
| Wait                            | Wait                      | Delay between query executions                   | Execute Workflow                 | Loop Over Items1                  |                                     |
| When Executed by Another Workflow| Execute Workflow Trigger  | Sub-workflow entry point                         | -                                | HTTP Request                     |                                     |
| HTTP Request                    | HTTP Request              | Calls Overpass API to fetch business data       | When Executed by Another Workflow| Extract List, No Operation        |                                     |
| Extract List                   | Set                       | Parses and extracts business list from response | HTTP Request                    | Split Out                       |                                     |
| No Operation, do nothing        | NoOp                      | Handles error branch from HTTP Request           | HTTP Request                    | -                               |                                     |
| Split Out                      | SplitOut                  | Splits business list into individual items       | Extract List                   | Set Output Fields               |                                     |
| Set Output Fields              | Set                       | Standardizes output fields                        | Split Out                     | Organize Data                  |                                     |
| Organize Data                 | Code                      | Cleans and organizes data                         | Set Output Fields              | Has No Website?                |                                     |
| Has No Website?               | If                        | Checks for missing website URL                    | Organize Data                 | Append Items (No Website and/not Email), Has Email? |                                     |
| Has Email?                   | If                        | Checks for presence of email                      | Has No Website?               | Append Items (No Website and/not Email), Merge Items with HTML, Loop Over Items |                                     |
| Append Items (No Website and/not Email) | Merge                     | Aggregates items missing website or email        | Has No Website?, Has Email?   | Append Items                   |                                     |
| Append Items                 | Merge                     | Aggregates all filtered items                     | Append Items (No Website and/not Email), Clean Emails | Filter Away Items With No Contact Info |                                     |
| Loop Over Items              | SplitInBatches            | Processes items with websites for scraping       | Has Email?                    | Get Website HTML, Merge Items with HTML |                                     |
| Merge Items with HTML        | Merge                     | Combines items after fetching website HTML       | Loop Over Items               | Scrape HomePage                |                                     |
| Get Website HTML             | HTTP Request              | Fetches homepage HTML for email scraping          | Loop Over Items               | Loop Over Items               |                                     |
| Scrape HomePage              | Code                      | Extracts emails from website HTML                 | Merge Items with HTML         | Clean Emails                  |                                     |
| Clean Emails                 | Code                      | Cleans and filters extracted emails               | Scrape HomePage              | Append Items                  |                                     |
| Filter Away Items With No Contact Info | Filter                    | Removes entries without any contact info          | Append Items                 | Google Sheets                 |                                     |
| Google Sheets                | Google Sheets             | Saves final leads to Google Sheets                 | Filter Away Items With No Contact Info | -                             |                                     |
| Sticky Note                   | Sticky Note               | (Empty content)                                   | -                            | -                             |                                     |
| Sticky Note1                  | Sticky Note               | (Empty content)                                   | -                            | -                             |                                     |
| Sticky Note2 (disabled)       | Sticky Note               | (Empty content, disabled)                         | -                            | -                             |                                     |
| Sticky Note3 (disabled)       | Sticky Note               | (Empty content, disabled)                         | -                            | -                             |                                     |
| Sticky Note6                  | Sticky Note               | (Empty content)                                   | -                            | -                             |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Start workflow manually with predefined queries.  
   - Configure static JSON input with fields: `outarea`, `inarea`, `keyword` (e.g., `"California", "Los Angeles", "real estate"`).

2. **Add SplitInBatches Node**  
   - Name: `Loop Over Items1`  
   - Purpose: Process queries one by one to avoid API timeouts.  
   - Connect output of Manual Trigger to this node.  
   - Use default batch size (1).

3. **Add Execute Workflow Node**  
   - Name: `Execute Workflow`  
   - Purpose: Call sub-workflow for each query.  
   - Connect output of `Loop Over Items1` to this node.  
   - Enable retry on failure with 3-second wait.  
   - Set "On Error" to continue regular output.

4. **Add Wait Node**  
   - Name: `Wait`  
   - Purpose: Delay between query executions.  
   - Connect output of `Execute Workflow` to this node.  
   - Connect output of `Wait` back to `Loop Over Items1` to continue looping.

5. **Create Sub-Workflow** (called by `Execute Workflow`)  
   - Entry node: `When Executed by Another Workflow` (Execute Workflow Trigger).  
   - Connect to `HTTP Request` node configured to call Overpass API with dynamic query parameters (`outarea`, `inarea`, `keyword`).  
   - Configure HTTP Request to continue on error.  
   - Connect HTTP Request success output to `Extract List` (Set node) that parses and extracts business data.  
   - Connect HTTP Request error output to `No Operation, do nothing` node.  
   - Connect `Extract List` to `Split Out` node to split list into individual items.

6. **Add Set Output Fields Node**  
   - Name: `Set Output Fields`  
   - Purpose: Standardize fields like name, address, phone, email, website, etc.  
   - Connect `Split Out` to this node.

7. **Add Code Node**  
   - Name: `Organize Data`  
   - Purpose: Clean and deduplicate data, standardize formats.  
   - Connect `Set Output Fields` to this node.

8. **Add If Node**  
   - Name: `Has No Website?`  
   - Purpose: Check if website URL is missing.  
   - Connect `Organize Data` to this node.  
   - True branch connects to `Append Items (No Website and/not Email)` (Merge node).  
   - False branch connects to next If node.

9. **Add If Node**  
   - Name: `Has Email?`  
   - Purpose: Check if email is present.  
   - Connect false branch of `Has No Website?` to this node.  
   - True branch connects to second input of `Append Items (No Website and/not Email)` (Merge node).  
   - False branch connects to `Merge Items with HTML` and `Loop Over Items`.

10. **Add Merge Node**  
    - Name: `Append Items (No Website and/not Email)`  
    - Purpose: Aggregate items missing website or email.  
    - Connect true branches of previous If nodes here.  
    - Connect output to `Append Items` (Merge node).

11. **Add Merge Node**  
    - Name: `Append Items`  
    - Purpose: Aggregate all filtered items.  
    - Connect output of `Append Items (No Website and/not Email)` and output of `Clean Emails` node here.  
    - Connect output to `Filter Away Items With No Contact Info` (Filter node).

12. **Add SplitInBatches Node**  
    - Name: `Loop Over Items`  
    - Purpose: Process items with websites but missing emails for scraping.  
    - Connect false branch of `Has Email?` to this node.  
    - First output connects to `Get Website HTML`.  
    - Second output connects to `Merge Items with HTML`.

13. **Add Merge Node**  
    - Name: `Merge Items with HTML`  
    - Purpose: Combine items after fetching website HTML.  
    - Connect output of `Loop Over Items` (second output) and output of `Get Website HTML` here.  
    - Connect output to `Scrape HomePage`.

14. **Add HTTP Request Node**  
    - Name: `Get Website HTML`  
    - Purpose: Fetch homepage HTML for email scraping.  
    - Configure to continue on error.  
    - Connect output back to `Loop Over Items` (to iterate).

15. **Add Code Node**  
    - Name: `Scrape HomePage`  
    - Purpose: Extract emails from website HTML using regex.  
    - Configure to continue on error.  
    - Connect output to `Clean Emails`.

16. **Add Code Node**  
    - Name: `Clean Emails`  
    - Purpose: Filter and deduplicate extracted emails.  
    - Connect output to `Append Items`.

17. **Add Filter Node**  
    - Name: `Filter Away Items With No Contact Info`  
    - Purpose: Remove entries without any contact info (email, phone, website).  
    - Connect output of `Append Items` here.  
    - Connect output to `Google Sheets`.

18. **Add Google Sheets Node**  
    - Name: `Google Sheets`  
    - Purpose: Save final leads to Google Sheets.  
    - Configure with Google OAuth2 credentials and target spreadsheet.  
    - Connect output of Filter node here.

19. **Authorize Credentials**  
    - Set up Google Sheets OAuth2 credentials with access to the target spreadsheet.  
    - No special credentials needed for HTTP Request to Overpass API.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                         |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow uses the free OpenStreetMap Overpass API to gather business data without paid API costs.         | Workflow description                                   |
| Video tutorial available for detailed walkthrough: https://youtu.be/6WVfAIXdwsE                                | Video tutorial link                                    |
| Customize queries by editing the input JSON in the manual trigger node to target different industries or areas. | Setup instructions                                     |
| Adjust email filtering regex in the "Clean Emails" code node to exclude unwanted email patterns.                | Data cleaning customization                            |
| Google Sheets node requires OAuth2 credentials authorized with access to the target spreadsheet.                | Credential setup                                       |
| The workflow handles API rate limiting and pagination by processing queries in batches with delays.            | Workflow design note                                   |
| Sub-workflow invoked by "Execute Workflow" node must be imported or created separately with matching logic.    | Sub-workflow dependency                                |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and modifying the "Generate Business Leads with OpenStreetMap Data and Save to Google Sheets" workflow in n8n. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and automation agents to work effectively with this workflow.