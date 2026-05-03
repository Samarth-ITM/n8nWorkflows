Filter URLs with AI-Powered robots.txt Compliance & Source Verification

https://n8nworkflows.xyz/workflows/filter-urls-with-ai-powered-robots-txt-compliance---source-verification-8138


# Filter URLs with AI-Powered robots.txt Compliance & Source Verification

---

### 1. Workflow Overview

This workflow, titled **"URL Officer - Respect robots.txt and Avoid Undesirable Sources"**, is designed to validate and filter URLs by verifying their compliance with site-specific robots.txt rules and by excluding URLs from forbidden or undesirable sources. It is ideal for scenarios where automated URL processing, web scraping, or content aggregation must respect website crawling rules and avoid restricted or blacklisted domains.

The workflow is logically divided into these main blocks:

- **1.1 Initialization & Input Handling:** Receives URLs and extracts their base domain for further processing.
- **1.2 Forbidden URLs Verification:** Checks if the URL is part of a forbidden URLs list stored in a database.
- **1.3 Robots.txt Retrieval & Cache Management:** Retrieves the robots.txt file for the base URL, caches it in a database, and updates the cache when necessary.
- **1.4 robots.txt Compliance Check:** Parses the robots.txt rules to determine if crawling the URL is allowed.
- **1.5 AI-Powered Decision & Output Preparation:** Uses AI language models to extract and verify information and prepares the final output indicating whether the URL is allowed or disallowed.
- **1.6 Database Maintenance (Disabled):** Nodes related to manual or testing database operations, disabled in the current configuration.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Input Handling

- **Overview:**  
Extracts the base URL from the input URL to standardize domain handling and initiates the workflow.

- **Nodes Involved:**  
  - Start  
  - Get Base URL  
  - Create forbidden urls table  
  - Check forbidden urls table  
  - If forbidden url Detected  

- **Node Details:**

  - **Start**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point for the workflow to receive URLs for processing.  
    - *Connections:* Outputs to `Get Base URL`.  
    - *Failure Cases:* None, but workflow inactive by default.

  - **Get Base URL**  
    - *Type:* Code  
    - *Role:* Extracts the base domain (e.g., scheme + domain) from the input URL for robots.txt retrieval and forbidden URL checks.  
    - *Configuration:* Uses JavaScript code to parse and output base URL.  
    - *Inputs:* URL from Start node.  
    - *Outputs:* Base URL to `Create forbidden urls table`.  
    - *Edge Cases:* Malformed URLs could cause parsing failure; validation recommended.

  - **Create forbidden urls table**  
    - *Type:* Postgres  
    - *Role:* Ensures the existence of a database table to store forbidden URLs.  
    - *Configuration:* Runs a SQL command to create the table if it does not exist.  
    - *Inputs:* Base URL from `Get Base URL`.  
    - *Outputs:* Triggers `Check forbidden urls table`.  
    - *Failure Cases:* Database connection errors, SQL syntax issues.

  - **Check forbidden urls table**  
    - *Type:* Postgres  
    - *Role:* Queries the forbidden URLs table to determine if the current URL is forbidden.  
    - *Configuration:* SQL SELECT query filtering by the input URL.  
    - *Inputs:* From `Create forbidden urls table`.  
    - *Outputs:* Leads to `If forbidden url Detected`.  
    - *Failure Cases:* DB query failure, empty results (no forbidden URL found).

  - **If forbidden url Detected**  
    - *Type:* If Condition  
    - *Role:* Branches workflow based on whether the URL is found in the forbidden list.  
    - *Configuration:* Checks if query returned any rows.  
    - *Inputs:* From `Check forbidden urls table`.  
    - *Outputs:*  
      - If true: `Return Disallow` (immediately disallow the URL).  
      - If false: `Create robots.txt Table` (proceed with robots.txt validation).  
    - *Edge Cases:* False positives if query logic is incorrect.

#### 1.2 Forbidden URLs Verification

- **Overview:**  
Decides early whether to block URLs from undesirable sources by referencing a forbidden URLs database table.

- **Nodes Involved:**  
  - Return Disallow  
  - Create robots.txt Table  

- **Node Details:**

  - **Return Disallow**  
    - *Type:* Set  
    - *Role:* Sets output indicating the URL is disallowed due to forbidden status.  
    - *Configuration:* Sets flags or messages in output data to indicate disallowance.  
    - *Inputs:* From `If forbidden url Detected` (true branch).  
    - *Outputs:* Ends workflow or routes to output nodes if any downstream exist.  
    - *Edge Cases:* None specific; mostly a data flag.

  - **Create robots.txt Table**  
    - *Type:* Postgres  
    - *Role:* Ensures the database table for caching robots.txt data exists.  
    - *Configuration:* SQL CREATE TABLE IF NOT EXISTS command.  
    - *Inputs:* From `If forbidden url Detected` (false branch).  
    - *Outputs:* Triggers `Check robots.txt Table`.  
    - *Failure Cases:* DB connection or SQL errors.

#### 1.3 Robots.txt Retrieval & Cache Management

- **Overview:**  
Fetches robots.txt for the base URL, manages caching in a Postgres table, and decides whether to update or reuse cached data.

- **Nodes Involved:**  
  - Check robots.txt Table  
  - If robots.txt Found and Updated  
  - Prepare Robots.txt Check  
  - Get Robots.txt  
  - Prepare Robots.txt Check 2  
  - Assume Disallow  
  - Upsert robots.txt Table  

- **Node Details:**

  - **Check robots.txt Table**  
    - *Type:* Postgres  
    - *Role:* Checks if robots.txt for the domain is already cached in the database.  
    - *Configuration:* SQL SELECT query filtering by base URL.  
    - *Inputs:* From `Create robots.txt Table`.  
    - *Outputs:* To `If robots.txt Found and Updated`.  
    - *Failure Cases:* DB connection or query errors.

  - **If robots.txt Found and Updated**  
    - *Type:* If Condition  
    - *Role:* Determines whether cached robots.txt exists and is up to date.  
    - *Configuration:* Checks for presence and timestamp freshness of cached robots.txt.  
    - *Inputs:* From `Check robots.txt Table`.  
    - *Outputs:*  
      - True: `Prepare Robots.txt Check 2` (use cached data).  
      - False: `Prepare Robots.txt Check` (fetch fresh robots.txt).  
    - *Edge Cases:* Cache staleness detection logic must be robust.

  - **Prepare Robots.txt Check**  
    - *Type:* Set  
    - *Role:* Prepares parameters for HTTP request to fetch robots.txt.  
    - *Configuration:* Sets URL to `${baseURL}/robots.txt` and HTTP method GET.  
    - *Inputs:* From `If robots.txt Found and Updated` (false branch).  
    - *Outputs:* To `Get Robots.txt`.  
    - *Edge Cases:* None direct; malformed baseURL may cause request failure.

  - **Get Robots.txt**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves robots.txt content from the domain.  
    - *Configuration:* GET request to robots.txt URL.  
    - *Inputs:* From `Prepare Robots.txt Check`.  
    - *Outputs:*  
      - On success: `Prepare Robots.txt Check 2`.  
      - On failure: `Assume Disallow`.  
    - *Failure Cases:* Network errors, 404 not found (no robots.txt), timeout.

  - **Prepare Robots.txt Check 2**  
    - *Type:* Set  
    - *Role:* Prepares data from robots.txt content for further code-based analysis.  
    - *Inputs:* From `Get Robots.txt` (success) or `Check robots.txt Table` (cached).  
    - *Outputs:* To `Check Robots.txt`.  
    - *Edge Cases:* Empty or malformed robots.txt content.

  - **Assume Disallow**  
    - *Type:* Set  
    - *Role:* Sets default output assuming URL is disallowed when robots.txt is unavailable.  
    - *Inputs:* From `Get Robots.txt` (error output).  
    - *Outputs:* To `Prepare Output`.  
    - *Edge Cases:* Conservative fallback; may block URLs unnecessarily.

  - **Upsert robots.txt Table**  
    - *Type:* Postgres  
    - *Role:* Inserts or updates robots.txt content in the cache table with timestamp.  
    - *Inputs:* From `Prepare Output`.  
    - *Outputs:* To `Output`.  
    - *Failure Cases:* DB write errors, race conditions.

#### 1.4 robots.txt Compliance Check

- **Overview:**  
Analyzes robots.txt rules to decide if crawling the specific URL is allowed or disallowed.

- **Nodes Involved:**  
  - Check Robots.txt  
  - If Link Allowed  
  - Information Extractor  
  - If Link Allowed 2  
  - Prepare Output for Link Allowed  
  - Prepare Output for Link Disallowed  

- **Node Details:**

  - **Check Robots.txt**  
    - *Type:* Code  
    - *Role:* Parses the robots.txt content and checks the URL against disallow/allow rules.  
    - *Inputs:* From `Prepare Robots.txt Check 2`.  
    - *Outputs:* To `If Link Allowed`.  
    - *Edge Cases:* Complex or non-standard robots.txt syntax may cause parsing errors.

  - **If Link Allowed**  
    - *Type:* If Condition  
    - *Role:* Branches based on whether the URL is allowed per robots.txt rules.  
    - *Inputs:* From `Check Robots.txt`.  
    - *Outputs:*  
      - True: `Information Extractor` (extracts relevant info using AI).  
      - False: `Prepare Output for Link Disallowed`.  
    - *Edge Cases:* False positives if robots.txt parsing is incorrect.

  - **Information Extractor**  
    - *Type:* Langchain Information Extractor  
    - *Role:* Uses AI to analyze URL or page content for further validation or metadata extraction.  
    - *Parameters:* Utilizes model selected dynamically by `Model Selector`.  
    - *Inputs:* From `If Link Allowed`.  
    - *Outputs:*  
      - Success: `If Link Allowed 2`.  
      - On error: `Assume False Output`.  
    - *Failure Cases:* AI model timeouts, API limits, or parsing failures.

  - **If Link Allowed 2**  
    - *Type:* If Condition  
    - *Role:* Final decision making based on AI extraction results.  
    - *Inputs:* From `Information Extractor`.  
    - *Outputs:*  
      - True branch: `Prepare Output for Link Allowed`.  
      - False branch: `Prepare Output for Link Disallowed`.  
    - *Edge Cases:* AI confidence thresholds may need tuning.

  - **Prepare Output for Link Allowed**  
    - *Type:* Set  
    - *Role:* Sets output data indicating the URL is allowed.  
    - *Inputs:* From `If Link Allowed 2`.  
    - *Outputs:* To `Prepare Output`.  

  - **Prepare Output for Link Disallowed**  
    - *Type:* Set  
    - *Role:* Sets output data indicating the URL is disallowed.  
    - *Inputs:* From `If Link Allowed` or `If Link Allowed 2` (false branch).  
    - *Outputs:* To `Prepare Output`.

#### 1.5 AI-Powered Decision & Output Preparation

- **Overview:**  
Finalizes output data and updates the robots.txt cache accordingly.

- **Nodes Involved:**  
  - Prepare Output  
  - Upsert robots.txt Table  
  - Output  

- **Node Details:**

  - **Prepare Output**  
    - *Type:* Set  
    - *Role:* Prepares final structured output data, possibly including URL status, metadata, and decision rationale.  
    - *Inputs:* From `Prepare Output for Link Allowed`, `Prepare Output for Link Disallowed`, and `Assume Disallow`.  
    - *Outputs:* To `Upsert robots.txt Table`.  

  - **Upsert robots.txt Table**  
    - *Revisited here for clarity*  
    - *Role:* Updates the robots.txt cache to reflect any new data or changes post-check.  
    - *Inputs:* From `Prepare Output`.  
    - *Outputs:* To `Output`.  

  - **Output**  
    - *Type:* Set  
    - *Role:* Final output node; could be connected to further downstream workflows or API responses.  
    - *Inputs:* From `Upsert robots.txt Table`.  
    - *Outputs:* End of workflow.  

#### 1.6 Database Maintenance (Disabled Nodes)

- Includes nodes for deleting tables, manual upsert, and manual table creation for both robots.txt and forbidden URLs tables. These nodes are disabled and serve as maintenance utilities.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                          | Input Node(s)                    | Output Node(s)                                 | Sticky Note                      |
|--------------------------------|----------------------------------|----------------------------------------|---------------------------------|------------------------------------------------|---------------------------------|
| Start                          | Execute Workflow Trigger          | Entry point                            | -                               | Get Base URL                                   |                                 |
| Get Base URL                   | Code                             | Extract base URL from input            | Start                           | Create forbidden urls table                     |                                 |
| Create forbidden urls table    | Postgres                         | Ensure forbidden URLs DB table exists  | Get Base URL                   | Check forbidden urls table                      |                                 |
| Check forbidden urls table     | Postgres                         | Query forbidden URLs table             | Create forbidden urls table      | If forbidden url Detected                       |                                 |
| If forbidden url Detected      | If                               | Branch on forbidden URL presence       | Check forbidden urls table      | Return Disallow, Create robots.txt Table        |                                 |
| Return Disallow                | Set                              | Mark URL disallowed                    | If forbidden url Detected       | (terminates or outputs)                         |                                 |
| Create robots.txt Table        | Postgres                         | Ensure robots.txt cache table exists   | If forbidden url Detected       | Check robots.txt Table                          |                                 |
| Check robots.txt Table         | Postgres                         | Check cached robots.txt                | Create robots.txt Table          | If robots.txt Found and Updated                 |                                 |
| If robots.txt Found and Updated| If                               | Branch on cache presence and freshness | Check robots.txt Table          | Prepare Robots.txt Check 2, Prepare Robots.txt Check |                                 |
| Prepare Robots.txt Check       | Set                              | Prepare HTTP request for robots.txt    | If robots.txt Found and Updated | Get Robots.txt                                  |                                 |
| Get Robots.txt                 | HTTP Request                    | Fetch robots.txt file                   | Prepare Robots.txt Check        | Prepare Robots.txt Check 2, Assume Disallow    |                                 |
| Prepare Robots.txt Check 2     | Set                              | Prepare data for robots.txt parsing    | Get Robots.txt, Check robots.txt Table | Check Robots.txt                         |                                 |
| Assume Disallow                | Set                              | Default to disallow if robots.txt missing | Get Robots.txt (error)          | Prepare Output                                  |                                 |
| Check Robots.txt               | Code                             | Parse and check robots.txt rules       | Prepare Robots.txt Check 2      | If Link Allowed                                 |                                 |
| If Link Allowed               | If                               | Branch on robots.txt URL permission     | Check Robots.txt               | Information Extractor, Prepare Output for Link Disallowed |                             |
| Information Extractor          | Langchain Information Extractor | AI extraction for URL validation       | If Link Allowed                | If Link Allowed 2, Assume False Output          |                                 |
| If Link Allowed 2             | If                               | Final decision based on AI extraction   | Information Extractor          | Prepare Output for Link Allowed, Prepare Output for Link Disallowed |                             |
| Prepare Output for Link Allowed| Set                              | Prepare output for allowed URL          | If Link Allowed 2             | Prepare Output                                  |                                 |
| Prepare Output for Link Disallowed | Set                           | Prepare output for disallowed URL       | If Link Allowed, If Link Allowed 2 | Prepare Output                              |                                 |
| Prepare Output                | Set                              | Compose final output                     | Prepare Output for Link Allowed, Prepare Output for Link Disallowed, Assume Disallow | Upsert robots.txt Table |                                 |
| Upsert robots.txt Table       | Postgres                         | Cache or update robots.txt content      | Prepare Output                | Output                                          |                                 |
| Output                       | Set                              | Final output node                       | Upsert robots.txt Table        | -                                              |                                 |
| Schedule Trigger             | Schedule Trigger                 | (Disabled) Scheduled start              | -                             | Start (Tests)                                   |                                 |
| Start (Tests)                | Set                              | (Disabled) Test input start             | Schedule Trigger              | -                                              |                                 |
| Create forbidden urls table  | Postgres                       | (Repetition node)                       | Get Base URL                 | Check forbidden urls table                      |                                 |
| Additional disabled nodes (Manual Upsert, Delete tables, Check all tables) | Postgres, Set | DB maintenance utilities (disabled) | Various                       | Various                                         |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the entry node:**  
   - Add an **Execute Workflow Trigger** node named `Start` as the entry point.

2. **Extract Base URL:**  
   - Add a **Code** node named `Get Base URL`.  
   - Configure it to parse the input URL and extract the base domain (scheme + host).  
   - Connect `Start` → `Get Base URL`.

3. **Create Forbidden URLs Table:**  
   - Add a **Postgres** node named `Create forbidden urls table`.  
   - Configure it with SQL to create a forbidden URLs table if it doesn't exist, including columns for URLs and metadata.  
   - Connect `Get Base URL` → `Create forbidden urls table`.

4. **Check Forbidden URLs Table:**  
   - Add a **Postgres** node named `Check forbidden urls table`.  
   - Configure it to query the forbidden URLs table filtered by the input URL.  
   - Connect `Create forbidden urls table` → `Check forbidden urls table`.

5. **If Forbidden URL Detected:**  
   - Add an **If** node named `If forbidden url Detected`.  
   - Configure it to check if results from previous query exist.  
   - Connect `Check forbidden urls table` → `If forbidden url Detected`.

6. **Return Disallow on Forbidden URL:**  
   - Add a **Set** node named `Return Disallow`.  
   - Configure it to set output indicating the URL is disallowed.  
   - Connect `If forbidden url Detected` (true) → `Return Disallow`.

7. **Create robots.txt Table:**  
   - Add a **Postgres** node named `Create robots.txt Table`.  
   - Configure it to create the robots.txt cache table if not existing.  
   - Connect `If forbidden url Detected` (false) → `Create robots.txt Table`.

8. **Check robots.txt Table:**  
   - Add a **Postgres** node named `Check robots.txt Table`.  
   - Configure SQL to check if robots.txt is cached for the base URL.  
   - Connect `Create robots.txt Table` → `Check robots.txt Table`.

9. **If robots.txt Found and Updated:**  
   - Add an **If** node named `If robots.txt Found and Updated`.  
   - Configure to check if cache exists and is fresh based on timestamp.  
   - Connect `Check robots.txt Table` → `If robots.txt Found and Updated`.

10. **Prepare Robots.txt Check (Fetch):**  
    - Add a **Set** node named `Prepare Robots.txt Check`.  
    - Configure it to set parameters for an HTTP GET request to `${baseURL}/robots.txt`.  
    - Connect `If robots.txt Found and Updated` (false) → `Prepare Robots.txt Check`.

11. **Get Robots.txt:**  
    - Add an **HTTP Request** node named `Get Robots.txt`.  
    - Configure it for GET method using URL from previous node.  
    - Set "Continue on Fail" to true.  
    - Connect `Prepare Robots.txt Check` → `Get Robots.txt`.

12. **Prepare Robots.txt Check 2:**  
    - Add a **Set** node named `Prepare Robots.txt Check 2`.  
    - Configure to prepare robots.txt content for parsing.  
    - Connect both:  
      - `If robots.txt Found and Updated` (true) → `Prepare Robots.txt Check 2`  
      - `Get Robots.txt` (success) → `Prepare Robots.txt Check 2`

13. **Assume Disallow (Error Fallback):**  
    - Add a **Set** node named `Assume Disallow`.  
    - Configure to set output indicating disallow if robots.txt fetch fails.  
    - Connect `Get Robots.txt` (error) → `Assume Disallow`.

14. **Check Robots.txt:**  
    - Add a **Code** node named `Check Robots.txt`.  
    - Configure JavaScript to parse robots.txt content and decide if the URL is allowed.  
    - Connect `Prepare Robots.txt Check 2` → `Check Robots.txt`.

15. **If Link Allowed:**  
    - Add an **If** node named `If Link Allowed`.  
    - Configure to branch based on robots.txt check result.  
    - Connect `Check Robots.txt` → `If Link Allowed`.

16. **Information Extractor:**  
    - Add a **Langchain Information Extractor** node named `Information Extractor`.  
    - Configure credentials for AI models (see below).  
    - Connect `If Link Allowed` (true) → `Information Extractor`.

17. **Assume False Output on AI error:**  
    - Connect `Information Extractor` error output to a **Set** node named `Assume False Output`.  
    - Configure it to mark the URL as disallowed if AI fails.

18. **If Link Allowed 2 (AI decision):**  
    - Add an **If** node named `If Link Allowed 2`.  
    - Configure it based on AI extraction result confidence or validation.  
    - Connect `Information Extractor` (success) → `If Link Allowed 2`.

19. **Prepare Output for Link Allowed:**  
    - Add a **Set** node named `Prepare Output for Link Allowed`.  
    - Configure to set output marking URL allowed.  
    - Connect `If Link Allowed 2` (true) → `Prepare Output for Link Allowed`.

20. **Prepare Output for Link Disallowed:**  
    - Add a **Set** node named `Prepare Output for Link Disallowed`.  
    - Configure output marking URL disallowed.  
    - Connect `If Link Allowed` (false) and `If Link Allowed 2` (false) → `Prepare Output for Link Disallowed`.

21. **Prepare Output:**  
    - Add a **Set** node named `Prepare Output`.  
    - Combine outputs from allowed/disallowed paths plus `Assume Disallow`.  
    - Connect all to `Prepare Output`.

22. **Upsert robots.txt Table:**  
    - Add a **Postgres** node named `Upsert robots.txt Table`.  
    - Configure to insert or update robots.txt cache data.  
    - Connect `Prepare Output` → `Upsert robots.txt Table`.

23. **Output:**  
    - Add a **Set** node named `Output`.  
    - Configure to finalize and optionally send output downstream.  
    - Connect `Upsert robots.txt Table` → `Output`.

24. **Configure Credentials for AI Nodes:**  
    - Setup credentials for the Langchain AI nodes:  
      - `Mistral Cloud Chat Model`  
      - `Groq Chat Model`  
      - `Google Gemini Chat Model`  
    - These models are dynamically selected by the `Model Selector` node.

25. **Model Selector Node:**  
    - Add a Langchain `Model Selector` node to route AI tasks to the appropriate AI model node based on conditions or preferences.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                      |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow respects robots.txt and forbidden URLs lists to ensure ethical and lawful crawling. | Project principle                                   |
| AI models used for information extraction include Mistral Cloud, Groq, Google Gemini.          | AI integration details                             |
| Disabled nodes provide database maintenance utilities (manual table creation, deletions).      | For advanced maintenance and testing only          |
| The workflow is tagged as WHITE LABEL and versioned V1.0 under the "URL Officer" project name. | Versioning and labeling                             |
| The workflow uses Postgres for caching robots.txt and forbidden URLs tables efficiently.       | Database dependency                                 |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---