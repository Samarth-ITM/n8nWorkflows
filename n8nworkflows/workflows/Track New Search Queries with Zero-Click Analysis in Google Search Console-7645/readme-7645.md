Track New Search Queries with Zero-Click Analysis in Google Search Console

https://n8nworkflows.xyz/workflows/track-new-search-queries-with-zero-click-analysis-in-google-search-console-7645


# Track New Search Queries with Zero-Click Analysis in Google Search Console

### 1. Workflow Overview

This workflow is designed to track **new search queries** appearing in Google Search Console for a specified website domain by comparing search analytics data between two date ranges. It identifies queries that had no impressions in the past period but have impressions in the recent period (last 7 days by default). The workflow then categorizes these new queries into those that have received clicks and those that have not (zero-click queries). This helps SEO professionals discover **new keyword opportunities** and understand user engagement trends for newly emerging queries.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Google Search Console Data Retrieval and Comparison**: Fetch search analytics comparing two date ranges to find new queries.
- **1.3 Filtering for New Queries with No Past Impressions**: Filter queries that had zero impressions in the past period.
- **1.4 Classification by Click Activity**: Further filter to separate queries into zero-click and has-click groups.
- **1.5 Documentation and Guidance**: Sticky note providing setup instructions and workflow explanation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Entry point of the workflow triggered manually by the user to initiate the data retrieval and analysis process.

- **Nodes Involved:**  
  - Manual Start

- **Node Details:**

  - **Manual Start**  
    - Type: `Start` node  
    - Role: Initiates workflow execution manually  
    - Configuration: Default, no parameters  
    - Input: None (trigger node)  
    - Output: Triggers the next node "Compare search analytics between two date ranges"  
    - Edge cases: None, but workflow depends on manual execution or external trigger if modified  
    - Version: 1

#### 1.2 Google Search Console Data Retrieval and Comparison

- **Overview:**  
  Queries Google Search Console for search analytics data comparing two defined date ranges: the recent 7 days and a custom past period (August 1, 2024 to August 12, 2025). This comparison identifies keywords with changing impression and click metrics.

- **Nodes Involved:**  
  - Compare search analytics between two date ranges

- **Node Details:**

  - **Compare search analytics between two date ranges**  
    - Type: `Google Search Console` node  
    - Role: Retrieves and compares search analytics data for the specified site over two date ranges  
    - Configuration:  
      - Site URL: `sc-domain:example.com` (Google Search Console domain property)  
      - Operation: `comparePageInsights` (compare data between period A and B)  
      - Date Range A: last 7 days (dynamic)  
      - Date Range B: custom range from August 1, 2024 to August 12, 2025  
      - Comparison Mode: custom  
      - Dimensions: `query` (search queries)  
      - Row limit: 10,000 entries to cover extensive data  
    - Key expressions/variables: Uses fixed date strings for dateRangeB, dynamic last7d for dateRangeA  
    - Input: Triggered by Manual Start  
    - Output: JSON data containing analytics metrics per query with metrics for both periods (impressions and clicks)  
    - Edge cases:  
      - Authentication errors if credentials are invalid or expired  
      - API rate limits or quota exceeded errors  
      - Large data sets causing timeout or incomplete data if row limit exceeded  
    - Version: 1  
    - Credentials: Requires Google Search Console OAuth2 credentials configured

#### 1.3 Filtering for New Queries with No Past Impressions

- **Overview:**  
  Filters out queries that had impressions in the past period (dateRangeB); only passing those queries with zero impressions previously (i.e., truly new queries).

- **Nodes Involved:**  
  - Filter (No Past Impressions)

- **Node Details:**

  - **Filter (No Past Impressions)**  
    - Type: `Filter` node  
    - Role: Checks if `impr_b` (impressions in past period B) equals zero  
    - Configuration: Condition: `$json.impr_b == 0` (loose type validation, case sensitive)  
    - Input: Output from the Google Search Console comparison node  
    - Output: Passes only records meeting condition to next nodes  
    - Edge cases:  
      - Missing or malformed `impr_b` field causes expression failure  
      - Zero impressions could be due to data delay or indexing issues, possibly false positives  
    - Version: 2.2

#### 1.4 Classification by Click Activity

- **Overview:**  
  Splits the new queries into two branches based on whether they have clicks or not in the recent period (dateRangeA). This allows separate handling of zero-click queries and those with clicks.

- **Nodes Involved:**  
  - Zero Click  
  - Has Click

- **Node Details:**

  - **Zero Click**  
    - Type: `Filter` node  
    - Role: Passes queries where clicks in recent period (`clicks_a`) equal zero  
    - Configuration: `$json.clicks_a == 0`  
    - Input: Output of Filter (No Past Impressions)  
    - Output: New queries with impressions but no clicks yet (zero-click queries)  
    - Edge cases: Click count may be zero due to data lag or under-reporting  
    - Version: 2.2

  - **Has Click**  
    - Type: `Filter` node  
    - Role: Passes queries where clicks in recent period (`clicks_a`) are greater than zero  
    - Configuration: `$json.clicks_a > 0`  
    - Input: Output of Filter (No Past Impressions)  
    - Output: New queries with impressions and clicks  
    - Edge cases: Possible minor data discrepancies if clicks are low or borderline  
    - Version: 2.2

#### 1.5 Documentation and Guidance

- **Overview:**  
  Provides a sticky note with a detailed explanation of the workflow, setup instructions, and output descriptions for user clarity.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: `Sticky Note` node  
    - Role: Documentation and user guidance within the workflow editor  
    - Configuration: Markdown content explaining workflow purpose, setup steps, and output interpretation  
    - Input/Output: None (informational only)  
    - Content Highlights:  
      - Purpose: Identify queries with no past impressions but recent impressions  
      - Setup steps: Credential setup, date range adjustment, manual or scheduled run  
      - Outputs: Explanation of zero-click and has-click branches  
    - Version: 1

---

### 3. Summary Table

| Node Name                           | Node Type                 | Functional Role                                   | Input Node(s)                        | Output Node(s)                           | Sticky Note                                                                                                           |
|-----------------------------------|---------------------------|-------------------------------------------------|------------------------------------|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Manual Start                      | Start                     | Manual trigger to start workflow                 | None                               | Compare search analytics between two date ranges |                                                                                                                       |
| Compare search analytics between two date ranges | Google Search Console     | Retrieve and compare search analytics data       | Manual Start                      | Filter (No Past Impressions)             |                                                                                                                       |
| Filter (No Past Impressions)      | Filter                    | Filter queries with zero past impressions        | Compare search analytics between two date ranges | Zero Click, Has Click                  |                                                                                                                       |
| Zero Click                       | Filter                    | Filter queries with zero clicks in recent period | Filter (No Past Impressions)       | None                                    |                                                                                                                       |
| Has Click                       | Filter                    | Filter queries with clicks in recent period      | Filter (No Past Impressions)       | None                                    |                                                                                                                       |
| Sticky Note                     | Sticky Note               | Documentation and workflow guidance               | None                               | None                                    | ## 🔎 Google Search Console - New Keywords (Last 7 Days)\n\nThis workflow identifies queries that had **no impressions in the past** but appeared in the **last 7 days**. It helps you find **new keyword opportunities**.\n\n### Setup Instructions\n1. **Connect Google Search Console** → add your credentials in the `Compare search analytics` node.\n2. (Optional) Adjust the date ranges for comparison.\n3. Run manually or use the `Weekly Trigger` to automate.\n\n### Output\n- **Zero Click** → new queries with impressions but no clicks yet.\n- **Has Click** → new queries with impressions and clicks.\n\nUse these results to optimize your SEO strategy 🚀 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Start Node**  
   - Add a `Start` node from the Core nodes section. This node will manually trigger the workflow.

2. **Add Google Search Console Node**  
   - Add a `Google Search Console` node.  
   - Configure it with your Google Search Console credentials (OAuth2).  
   - Set the following parameters:  
     - **Operation:** `comparePageInsights`  
     - **Site URL Compare:** `sc-domain:example.com` (replace with your actual domain)  
     - **Date Range Mode A:** `last7d` (last 7 days)  
     - **Date Range Mode B:** `custom`  
     - **Start Date B:** `2024-08-01T00:00:00` (adjust as needed)  
     - **End Date B:** `2025-08-12T00:00:00` (adjust as needed)  
     - **Compare Mode:** `custom`  
     - **Dimensions Compare:** `query`  
     - **Row Limit Compare:** 10,000 (to capture sufficient data)  
   - Connect the `Manual Start` node output to this node’s input.

3. **Create Filter (No Past Impressions) Node**  
   - Add a `Filter` node.  
   - Configure with condition: `$json.impr_b == 0` to pass only queries with zero impressions in the past period.  
   - Connect the output of the Google Search Console node to this filter’s input.

4. **Create Zero Click Filter Node**  
   - Add another `Filter` node.  
   - Configure condition: `$json.clicks_a == 0` to find queries with no clicks in the recent period.  
   - Connect the output of the "Filter (No Past Impressions)" node to this filter’s input.

5. **Create Has Click Filter Node**  
   - Add another `Filter` node.  
   - Configure condition: `$json.clicks_a > 0` to find queries with clicks in the recent period.  
   - Connect the output of the "Filter (No Past Impressions)" node to this filter’s input.

6. **Add Sticky Note for Documentation**  
   - Add a `Sticky Note` node to the canvas.  
   - Paste the following content (Markdown supported):  
     ```
     ## 🔎 Google Search Console - New Keywords (Last 7 Days)

     This workflow identifies queries that had **no impressions in the past** but appeared in the **last 7 days**. It helps you find **new keyword opportunities**.

     ### Setup Instructions
     1. **Connect Google Search Console** → add your credentials in the `Compare search analytics` node.
     2. (Optional) Adjust the date ranges for comparison.
     3. Run manually or use the `Weekly Trigger` to automate.

     ### Output
     - **Zero Click** → new queries with impressions but no clicks yet.
     - **Has Click** → new queries with impressions and clicks.

     Use these results to optimize your SEO strategy 🚀
     ```
   - Place the sticky note conveniently near the start or comparison nodes.

7. **Credential Setup**  
   - In the Google Search Console node, configure OAuth2 credentials with access to the target site’s Search Console data.  
   - Ensure token refresh is set up to avoid auth errors.

8. **Testing and Running**  
   - Run the workflow manually from the `Manual Start`.  
   - Verify outputs of filters are as expected — zero-click and has-click queries.

9. **Optional Automation**  
   - Add a `Cron` node or `Interval` node to trigger this workflow weekly or at desired intervals to keep data updated automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow identifies new keyword opportunities by detecting queries with no prior impressions but recent impressions and clicks. | Workflow purpose and SEO optimization strategy                                                                |
| Adjust date ranges in the Google Search Console node to customize analysis periods.                                            | Customization of date ranges                                                                                    |
| Requires Google Search Console OAuth2 credentials with appropriate permissions.                                                | Credential setup instructions                                                                                   |
| Use the zero-click and has-click outputs to prioritize SEO efforts: zero-click may indicate opportunity for better CTR.       | SEO strategy insights                                                                                           |
| For automation, consider adding a weekly trigger using Cron node.                                                             | Automation best practice                                                                                        |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.