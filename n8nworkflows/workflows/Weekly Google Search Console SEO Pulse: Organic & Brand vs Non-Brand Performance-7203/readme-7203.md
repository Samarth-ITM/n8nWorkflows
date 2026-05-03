Weekly Google Search Console SEO Pulse: Organic & Brand vs Non-Brand Performance

https://n8nworkflows.xyz/workflows/weekly-google-search-console-seo-pulse--organic---brand-vs-non-brand-performance-7203


# Weekly Google Search Console SEO Pulse: Organic & Brand vs Non-Brand Performance

### 1. Workflow Overview

This workflow automates the weekly extraction, processing, and reporting of Google Search Console (GSC) SEO performance data, with a specific focus on brand versus non-brand query performance. It targets SEO teams or digital marketers who want a structured, comparative weekly pulse on organic search performance segmented by brand-related queries.

The workflow runs every Monday afternoon, comparing two time periods: "2 Weeks Ago" and "Last Week". It pulls raw GSC data, segments clicks into brand and non-brand categories based on predefined brand terms, calculates week-over-week changes, and sends a formatted email summary with key SEO metrics including clicks, impressions, CTR, and average position.

Logical blocks:

- **1.1 Scheduling and Date Definition**: Triggers the workflow weekly and calculates the two date ranges for comparison.
- **1.2 Data Retrieval from Google Search Console**: Pulls raw search query data for both time periods.
- **1.3 Brand vs Non-Brand Filtering & Aggregation**: Separates clicks into brand and non-brand buckets using brand terms.
- **1.4 Data Flattening and Merging**: Prepares data for comparison by flattening and merging time periods.
- **1.5 Performance Table Generation**: Generates an HTML table showing metrics and percent changes between time periods.
- **1.6 Email Dispatch**: Sends the performance summary via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Date Definition

- **Overview:**  
  This block triggers the workflow on a weekly schedule and defines the two date ranges ("2 Weeks Ago" and "Last Week") used for comparative analysis.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Define Weeks  
  - If4

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
    - Role: Initiates the workflow every Monday at 15:00 (3 PM).  
    - Configuration: Weekly interval, trigger day Monday, trigger hour 15.  
    - Inputs: None (trigger node).  
    - Outputs: Passes control to Define Weeks node.  
    - Potential failures: Scheduling misconfiguration or time zone issues.

  - **Define Weeks**  
    - Type: Code (JavaScript) node  
    - Role: Calculates two date ranges relative to the current date:  
      - "2 Weeks Ago": 15 to 9 days before today  
      - "Last Week": 8 to 2 days before today  
    - Configuration: Custom JavaScript code formatting ISO dates.  
    - Inputs: From Schedule Trigger.  
    - Outputs: Array with two objects containing label, startDate, and endDate.  
    - Edge cases: Date calculations could be off if system date/time is incorrect.

  - **If4**  
    - Type: If node (conditional branching)  
    - Role: Routes data based on the label property ("2 Weeks Ago" or otherwise).  
    - Configuration: Checks if label equals "2 Weeks Ago".  
    - Inputs: From Define Weeks node.  
    - Outputs: Two branches for separate handling of the two time periods.  
    - Edge cases: Label mismatch or case sensitivity issues could misroute data.

---

#### 1.2 Data Retrieval from Google Search Console

- **Overview:**  
  Retrieves query-level GSC data for each date range via Google Search Console API calls, separately for both time periods.

- **Nodes Involved:**  
  - Brand/NB Pull (for "2 Weeks Ago")  
  - Brand/NB Pull1 (for "Last Week")  
  - Total Metrics Pull (for "2 Weeks Ago")  
  - Total Metrics Pull1 (for "Last Week")

- **Node Details:**

  - **Brand/NB Pull & Brand/NB Pull1**  
    - Type: HTTP Request node  
    - Role: POST request to GSC API querying search data with dimension "query" and row limit 5000 for the respective date range.  
    - Configuration: JSON body includes startDate and endDate dynamically from input, dimension set to "query".  
    - Credentials: Uses OAuth2 credential for Google Search Console API access.  
    - Inputs: From If4 node branches corresponding to each time period.  
    - Outputs: JSON response with query data rows.  
    - Edge cases: API quota limits, authentication errors, empty or incomplete data, network timeouts.

  - **Total Metrics Pull & Total Metrics Pull1**  
    - Type: HTTP Request node  
    - Role: Pulls total metrics without dimension filters for the respective date range. The body is simpler (no dimensions).  
    - Configuration: Similar to Brand/NB Pull nodes but without dimension "query".  
    - Credentials: Same as above.  
    - Inputs: From If4 node branches.  
    - Outputs: JSON response with aggregated metric data.  
    - Edge cases: Same as above.

---

#### 1.3 Brand vs Non-Brand Filtering & Aggregation

- **Overview:**  
  Processes raw GSC query data to separate clicks into brand and non-brand categories based on a user-defined list of brand terms.

- **Nodes Involved:**  
  - Brand Filter (processes "Last Week" data)  
  - Brand Filter1 (processes "2 Weeks Ago" data)

- **Node Details:**

  - **Brand Filter & Brand Filter1**  
    - Type: Code (JavaScript) node  
    - Role: Iterates over GSC query rows; sums clicks where queries include any brand term vs. those that don't.  
    - Configuration: User must update `brandTerms` array with relevant terms.  
    - Inputs: From Brand/NB Pull1 ("Last Week") and Brand/NB Pull ("2 Weeks Ago") nodes respectively.  
    - Outputs: JSON object with startDate, endDate, label, brandClicks, and nonBrandClicks for the period.  
    - Edge cases: Missing or empty rows, queries without keys, brand term matching errors (case sensitivity, partial matches), no brand terms set.

---

#### 1.4 Data Flattening and Merging

- **Overview:**  
  Flattens the total metrics data to extract the first row and merges brand/non-brand click data with total metrics for each time period, then merges both time periods for comparison.

- **Nodes Involved:**  
  - Flatten9 (for "2 Weeks Ago" total metrics)  
  - Flatten8 (for "Last Week" total metrics)  
  - Merge 2 Weeks Ago  
  - Merge Last Week  
  - Merge Time Periods  
  - Add Brand Name

- **Node Details:**

  - **Flatten9 & Flatten8**  
    - Type: Code node  
    - Role: Extracts the first row from total metrics response and merges it with the corresponding date range info.  
    - Configuration: JavaScript code accesses `$json.rows[0]` and appends startDate/endDate from inputs.  
    - Inputs: From Total Metrics Pull (Flatten9) and Total Metrics Pull1 (Flatten8).  
    - Outputs: Flattened summary metric object per time period.  
    - Edge cases: Empty rows, missing data.

  - **Merge 2 Weeks Ago & Merge Last Week**  
    - Type: Merge node (combine mode, by position)  
    - Role: Combines brand filtered data and flattened total metrics for each time period into a single dataset.  
    - Inputs:  
      - Merge 2 Weeks Ago: Brand Filter1 and Flatten9  
      - Merge Last Week: Brand Filter and Flatten8  
    - Outputs: Combined metrics for each time period.  
    - Edge cases: Data misalignment if inputs have differing lengths or missing data.

  - **Merge Time Periods**  
    - Type: Merge node  
    - Role: Combines the two time periods' data sets into one array for final processing.  
    - Inputs: Merge 2 Weeks Ago and Merge Last Week nodes.  
    - Outputs: Array with both time periods' data objects.  
    - Edge cases: Data ordering issues, missing periods.

  - **Add Brand Name**  
    - Type: Code node  
    - Role: Injects a static brand name (placeholder "BRAND NAME") and bundles all collected data under a "data" field for downstream processing.  
    - Inputs: From Merge Time Periods.  
    - Outputs: JSON object with brand name and array of data objects.  
    - Configuration: User must update brand name accordingly.  
    - Edge cases: Hardcoded brand name could be overlooked in multi-brand environments.

---

#### 1.5 Performance Table Generation

- **Overview:**  
  Creates an HTML table summarizing brand and non-brand clicks, impressions, CTR, and position metrics for each time period, including week-over-week percent changes with color-coded indicators.

- **Nodes Involved:**  
  - Generate Brand Performance Table

- **Node Details:**

  - **Generate Brand Performance Table**  
    - Type: Code node  
    - Role:  
      - Iterates over combined data for each brand segment.  
      - Formats numbers and percentages, calculates week-over-week percentage changes (with positive changes in green, negative in red).  
      - Constructs an HTML table with labeled rows for "2 Weeks Ago" and "Last Week".  
      - Includes columns for raw metrics and % delta for clicks, impressions, CTR, and average position.  
    - Inputs: From Add Brand Name node.  
    - Outputs: JSON with a single field `htmlTable` containing the full HTML string.  
    - Edge cases: Missing data for either time period, division by zero in percent change, formatting errors.

---

#### 1.6 Email Dispatch

- **Overview:**  
  Sends the generated HTML performance table via Gmail as a scheduled weekly email.

- **Nodes Involved:**  
  - Weekly GSC Metric Email

- **Node Details:**

  - **Weekly GSC Metric Email**  
    - Type: Gmail node (n8n-nodes-base.gmail)  
    - Role: Sends an email with the HTML table as the email body.  
    - Configuration:  
      - Subject line: "Weekly GSC Metric Send"  
      - Message body uses expression to access `htmlTable` from previous node output.  
      - Uses OAuth2 credentials for Gmail account.  
    - Inputs: From Generate Brand Performance Table.  
    - Outputs: None (terminal node).  
    - Edge cases: Authentication failure, email quota limits, invalid HTML rendering in email clients.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                                   | Input Node(s)                 | Output Node(s)                | Sticky Note                                          |
|-------------------------------|-------------------------|--------------------------------------------------|------------------------------|------------------------------|------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger        | Triggers workflow weekly                          | —                            | Define Weeks                 |                                                      |
| Define Weeks                 | Code                    | Defines date ranges for comparison                | Schedule Trigger             | If4                         |                                                      |
| If4                          | If                      | Routes data by label ("2 Weeks Ago" or "Last Week") | Define Weeks                | Brand/NB Pull, Brand/NB Pull1 |                                                      |
| Brand/NB Pull                | HTTP Request            | Pulls GSC query data for "2 Weeks Ago"            | If4                         | Brand Filter1                | ## Connect to GSC account                             |
| Brand/NB Pull1               | HTTP Request            | Pulls GSC query data for "Last Week"               | If4                         | Brand Filter                 | ## Connect to GSC account                             |
| Total Metrics Pull           | HTTP Request            | Pulls total GSC metrics for "2 Weeks Ago"          | If4                         | Flatten9                    | ## Connect to GSC account                             |
| Total Metrics Pull1          | HTTP Request            | Pulls total GSC metrics for "Last Week"             | If4                         | Flatten8                    | ## Connect to GSC account                             |
| Brand Filter                 | Code                    | Filters and sums brand/non-brand clicks for "Last Week" | Brand/NB Pull1             | Merge Last Week              | ## Update with your own brand term(s)                 |
| Brand Filter1                | Code                    | Filters and sums brand/non-brand clicks for "2 Weeks Ago" | Brand/NB Pull              | Merge 2 Weeks Ago            | ## Update with your own brand term(s)                 |
| Flatten9                    | Code                    | Flattens total metrics for "2 Weeks Ago"            | Total Metrics Pull           | Merge 2 Weeks Ago            |                                                      |
| Flatten8                    | Code                    | Flattens total metrics for "Last Week"               | Total Metrics Pull1          | Merge Last Week              |                                                      |
| Merge 2 Weeks Ago           | Merge                   | Combines brand-filtered and total metrics for "2 Weeks Ago" | Brand Filter1, Flatten9     | Merge Time Periods           |                                                      |
| Merge Last Week             | Merge                   | Combines brand-filtered and total metrics for "Last Week" | Brand Filter, Flatten8      | Merge Time Periods           |                                                      |
| Merge Time Periods          | Merge                   | Combines both time periods' data for comparison     | Merge 2 Weeks Ago, Merge Last Week | Add Brand Name             |                                                      |
| Add Brand Name              | Code                    | Adds static brand name and bundles data             | Merge Time Periods           | Generate Brand Performance Table |                                                      |
| Generate Brand Performance Table | Code                | Generates an HTML table with metrics and % changes  | Add Brand Name               | Weekly GSC Metric Email      |                                                      |
| Weekly GSC Metric Email     | Gmail                   | Sends the HTML table via Gmail                       | Generate Brand Performance Table | —                        | ## Connect to Gmail account or update to something else |
| Sticky Note                 | Sticky Note             | Instruction to connect Gmail account                 | —                            | —                            | ## Connect to Gmail account or update to something else |
| Sticky Note1                | Sticky Note             | Instruction to connect GSC account                    | —                            | —                            | ## Connect to GSC account                             |
| Sticky Note2                | Sticky Note             | Instruction to update brand terms                      | —                            | —                            | ## Update with your own brand term(s)                 |
| Sticky Note3                | Sticky Note             | Workflow overview and setup instructions              | —                            | —                            | See section 5 for full content                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run weekly on Monday at 15:00 (3 PM).  
   - Connect to the next node.

2. **Add a Code node named "Define Weeks"**  
   - Paste JavaScript code that calculates two date ranges based on current date:  
     - "2 Weeks Ago": 15 to 9 days ago  
     - "Last Week": 8 to 2 days ago  
   - Output an array with these two date ranges labeled accordingly.  
   - Connect Schedule Trigger output to this node.

3. **Add an If node named "If4"**  
   - Condition: Check if the input's `label` equals "2 Weeks Ago" (case sensitive).  
   - Branch "true": For "2 Weeks Ago" data.  
   - Branch "false": For "Last Week" data.  
   - Connect Define Weeks output to this node.

4. **Add two HTTP Request nodes to pull GSC query data:**  
   - "Brand/NB Pull" for "2 Weeks Ago" branch  
   - "Brand/NB Pull1" for "Last Week" branch  
   - Both configured as POST requests to Google Search Console API endpoint (URL not shown here; user sets this according to GSC API docs).  
   - Body (JSON):  
     ```json
     {
       "startDate": "={{ $json.startDate }}",
       "endDate": "={{ $json.endDate }}",
       "dimensions": ["query"],
       "rowLimit": 5000
     }
     ```  
   - Authentication: Use Google OAuth2 credentials connected to GSC.  
   - Connect "If4" true branch to "Brand/NB Pull" and false branch to "Brand/NB Pull1".

5. **Add two HTTP Request nodes to pull total GSC metrics:**  
   - "Total Metrics Pull" for "2 Weeks Ago" branch  
   - "Total Metrics Pull1" for "Last Week" branch  
   - Both POST requests with body:  
     ```json
     {
       "startDate": "={{ $json.startDate }}",
       "endDate": "={{ $json.endDate }}",
       "rowLimit": 5000
     }
     ```  
   - No dimensions specified.  
   - Authentication: Same Google OAuth2 credentials.  
   - Connect "If4" true branch to "Total Metrics Pull" node and false branch to "Total Metrics Pull1" node.

6. **Add two Code nodes for brand filtering:**  
   - "Brand Filter1" connected after "Brand/NB Pull" (2 Weeks Ago)  
   - "Brand Filter" connected after "Brand/NB Pull1" (Last Week)  
   - JavaScript code to:  
     - Define `brandTerms` array with your brand keywords (e.g., `['brandname', 'brand']`).  
     - Sum clicks for queries containing brand terms in `brandClicks`.  
     - Sum clicks for other queries in `nonBrandClicks`.  
     - Output object with startDate, endDate, label, brandClicks, and nonBrandClicks.  
   - Replace placeholder `['BRAND TERMS']` with actual terms.

7. **Add two Code nodes to flatten total metric responses:**  
   - "Flatten9" after "Total Metrics Pull"  
   - "Flatten8" after "Total Metrics Pull1"  
   - JavaScript extracts the first row from `.rows` and merges with startDate and endDate.  
   - Connect respective nodes.

8. **Add two Merge nodes:**  
   - "Merge 2 Weeks Ago": combine "Brand Filter1" and "Flatten9" by position.  
   - "Merge Last Week": combine "Brand Filter" and "Flatten8" by position.  
   - Connect accordingly.

9. **Add a Merge node named "Merge Time Periods"**  
   - Combine "Merge 2 Weeks Ago" and "Merge Last Week" outputs.  
   - Connect both merges from step 8 to this node.

10. **Add a Code node named "Add Brand Name"**  
    - Add a static brand name string (replace `"BRAND NAME"` with your brand).  
    - Bundle all merged data under a `data` array field.  
    - Connect output of "Merge Time Periods" to this node.

11. **Add a Code node named "Generate Brand Performance Table"**  
    - JavaScript to:  
      - Iterate over input data.  
      - Format numbers and percentages.  
      - Calculate percent changes for clicks, impressions, CTR, position.  
      - Generate an HTML table with color-coded % changes (green for positive, red for negative).  
    - Connect output of "Add Brand Name" here.

12. **Add a Gmail node named "Weekly GSC Metric Email"**  
    - Use OAuth2 Gmail credentials.  
    - Subject: "Weekly GSC Metric Send"  
    - Message body: Use expression to insert the `htmlTable` field from previous node output.  
    - Connect "Generate Brand Performance Table" to this node.

13. **Add Sticky Notes as documentation:**  
    - Connect GSC OAuth2 credentials to HTTP Request nodes.  
    - Connect Gmail OAuth2 credentials to Gmail node.  
    - Update brand terms and brand name placeholders in code nodes.  
    - Use sticky notes to provide instructions and reminders.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow generates a weekly performance summary from Google Search Console, focused on brand-level SEO metrics and week-over-week trends. It provides a structured view by brand segment, with clean formatting and color-coded percent changes for quick insights. It is designed for SEO teams to track directional shifts and support deeper analysis. Setup requires ~5-10 minutes and connected GSC and Gmail accounts. | See Sticky Note3 in the workflow for detailed overview and setup instructions.                   |
| Update the `brandTerms` array in the Brand Filter code nodes to reflect your actual brand keywords or phrases for accurate segmentation between brand and non-brand queries.                                                                                                                                                                                                                                                                                                                                  | Sticky Note2                                                                                     |
| Gmail node requires OAuth2 credentials configured for your Gmail account or use an alternative email notification node as needed.                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note and Gmail node configuration                                                        |
| Google Search Console API credentials must be set up as OAuth2 in n8n for HTTP Request nodes to function correctly.                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note1                                                                                     |
| The workflow uses JavaScript code nodes extensively; ensure you understand and modify the code snippets for custom brand terms, brand name, and table formatting as needed.                                                                                                                                                                                                                                                                                                                                  | Code nodes "Brand Filter", "Add Brand Name", and "Generate Brand Performance Table"              |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a no-code automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.