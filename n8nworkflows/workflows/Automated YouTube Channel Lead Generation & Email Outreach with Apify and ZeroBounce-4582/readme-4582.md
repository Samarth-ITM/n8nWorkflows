Automated YouTube Channel Lead Generation & Email Outreach with Apify and ZeroBounce

https://n8nworkflows.xyz/workflows/automated-youtube-channel-lead-generation---email-outreach-with-apify-and-zerobounce-4582


# Automated YouTube Channel Lead Generation & Email Outreach with Apify and ZeroBounce

### 1. Workflow Overview

This workflow automates the monitoring of SEO keyword rankings by integrating data from Airtable and the Firecrawl API, then updates rankings and notifies the team via Slack if any changes occur. It is designed for SEO teams, content managers, and digital marketers who want to keep track of keyword performance without manual intervention.

The workflow is logically divided into three main blocks:

- **1.1 Fetch and Merge Data:** Triggered by Airtable record changes, it fetches keywords and related metadata from Airtable, queries the Firecrawl API for updated ranking data, and merges these datasets for comparison.
- **1.2 Compare & Update Airtable Record:** Compares the current ranking from Airtable with the new rank from Firecrawl using a code node, then updates Airtable if there is a rank change.
- **1.3 Conditional Alert:** Checks if the rank has changed and, if so, sends a Slack notification; otherwise, the workflow ends without action.

---

### 2. Block-by-Block Analysis

#### 1.1 Fetch and Merge Data

**Overview:**  
This block initiates the workflow by detecting new or updated keywords in Airtable. It fetches the relevant keyword data, then calls the Firecrawl API to retrieve updated ranking information and merges both data sources for the next processing step.

**Nodes Involved:**  
- Fetch Keywords from Airtable  
- Check Rank via Firecrawl  
- Combine Airtable + Firecrawl Result  

**Node Details:**

- **Fetch Keywords from Airtable**  
  - *Type:* Airtable Trigger  
  - *Role:* Starts the workflow when a record with a keyword is created or updated in Airtable.  
  - *Configuration:* Monitors a specific Airtable base and table, triggered every minute on changes to the `Keyword` field. Fetches fields: `Keyword`, `Target URL`, and `Current Rank`.  
  - *Expressions:* Uses Airtable URLs and personal access token credentials for authentication.  
  - *Input:* Triggered by Airtable changes.  
  - *Output:* Emits keyword data for downstream nodes.  
  - *Edge cases:* Potential API authentication failure, rate limits, or missing fields in Airtable records.

- **Check Rank via Firecrawl**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Firecrawl's SERP API to retrieve the latest keyword ranking results.  
  - *Configuration:*  
    - URL: `https://api.firecrawl.dev/serp`  
    - Method: POST  
    - Body Parameters: Sends `query` with the keyword, `country` as `us`, and `language` as `en`.  
    - Header: Authorization Bearer token required (placeholder `YOUR_API_KEY` to be replaced with a valid API key).  
  - *Expressions:* Uses `={{ $json.fields.Keyword }}` to dynamically pass the keyword from Airtable data.  
  - *Input:* Receives keyword data from Airtable Trigger.  
  - *Output:* Returns an array of search results with URLs and titles.  
  - *Edge cases:* API key invalid or missing, network timeouts, malformed response handling.

- **Combine Airtable + Firecrawl Result**  
  - *Type:* Merge  
  - *Role:* Joins the original Airtable data and Firecrawl results into a single unified item for comparison.  
  - *Configuration:* Default merge settings (no special mode specified).  
  - *Input:* Receives data from both `Fetch Keywords from Airtable` and `Check Rank via Firecrawl`.  
  - *Output:* Emits combined data for the comparison node.  
  - *Edge cases:* Missing data from either source could cause incomplete merges.

---

#### 1.2 Compare & Update Airtable Record

**Overview:**  
This block compares the existing rank with the newly fetched rank and updates the Airtable record if a change is detected.

**Nodes Involved:**  
- Compare Ranks  
- Update Airtable Record  

**Node Details:**

- **Compare Ranks**  
  - *Type:* Code (JavaScript)  
  - *Role:* Compares the `Current Rank` from Airtable with the new rank found in Firecrawl results and flags if the rank has changed.  
  - *Configuration:*  
    - Extracts the search results array and Airtable fields.  
    - Searches for the target URL in the top 10 results to find the current rank position.  
    - Returns an object containing keyword, old and new ranks, rank change flag, and additional metadata.  
  - *Expressions:* Custom JS code that handles defensive checks and data parsing.  
  - *Input:* Combined data from previous merge node.  
  - *Output:* Emits an object with comparison results and rank change flag.  
  - *Edge cases:* No results found, target URL not in top 10, missing or incorrect data fields.

- **Update Airtable Record**  
  - *Type:* Airtable  
  - *Role:* Updates the original Airtable record with the new rank if it has changed.  
  - *Configuration:*  
    - Uses the record ID from the comparison output to identify the row.  
    - Updates the `Current Rank` field with the new rank.  
  - *Input:* Receives rank comparison data.  
  - *Output:* Passes data to the next conditional check.  
  - *Credentials:* Uses Airtable Personal Access Token with appropriate permissions.  
  - *Edge cases:* Airtable API errors, record lock or permission issues.

---

#### 1.3 Conditional Alert

**Overview:**  
This block decides whether to notify the team via Slack based on if the keyword rank has changed.

**Nodes Involved:**  
- Check if Rank Changed  
- Send Slack Notification  
- No Operation, do nothing  

**Node Details:**

- **Check if Rank Changed**  
  - *Type:* If  
  - *Role:* Evaluates whether the rank has changed by comparing the `current_rank` and `new_rank`.  
  - *Configuration:* Checks if these two numeric values are not equal; outputs `true` or `false` branches accordingly.  
  - *Input:* Receives data from Airtable update node.  
  - *Output:*  
    - `true`: proceeds to Slack notification.  
    - `false`: proceeds to a no-op node to end the workflow gracefully.  
  - *Edge cases:* Data type mismatch, missing rank values.

- **Send Slack Notification**  
  - *Type:* Slack  
  - *Role:* Sends a message to a Slack channel notifying about the rank change.  
  - *Configuration:*  
    - Uses a webhook ID configured in Slack credentials.  
    - Posts to a specific Slack channel with a message containing keyword, target URL, new rank, and context.  
  - *Input:* From the true branch of the If node.  
  - *Credentials:* Slack OAuth or API token with posting permissions.  
  - *Edge cases:* Slack API rate limits, invalid channel ID, message formatting errors.

- **No Operation, do nothing**  
  - *Type:* NoOp  
  - *Role:* Placeholder node to terminate the workflow path when no rank change occurs.  
  - *Input:* From the false branch of the If node.  
  - *Output:* Ends the workflow without actions.

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                              | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                  |
|------------------------------|-----------------------|----------------------------------------------|----------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| Fetch Keywords from Airtable  | Airtable Trigger      | Trigger workflow on Airtable keyword changes | None                             | Check Rank via Firecrawl, Combine Airtable + Firecrawl Result | See "Section 1: Fetch and Merge Data" sticky note explaining purpose and fields.                              |
| Check Rank via Firecrawl      | HTTP Request          | Retrieve latest keyword rank data from Firecrawl API | Fetch Keywords from Airtable     | Combine Airtable + Firecrawl Result | As above.                                                                                                    |
| Combine Airtable + Firecrawl Result | Merge             | Merge Airtable data and Firecrawl API results | Fetch Keywords from Airtable, Check Rank via Firecrawl | Compare Ranks                  | As above.                                                                                                    |
| Compare Ranks                | Code                  | Compare current and new ranks, determine changes | Combine Airtable + Firecrawl Result | Update Airtable Record          | See "Section 2: Compare & Update Airtable Record" sticky note detailing logic and fields updated.             |
| Update Airtable Record        | Airtable              | Update Airtable record with new rank         | Compare Ranks                    | Check if Rank Changed           | As above.                                                                                                    |
| Check if Rank Changed         | If                    | Check if rank has changed to branch workflow  | Update Airtable Record           | Send Slack Notification, No Operation, do nothing | See "Section 3: Conditional Alert" sticky note describing decision logic and notification.                   |
| Send Slack Notification       | Slack                 | Notify team on Slack about rank changes       | Check if Rank Changed (true branch) | None                          | As above.                                                                                                    |
| No Operation, do nothing      | NoOp                  | End workflow path if no rank change            | Check if Rank Changed (false branch) | None                          | As above.                                                                                                    |
| Sticky Note                  | Sticky Note           | Documentation notes                            | None                             | None                           | Multiple sticky notes provide detailed section descriptions and overall workflow purpose and instructions. |
| Sticky Note1                 | Sticky Note           | Documentation notes                            | None                             | None                           | As above.                                                                                                    |
| Sticky Note2                 | Sticky Note           | Documentation notes                            | None                             | None                           | As above.                                                                                                    |
| Sticky Note4                 | Sticky Note           | Overall workflow overview                       | None                             | None                           | Provides high-level description, purpose, and visual flow overview with links to resources.                  |
| Sticky Note9                 | Sticky Note           | Support & contact info                          | None                             | None                           | Contains contact emails and links to YouTube and LinkedIn resources.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Trigger Node**  
   - Type: Airtable Trigger  
   - Configure: Set Base ID and Table ID for your Airtable base containing keyword data.  
   - Trigger on changes to the `Keyword` field, polling every minute.  
   - Authenticate using a Personal Access Token with read permissions.  
   - Fields to fetch: `Keyword`, `Target URL`, `Current Rank`.  

2. **Create HTTP Request Node - Firecrawl API**  
   - Type: HTTP Request  
   - Connect input from Airtable Trigger.  
   - Configure:  
     - URL: `https://api.firecrawl.dev/serp`  
     - Method: POST  
     - Body Parameters: JSON with keys:  
       - `query`: expression `={{ $json.fields.Keyword }}`  
       - `country`: `"us"`  
       - `language`: `"en"`  
     - Headers: Authorization Bearer token with your Firecrawl API key.  
   - Set response to JSON.  

3. **Create Merge Node**  
   - Type: Merge  
   - Connect two inputs:  
     - First: Airtable Trigger output  
     - Second: Firecrawl HTTP Request output  
   - Use default merge mode to combine both inputs into one data stream.  

4. **Create Code Node - Compare Ranks**  
   - Type: Code  
   - Connect input from Merge node.  
   - Paste the following JavaScript logic:  
     ```js
     const resultItem = items.find(item => Array.isArray(item.json.results));
     const dataItem = items.find(item => item.json.fields && item.json.fields["Target URL"]);

     if (!resultItem || !dataItem) {
       throw new Error("Missing result or Airtable data");
     }

     const results = resultItem.json.results;
     const fields = dataItem.json.fields;

     const targetUrl = fields["Target URL"];
     const keyword = fields["Keyword"];

     let rank = null;

     for (let i = 0; i < results.length; i++) {
       const resultUrl = results[i].url?.toLowerCase() || '';
       if (resultUrl.includes(targetUrl.toLowerCase())) {
         rank = i + 1;
         break;
       }
     }

     return [
       {
         json: {
           keyword,
           target_url: targetUrl,
           current_rank: fields["Current Rank"],
           new_rank: rank !== null ? rank : "Not in Top 10",
           rank_changed: rank !== null && rank !== fields["Current Rank"],
           notes: fields["Notes"] || "",
           airtable_id: dataItem.json.id,
           raw_results: results
         }
       }
     ];
     ```  
   - This code searches the Firecrawl results for the target URL and determines rank changes.

5. **Create Airtable Node - Update Record**  
   - Type: Airtable  
   - Connect input from Code node.  
   - Configure:  
     - Use same Airtable Base and Table as the trigger.  
     - Set operation to "Update".  
     - Use `id` from code output (`airtable_id`) to identify the record.  
     - Update `Current Rank` field with `new_rank` from code output.  
   - Authenticate with same Airtable credentials.

6. **Create If Node - Check if Rank Changed**  
   - Type: If  
   - Connect input from Airtable Update node.  
   - Condition: Check if `current_rank` (from code output) is not equal to `new_rank`.  
   - If **true**, proceed to Slack notification.  
   - If **false**, end workflow.

7. **Create Slack Node - Send Notification**  
   - Type: Slack  
   - Connect input from If node's true branch.  
   - Configure:  
     - Select Slack channel to post in.  
     - Compose message containing keyword, target URL, new rank, and optionally timestamp or notes.  
   - Authenticate with Slack API credentials having posting permissions.

8. **Create No Operation Node**  
   - Type: NoOp  
   - Connect input from If node's false branch.  
   - This node ends the workflow silently when no rank change is detected.

9. **Link the Nodes in Order:**  
   `Airtable Trigger` → `Check Rank via Firecrawl` → `Merge` → `Compare Ranks` → `Update Airtable Record` → `Check if Rank Changed` → [True → `Send Slack Notification`] / [False → `No Operation`]

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow automates keyword rank monitoring, updates Airtable records, and sends Slack notifications on rank changes.                       | General project overview                                                                                   |
| Firecrawl API documentation and usage: https://api.firecrawl.dev                                                                            | Used for fetching keyword ranking data                                                                    |
| Airtable Personal Access Token required with read/write permissions for targeted base and table                                           | Credential setup instructions                                                                              |
| Slack webhook or OAuth2 token with permission to post messages in selected channel                                                          | Credential setup instructions                                                                              |
| Contact for workflow assistance: Yaron@nofluff.online                                                                                      | Support email                                                                                              |
| Helpful resources: YouTube channel https://www.youtube.com/@YaronBeen/videos and LinkedIn https://www.linkedin.com/in/yaronbeen/           | Additional learning and support resources                                                                  |
| The workflow is designed to run automatically with minimal manual intervention, ideal for SEO teams and digital agencies.                   | Strategic usage note                                                                                         |
| Replace placeholder API keys and tokens with valid credentials before activating the workflow                                               | Important setup reminder                                                                                     |

---

This documentation fully describes the workflow’s structure, logic, configuration, and provides a clear blueprint for reproduction or modification. It also highlights potential errors and edge cases to anticipate during operation.