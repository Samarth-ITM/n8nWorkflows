Weekly SEO Watchlist Audit to Google Sheets with Gemini and Decodo

https://n8nworkflows.xyz/workflows/weekly-seo-watchlist-audit-to-google-sheets-with-gemini-and-decodo-9982


# Weekly SEO Watchlist Audit to Google Sheets with Gemini and Decodo

### 1. Workflow Overview

This workflow automates a **Weekly SEO Watchlist Audit** for a list of URLs stored in Google Sheets, leveraging AI-powered analysis via **Google Gemini** and **Decodo** services. It is designed for content and SEO teams aiming to monitor key web pages regularly, receive concise SEO audit summaries, and have detailed issues tracked in structured Google Sheets tabs for easy prioritization and action.

The workflow is logically organized into these main blocks:

- **1.1 Scheduled Trigger & Configuration**  
  Starts the workflow on a weekly schedule and loads configuration parameters such as the Google Sheet ID.

- **1.2 Input Retrieval**  
  Reads the list of URLs to audit from a configured Google Sheets input tab.

- **1.3 URL Processing Loop**  
  Processes URLs in batches, iterating through each URL sequentially.

- **1.4 Page Content Fetching (Decodo)**  
  Uses Decodo API to fetch the webpage content and metadata (status code, markdown content).

- **1.5 AI SEO Audit Generation (Google Gemini + Output Parser)**  
  Applies Google Gemini AI to analyze the fetched content and generate a structured SEO audit JSON. An output parser enforces strict schema compliance.

- **1.6 Data Transformation for Google Sheets**  
  Transforms the AI-generated JSON into two sets of rows: a summary row per URL and an exploded list of individual SEO issues.

- **1.7 Data Writing to Google Sheets**  
  Appends the summary and detailed issues to their respective tabs in Google Sheets.

- **1.8 Workflow Completion and Loop Continuation**  
  Joins data streams and controls batch loop continuation until all URLs are processed, then ends.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Scheduled Trigger & Configuration

- **Overview:**  
  Initiates the workflow on a weekly recurrence and sets key configuration variables such as the Google Sheet ID.

- **Nodes Involved:**  
  - Trigger: Weekly  
  - Set: Configure Workflow  
  - Sticky Note1 (interval reminder)  
  - Sticky Note2 (sheet_id reminder)  

- **Node Details:**

  - **Trigger: Weekly**  
    - Type: Schedule Trigger  
    - Configuration: Triggers every week (interval set to weeks)  
    - Input: None  
    - Output: Triggers Set node  
    - Failure modes: None expected; if n8n scheduler fails, no execution occurs.

  - **Set: Configure Workflow**  
    - Type: Set Node  
    - Purpose: Stores the Google Sheet ID in a variable `sheet_id` (default placeholder "change me!")  
    - Inputs: From Trigger node  
    - Outputs: Sends configuration downstream  
    - Edge cases: Misconfiguration (missing or invalid sheet_id) will cause Google Sheets nodes to fail.

  - **Sticky Notes:**  
    Provide user instructions on adjusting schedule and entering sheet ID.

---

#### Block 1.2: Input Retrieval

- **Overview:**  
  Reads the list of URLs from the Google Sheets input tab for processing.

- **Nodes Involved:**  
  - Read Input URLs (Google Sheets)  
  - Sticky Note3 (input sheet URL column reminder)  

- **Node Details:**

  - **Read Input URLs (Google Sheets)**  
    - Type: Google Sheets (Read operation)  
    - Configuration: Reads from sheet with GID=0 (Input tab) within document ID from `sheet_id`  
    - Credentials: Google Sheets OAuth2  
    - Output: JSON array of rows, expects a column named `URL`  
    - Edge cases:  
      - Missing or invalid sheet ID causes read failure  
      - Missing `URL` column leads to empty or invalid data downstream  
      - API rate limits or auth errors possible

---

#### Block 1.3: URL Processing Loop

- **Overview:**  
  Splits input URLs into batches and processes each URL sequentially through the SEO audit steps.

- **Nodes Involved:**  
  - Loop URLs (Split in Batches)  
  - Sticky Note (Workflow overview)  

- **Node Details:**

  - **Loop URLs (Split in Batches)**  
    - Type: Split In Batches  
    - Configuration: Default batch size (1) to process URLs one-by-one  
    - Input: List of URLs from Google Sheets read node  
    - Outputs: Two outputs: one to Decodo fetch, second to Done node (end signal)  
    - Edge cases:  
      - Batch processing errors will stop or skip URLs  
      - Empty input list leads to no iterations  
      - Ensures controlled sequential processing for API rate limits

---

#### Block 1.4: Page Content Fetching (Decodo)

- **Overview:**  
  Fetches webpage content and metadata for each URL using Decodo's content extraction API.

- **Nodes Involved:**  
  - Decodo: Fetch Page Content  

- **Node Details:**

  - **Decodo: Fetch Page Content**  
    - Type: Decodo API Node  
    - Configuration: URL parameter dynamically taken from current batch item (`$json.URL_list`)  
    - Credentials: Decodo API key  
    - Output: JSON containing `content`, `status_code`, `updated_at`, etc.  
    - Error Handling: `onError` set to continueRegularOutput to avoid workflow halt on failures  
    - Retry: Enabled (retryOnFail=true) for transient network issues  
    - Edge cases:  
      - Network or API errors  
      - Empty or malformed content  
      - Rate limits

---

#### Block 1.5: AI SEO Audit Generation (Google Gemini + Output Parser)

- **Overview:**  
  Sends the fetched page content to Google Gemini to generate a strict JSON SEO audit, then parses the output enforcing the schema.

- **Nodes Involved:**  
  - AI Model: Google Gemini  
  - Output Parser: Enforce SEO JSON  
  - Generate SEO Audit  

- **Node Details:**

  - **AI Model: Google Gemini**  
    - Type: LangChain Google Gemini Chat Model  
    - Configuration: Uses Google Palm API credentials  
    - Input: Takes content and metadata from Decodo node  
    - Output: AI text response (JSON audit object)  
    - Edge cases:  
      - API quota limits or auth errors  
      - Model response errors or timeouts  
      - Unexpected output format (mitigated by output parser)

  - **Output Parser: Enforce SEO JSON**  
    - Type: LangChain Output Parser (Structured)  
    - Configuration: Custom JSON schema defined manually, expecting strict structure with fields like url, decodo_score, top_issues, all_issues, etc.  
    - Input: Raw Gemini output text  
    - Output: Parsed JSON conforming to schema  
    - Edge cases:  
      - Parsing failures if AI returns invalid JSON  
      - Missing or malformed fields

  - **Generate SEO Audit**  
    - Type: LangChain Chain LLM Node  
    - Configuration:  
      - Custom prompt instructing AI to return only a valid JSON matching the schema  
      - Uses inputs: url, status code, fetched content, timestamp  
      - Enforces strict output rules to allow downstream processing  
    - Input: Output from Decodo content fetch  
    - Output: Parsed JSON audit object  
    - Edge cases similar to AI model node; ensures output conforms to expected schema

---

#### Block 1.6: Data Transformation for Google Sheets

- **Overview:**  
  Converts the structured audit JSON into two formats: one summary row for the URL, and multiple rows for individual SEO issues.

- **Nodes Involved:**  
  - Transform: Summary Row  
  - Transform: Explode Issues  

- **Node Details:**

  - **Transform: Summary Row**  
    - Type: Code Node (JavaScript)  
    - Configuration: Extracts key summary fields from audit JSON, joins recommended actions into a single string separated by newlines, serializes all issues as JSON string for reference  
    - Input: Audit JSON object  
    - Output: One summary row object suitable for Google Sheets append  
    - Edge cases: Handles missing fields gracefully by providing defaults or empty strings

  - **Transform: Explode Issues**  
    - Type: Code Node (JavaScript)  
    - Configuration: Maps the `all_issues` array into multiple row objects, flattening nested issues into individual rows with relevant fields (id, title, severity, etc.)  
    - Input: Audit JSON object  
    - Output: Multiple issue row objects  
    - Edge cases: Returns empty array if no issues present

---

#### Block 1.7: Data Writing to Google Sheets

- **Overview:**  
  Appends the summary row and all individual issues rows into their respective Google Sheets tabs.

- **Nodes Involved:**  
  - Google Sheets: Append Summary  
  - Google Sheets: Append Issues  
  - Join Summary + Issues  
  - Sticky Note4 (columns match reminder)  

- **Node Details:**

  - **Google Sheets: Append Summary**  
    - Type: Google Sheets Append Operation  
    - Configuration: Appends to tab with GID=2024550449 ("Output") using document ID from configuration node  
    - Column mapping matches summary row fields like URL, Status, Priority, Decodo Score, etc.  
    - Error Handling: `onError` set to continueRegularOutput to avoid workflow halt  
    - Edge cases:  
      - Schema mismatch causes append failure  
      - Auth or quota errors

  - **Google Sheets: Append Issues**  
    - Type: Google Sheets Append Operation  
    - Configuration: Appends to tab with GID=1837304635 ("All Issues") using same document ID  
    - Columns include detailed issue info: Issue ID, Title, Severity, Evidence, Recommendation, etc.  
    - Error Handling: Same as Append Summary  
    - Edge cases: Similar to Append Summary

  - **Join Summary + Issues**  
    - Type: Merge Node  
    - Configuration: Joins the outputs of appending summary and issues to synchronize flow and signal batch completion  
    - Input: From both append nodes  
    - Output: Connects back to Loop node to continue batch processing

---

#### Block 1.8: Workflow Completion and Loop Continuation

- **Overview:**  
  Controls the continuation of the batch loop and terminates the workflow once all URLs are processed.

- **Nodes Involved:**  
  - Done (NoOp)  

- **Node Details:**

  - **Done**  
    - Type: No Operation Node  
    - Purpose: Marks the end of the batch loop iteration on one output branch of the Split In Batches node  
    - Input: From Loop URLs node  
    - Output: None  
    - Edge cases: None

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                            | Input Node(s)                      | Output Node(s)                       | Sticky Note                                            |
|-------------------------------|---------------------------------------|--------------------------------------------|----------------------------------|------------------------------------|--------------------------------------------------------|
| Trigger: Weekly                | Schedule Trigger                      | Starts the workflow weekly                  | None                             | Set: Configure Workflow             | Adjust interval to your reporting cadence (e.g., every Monday 09:00 UTC). |
| Set: Configure Workflow        | Set                                  | Stores sheet_id configuration               | Trigger: Weekly                  | Read Input URLs (Google Sheets)    | Enter sheet_id for Input, Output, and All Issues tabs. |
| Read Input URLs (Google Sheets)| Google Sheets (Read)                  | Reads list of URLs from Google Sheets       | Set: Configure Workflow          | Loop URLs (Split in Batches)        | Make sure your Input tab contains a column named URL.  |
| Loop URLs (Split in Batches)   | Split In Batches                     | Iterates over URLs one-by-one                | Read Input URLs (Google Sheets)  | Decodo: Fetch Page Content, Done    |                                                        |
| Decodo: Fetch Page Content     | Decodo API                          | Fetches page content & metadata              | Loop URLs (Split in Batches)     | Generate SEO Audit                  |                                                        |
| AI Model: Google Gemini        | LangChain Google Gemini Model        | Generates SEO audit JSON                      | Generate SEO Audit (ai_languageModel) | Output Parser: Enforce SEO JSON     |                                                        |
| Output Parser: Enforce SEO JSON| LangChain Output Parser Structured   | Parses AI output enforcing JSON schema       | AI Model: Google Gemini          | Generate SEO Audit (ai_outputParser) |                                                        |
| Generate SEO Audit             | LangChain Chain LLM                  | Sends prompt to AI with content to generate SEO audit | Decodo: Fetch Page Content     | AI Model: Google Gemini             |                                                        |
| Transform: Summary Row         | Code                                | Flattens audit JSON into summary row         | Generate SEO Audit               | Google Sheets: Append Summary       | Ensure all columns match the expected field names in this node. |
| Transform: Explode Issues      | Code                                | Flattens audit JSON array into individual issues rows | Generate SEO Audit           | Google Sheets: Append Issues        |                                                        |
| Google Sheets: Append Summary  | Google Sheets (Append)               | Appends summary data to Google Sheets        | Transform: Summary Row           | Join Summary + Issues               |                                                        |
| Google Sheets: Append Issues   | Google Sheets (Append)               | Appends detailed issues data to Google Sheets | Transform: Explode Issues      | Join Summary + Issues               |                                                        |
| Join Summary + Issues          | Merge                               | Joins append operations to continue batch loop | Google Sheets: Append Summary, Google Sheets: Append Issues | Loop URLs (Split in Batches)       |                                                        |
| Done                          | No Operation                        | Marks end of batch processing                 | Loop URLs (Split in Batches)     | None                               |                                                        |
| Sticky Note                   | Sticky Note                        | Documentation and user guidance               | None                           | None                               | See individual sticky note contents in overview        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every week (interval: weeks, default 1)  
   - Name it `Trigger: Weekly`.

2. **Add a Set node** named `Set: Configure Workflow`  
   - Add a string parameter `sheet_id` with value `"change me!"` (to be replaced with actual Google Sheet ID).  
   - Connect `Trigger: Weekly` output to this node's input.

3. **Add a Google Sheets node** for reading URLs named `Read Input URLs (Google Sheets)`  
   - Operation: Read Rows  
   - Document ID: Use expression to get `={{ $json.sheet_id }}` from previous Set node.  
   - Sheet Name: Use GID=0 or equivalent for input tab.  
   - Credentials: Set with Google Sheets OAuth2 credentials.  
   - Connect `Set: Configure Workflow` output to this node.

4. **Add a Split In Batches node** named `Loop URLs (Split in Batches)`  
   - Batch size: 1 (default)  
   - Connect `Read Input URLs (Google Sheets)` output to this node.

5. **Add a Decodo node** named `Decodo: Fetch Page Content`  
   - Parameter `url`: Use expression `={{ $json.URL_list }}` to get current URL from batch.  
   - Credentials: Set Decodo API credentials.  
   - Configure `onError` to continue on error and enable retry on fail.  
   - Connect `Loop URLs (Split in Batches)` first output to this node.

6. **Add LangChain Chain LLM node** named `Generate SEO Audit`  
   - Paste the provided prompt instructing the AI to generate strict JSON SEO audit.  
   - Use variables in prompt to pass URL, status_code, fetched_at, content from Decodo node output.  
   - Connect Decodo node output to `Generate SEO Audit`.

7. **Add LangChain Google Gemini node** named `AI Model: Google Gemini`  
   - Credentials: Google Palm API credentials.  
   - Connect `Generate SEO Audit` output (ai_languageModel) to this node.

8. **Add LangChain Output Parser node** named `Output Parser: Enforce SEO JSON`  
   - Set schema type as manual with the provided JSON schema for the SEO audit output.  
   - Connect `AI Model: Google Gemini` ai_outputParser output to this node.  
   - Connect this output back to `Generate SEO Audit` (ai_outputParser) input.

9. **Add a Code node** named `Transform: Summary Row`  
   - JavaScript code to extract summary fields from audit JSON, join recommended actions, serialize issues as JSON string (see detailed code in analysis).  
   - Connect `Generate SEO Audit` output to this node.

10. **Add a Code node** named `Transform: Explode Issues`  
    - JavaScript code to map `all_issues` array into individual row objects for each issue.  
    - Connect `Generate SEO Audit` output to this node.

11. **Add two Google Sheets Append nodes:**  
    - `Google Sheets: Append Summary` appending to the summary output tab (GID=2024550449) using `sheet_id` from Set node.  
      Map columns: URL, Status, Priority, Top Issues, Decodo Score, Last Checked, Suggested Meta, Suggested Title, Recommended Actions.  
    - `Google Sheets: Append Issues` appending to the detailed issues tab (GID=1837304635) using same `sheet_id`.  
      Map columns: URL, Issue ID, Title, Severity, Evidence, Recommendation, Impact, Effort, Score Delta, Examples.

12. **Connect `Transform: Summary Row` output to `Google Sheets: Append Summary` input.**  
    Connect `Transform: Explode Issues` output to `Google Sheets: Append Issues` input.

13. **Add a Merge node** named `Join Summary + Issues`  
    - Connect outputs of both Google Sheets Append nodes to this merge node.  
    - Output connects back to `Loop URLs (Split in Batches)` second output to continue batch processing.

14. **Connect the second output of `Loop URLs (Split in Batches)` to a No Operation node** named `Done` to mark workflow completion per batch.

15. **Add sticky notes in the editor** to provide user instructions as per the original workflow (schedule adjustment, sheet_id entry, input URL column requirement, column mapping reminders).

16. **Credential Setup:**  
    - Google Sheets OAuth2 with access to the relevant spreadsheet.  
    - Decodo API key for content fetching.  
    - Google Palm API credentials for Gemini.

17. **Final Checks:**  
    - Replace the placeholder `sheet_id` with your actual Google Sheets document ID.  
    - Ensure input tab has a column named `URL`.  
    - Make sure output tabs have headers matching the mapped columns exactly.  
    - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Sign up for Decodo with a discount via [HERE](https://visit.decodo.com/discount).                         | Decodo API account setup and discounts                           |
| Automatically fetches page content, generates compact SEO audit, and writes per-URL summary and issues. | Workflow overview and benefits for SEO/content teams             |
| The output schema is strictly enforced to ensure consistent data for reporting and further automation.  | AI output parser schema definition                               |
| Adjust the weekly trigger to fit your reporting cadence (e.g., every Monday at 09:00 UTC).               | Scheduling flexibility                                            |
| Input Google Sheet must have a column named `URL`.                                                       | Input data requirement                                           |
| Output tabs must have headers matching the node's mapping exactly (e.g., "URL", "Decodo Score", etc.).  | Google Sheets configuration requirement                          |

---

This completes the comprehensive documentation and reproducible guide for the "Weekly SEO Watchlist Audit to Google Sheets with Gemini and Decodo" workflow.