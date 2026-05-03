Automated YouTube Channel Lead Generation & Email Outreach with Apify and ZeroBounce

https://n8nworkflows.xyz/workflows/automated-youtube-channel-lead-generation---email-outreach-with-apify-and-zerobounce-4579


# Automated YouTube Channel Lead Generation & Email Outreach with Apify and ZeroBounce

### 1. Workflow Overview

This workflow automates the monitoring of keyword rankings for SEO purposes by integrating Airtable as a keyword database, Firecrawl API as a rank-checking service, and Slack for notification. It continuously fetches keywords from Airtable, retrieves their latest ranking data from Firecrawl, compares it to stored ranks, updates Airtable accordingly, and alerts a Slack channel if rank changes occur.

Logical blocks are organized as follows:

- **1.1 Fetch and Merge Data**  
  Trigger keyword updates from Airtable, query Firecrawl API for ranking data, and merge these data sources for analysis.

- **1.2 Compare and Update Airtable Record**  
  Compare current and new rank values, determine if changes occurred, and update Airtable records with new rank data.

- **1.3 Conditional Notification**  
  Check if the rank changed and send a notification to Slack if true; otherwise, stop the workflow gracefully.

Supporting nodes include sticky notes for documentation and a no-operation node to end the workflow when no action is needed.

---

### 2. Block-by-Block Analysis

#### 1.1 Fetch and Merge Data

**Overview:**  
This block initiates the workflow by detecting new or updated keyword entries in Airtable, querying the Firecrawl API to obtain current search rank data, and merging both data sets for further processing.

**Nodes Involved:**  
- Fetch Keywords from Airtable  
- Check Rank via Firecrawl  
- Combine Airtable + Firecrawl Result

**Node Details:**

- **Fetch Keywords from Airtable**  
  - Type: Airtable Trigger  
  - Role: Watches specified Airtable base and table for new or updated records containing keywords.  
  - Configuration: Polls every minute; triggers on changes in the field `Keyword`. Fetches fields `Keyword`, `Target URL`, and `Current Rank`. Uses Airtable Personal Access Token credential.  
  - Input: None (trigger node)  
  - Output: Emits records containing keyword data to downstream nodes.  
  - Failures: Possible due to API rate limits, authentication errors, or malformed data.  
  - Version: 1

- **Check Rank via Firecrawl**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Firecrawl API to fetch ranking data for the given keyword.  
  - Configuration: POST to `https://api.firecrawl.dev/serp` with JSON body including `query` (keyword), `country` (US), `language` (EN). Requires Bearer token in Authorization header (replace `YOUR_API_KEY`).  
  - Input: Keyword from Airtable trigger node.  
  - Output: JSON response with search results including URLs and rankings.  
  - Failures: Authentication errors if API key is invalid; network timeouts; malformed responses; API quota exceeded.  
  - Version: 4.2

- **Combine Airtable + Firecrawl Result**  
  - Type: Merge  
  - Role: Joins the original Airtable keyword data and the Firecrawl API response into a single item for comparison.  
  - Configuration: Uses default merge mode (likely "combine") to unify data from two inputs.  
  - Input: Two inputs — Airtable data and Firecrawl response.  
  - Output: Combined JSON object containing both sources.  
  - Failures: Merge may fail if inputs are missing or malformed.  
  - Version: 3.1

---

#### 1.2 Compare and Update Airtable Record

**Overview:**  
This block compares the current rank stored in Airtable with the new rank obtained from Firecrawl. If a change is detected, it updates the Airtable record accordingly.

**Nodes Involved:**  
- Compare Ranks  
- Update Airtable Record

**Node Details:**

- **Compare Ranks**  
  - Type: Code (JavaScript)  
  - Role: Parses merged data, finds the rank of the target URL within Firecrawl results, and compares it to the current rank from Airtable. Sets flags and prepares output for updating.  
  - Configuration: Custom JS code:
    - Finds Firecrawl results array and Airtable data object.  
    - Compares target URL (case-insensitive match within results).  
    - Determines new rank or "Not in Top 10" if not found.  
    - Calculates if rank changed (boolean).  
    - Outputs a structured JSON including keyword, ranks, flags, notes, Airtable record ID, and raw results.  
  - Input: Combined Airtable + Firecrawl data.  
  - Output: JSON with comparison results and metadata for update.  
  - Failures: Throws error if required fields missing or array structure unexpected. Expression failures if JSON paths change.  
  - Version: 2

- **Update Airtable Record**  
  - Type: Airtable  
  - Role: Updates specific Airtable record with the new rank and other optional fields.  
  - Configuration: Uses record ID from comparison output to update the field `Current Rank` to the new rank. Airtable base and table specified; uses Airtable token credential.  
  - Input: Output from Compare Ranks node.  
  - Output: Confirmation of update (record details).  
  - Failures: API errors, permission issues, invalid record ID, data conversion failures.  
  - Version: 2.1

---

#### 1.3 Conditional Notification

**Overview:**  
This block conditionally sends a Slack notification if the rank has changed, otherwise ends the workflow with no operation.

**Nodes Involved:**  
- Check if Rank Changed  
- Send Slack Notification  
- No Operation, do nothing

**Node Details:**

- **Check if Rank Changed**  
  - Type: If  
  - Role: Evaluates boolean flag `rankChanged` from the comparison output to decide if notification is needed.  
  - Configuration: Condition checks if previous `current_rank` does not equal `new_rank`.  
  - Input: From Update Airtable Record output.  
  - Output: Two branches — true (rank changed) and false (no change).  
  - Failures: Expression errors if input data missing or malformed.  
  - Version: 2.2

- **Send Slack Notification**  
  - Type: Slack  
  - Role: Posts a message to a specific Slack channel notifying about the rank change and details.  
  - Configuration: Uses Slack OAuth credential; posts to channel ID `C08TTV0CC3E`. Message text can be customized (in this workflow, it is currently "hi" placeholder, but should be replaced by dynamic content).  
  - Input: True branch from If node.  
  - Output: Slack API response confirming message posted.  
  - Failures: Slack API rate limits, permission errors, invalid channel ID.  
  - Version: 2.3

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Ends workflow gracefully when no update or notification is needed.  
  - Configuration: No parameters.  
  - Input: False branch from If node.  
  - Output: None.  
  - Failures: None.  
  - Version: 1

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                       | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                   |
|------------------------------|----------------------|------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Fetch Keywords from Airtable  | Airtable Trigger     | Trigger on keyword updates         | None                        | Check Rank via Firecrawl, Combine Airtable + Firecrawl Result | Section 1: Fetch and Merge Data explains this is the starting point fetching keywords        |
| Check Rank via Firecrawl      | HTTP Request         | Query Firecrawl API for rank data  | Fetch Keywords from Airtable | Combine Airtable + Firecrawl Result | Section 1: Fetch and Merge Data describes this API call                                     |
| Combine Airtable + Firecrawl Result | Merge            | Merge Airtable data and Firecrawl results | Fetch Keywords from Airtable, Check Rank via Firecrawl | Compare Ranks                | Section 1: Merge prepares data for comparison                                               |
| Compare Ranks                | Code                 | Determine rank changes and flags   | Combine Airtable + Firecrawl Result | Update Airtable Record       | Section 2: Compare & Update Airtable Record describes rank comparison and flag setting       |
| Update Airtable Record        | Airtable             | Update rank data in Airtable       | Compare Ranks               | Check if Rank Changed        | Section 2: Update Airtable Record updates rank in Airtable                                  |
| Check if Rank Changed         | If                   | Branch based on rank change flag   | Update Airtable Record       | Send Slack Notification, No Operation | Section 3: Conditional Alert checks and routes notification                                 |
| Send Slack Notification       | Slack                | Notify Slack channel of rank change| Check if Rank Changed (true) | None                        | Section 3: Sends Slack alert if rank changed                                               |
| No Operation, do nothing      | NoOp                 | Ends workflow when no change       | Check if Rank Changed (false) | None                        | Section 3: Ends workflow gracefully if no rank change                                      |
| Sticky Note                  | Sticky Note          | Documentation                      | None                        | None                        | Contains detailed explanations for Section 1                                              |
| Sticky Note1                 | Sticky Note          | Documentation                      | None                        | None                        | Contains detailed explanations for Section 2                                              |
| Sticky Note2                 | Sticky Note          | Documentation                      | None                        | None                        | Contains detailed explanations for Section 3                                              |
| Sticky Note9                 | Sticky Note          | Workflow assistance contact info   | None                        | None                        | Provides contact and external resources                                                    |
| Sticky Note4                 | Sticky Note          | Full workflow purpose & overview   | None                        | None                        | Contains an extensive overview and conceptual explanation of the entire workflow           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Trigger Node**  
   - Name: `Fetch Keywords from Airtable`  
   - Type: Airtable Trigger  
   - Configure with your Airtable Base ID and Table ID (corresponding to your keyword list).  
   - Trigger on field changes for `Keyword` field, polling every minute.  
   - Use Airtable Personal Access Token credential with read access.  
   - Output fields: `Keyword`, `Target URL`, `Current Rank`.

2. **Create HTTP Request Node**  
   - Name: `Check Rank via Firecrawl`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/serp`  
   - Headers: Authorization: Bearer YOUR_API_KEY (replace with valid Firecrawl API key).  
   - Body (JSON):  
     ```json
     {
       "query": "={{ $json.fields.Keyword }}",
       "country": "us",
       "language": "en"
     }
     ```  
   - Connect input from `Fetch Keywords from Airtable`.  
   - Set to send body and headers.  
   - Set version 4.2 or latest.

3. **Create Merge Node**  
   - Name: `Combine Airtable + Firecrawl Result`  
   - Type: Merge  
   - Connect two inputs:  
     - First from `Fetch Keywords from Airtable` (original data)  
     - Second from `Check Rank via Firecrawl` (API results)  
   - Use default merge mode (combine).  
   - Output combined data for comparison.

4. **Create Code Node**  
   - Name: `Compare Ranks`  
   - Type: Code  
   - Paste the provided JavaScript code that:  
     - Extracts Firecrawl results and Airtable data.  
     - Finds rank of target URL in Firecrawl results.  
     - Compares current rank vs new rank.  
     - Adds `rank_changed` boolean flag.  
     - Outputs keyword, ranks, notes, Airtable record ID, and raw results.  
   - Connect input from `Combine Airtable + Firecrawl Result`.

5. **Create Airtable Node**  
   - Name: `Update Airtable Record`  
   - Type: Airtable  
   - Configure to update record by ID using: `={{ $json.airtable_id }}`  
   - Update field `Current Rank` with value `={{ $json.new_rank }}`  
   - Use same Airtable credential as trigger.  
   - Connect input from `Compare Ranks`.

6. **Create If Node**  
   - Name: `Check if Rank Changed`  
   - Type: If  
   - Condition: Check if `={{ $('Compare Ranks').item.json.current_rank }}` !== `={{ $('Compare Ranks').item.json.new_rank }}`  
   - Connect input from `Update Airtable Record`.

7. **Create Slack Node**  
   - Name: `Send Slack Notification`  
   - Type: Slack  
   - Configure Slack OAuth2 credential.  
   - Set channel ID to your target Slack channel (e.g., `C08TTV0CC3E`).  
   - Customize message text to include keyword, target URL, new rank, and timestamp dynamically.  
   - Connect input from `Check if Rank Changed` true branch.

8. **Create No Operation Node**  
   - Name: `No Operation, do nothing`  
   - Type: NoOp  
   - Connect input from `Check if Rank Changed` false branch.

9. **Finalize Connections**  
   - Connect nodes in order:  
     `Fetch Keywords from Airtable` → `Check Rank via Firecrawl` → `Combine Airtable + Firecrawl Result` → `Compare Ranks` → `Update Airtable Record` → `Check if Rank Changed` →  
     - True → `Send Slack Notification`  
     - False → `No Operation, do nothing`.

10. **Test the Workflow**  
   - Ensure API keys and credentials are valid.  
   - Verify Airtable base and table IDs match your setup.  
   - Test with sample keywords to confirm ranks update and Slack notifications trigger correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| For support or questions, contact: Yaron@nofluff.online                                                                                         | Workflow Assistance sticky note                                                    |
| Tips and tutorials available at YouTube: https://www.youtube.com/@YaronBeen/videos                                                            | Workflow Assistance sticky note                                                    |
| LinkedIn profile for expert insights: https://www.linkedin.com/in/yaronbeen/                                                                   | Workflow Assistance sticky note                                                    |
| This workflow automates SEO rank tracking and notification, ideal for SEO teams, content managers, digital agencies, and growth hackers.       | Sticky note with full workflow overview                                            |
| Firecrawl API documentation needed to understand API limits and response structure: https://api.firecrawl.dev (not included here)              | Essential for integrating and troubleshooting HTTP Request node                    |
| Slack API rate limits and permissions should be reviewed to avoid notification failures                                                         | Relevant for Slack node configuration                                              |
| Airtable API quotas and field naming conventions must be consistent with this workflow for error-free operation                                 | Relevant for Airtable trigger and update nodes                                    |

---

**Disclaimer:**  
The content above is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material. All handled data are legal and public.