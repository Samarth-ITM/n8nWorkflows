Tracking Week-over-Week Changes in Google Search Console Data with a Focus on Significant Movements Across Keyword Segments

https://n8nworkflows.xyz/workflows/tracking-week-over-week-changes-in-google-search-console-data-with-a-focus-on-significant-movements-across-keyword-segments-6006


# Tracking Week-over-Week Changes in Google Search Console Data with a Focus on Significant Movements Across Keyword Segments

### 1. Workflow Overview

This workflow is designed to track week-over-week changes in Google Search Console (GSC) data, with a specific focus on identifying and highlighting significant movements across keyword segments. It targets SEO teams and analysts aiming to monitor organic search performance shifts efficiently by surfacing notable changes in clicks, CTR, impressions, and position at the query and page levels. The workflow runs weekly, compares two consecutive weeks of GSC data, segments keywords into categories such as brand, recipes, and nonbrand, and filters the most impactful changes for alerts and detailed reporting.

**Logical Blocks:**

- **1.1 Schedule and Define Weeks:** Trigger the workflow weekly and calculate date ranges for the prior and last weeks.  
- **1.2 Data Retrieval from GSC:** Fetch GSC data for both defined weeks, including pages and queries.  
- **1.3 Data Preparation and Flattening:** Label data with week identifiers and flatten nested GSC responses into simple row structures.  
- **1.4 Data Merging and Delta Calculation:** Merge prior and last week data by page-query pairs and calculate differences and percentage changes for key metrics.  
- **1.5 Keyword Segmentation:** Tag queries/pages into segments (brand, recipes, nonbrand) based on content and brand terms.  
- **1.6 Filtering and Alerting:** For each segment, filter top movers based on threshold criteria, send Slack alerts if applicable, and prepare HTML reports.  
- **1.7 Reporting:** Consolidate segment reports and send a comprehensive email with detailed tables of top movers.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule and Define Weeks

- **Overview:**  
  This block triggers the workflow every Monday at 15:00 and calculates the date ranges for the prior week (14 to 8 days ago) and last week (7 to 1 days ago).

- **Nodes Involved:**  
  - Schedule Trigger  
  - Define Weeks  
  - If

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Config: Runs every Monday at 15:00 (cron `0 15 * * 1`)  
    - Inputs: None (trigger node)  
    - Outputs: Triggers downstream nodes  
    - Edge Cases: Cron misconfiguration or n8n runtime issues could prevent trigger.

  - **Define Weeks**  
    - Type: Code  
    - Config: Uses JavaScript to compute and format dates for prior and last week periods relative to current date.  
    - Key variables: `priorStart`, `priorEnd`, `lastStart`, `lastEnd` formatted as ISO date strings.  
    - Inputs: Trigger output  
    - Outputs: Two items labeled `priorWeek` and `lastWeek` with respective date ranges.  
    - Edge Cases: Date calculation errors if system date is incorrect.

  - **If**  
    - Type: If (conditional node)  
    - Config: Routes flow based on the `label` property, checking if it equals `"priorWeek"` or `"lastWeek"`.  
    - Inputs: Items from Define Weeks  
    - Outputs: Two branches — one for `priorWeek`, one for `lastWeek`.  
    - Edge Cases: Mismatched or missing labels could cause misrouting.

---

#### 1.2 Data Retrieval from GSC

- **Overview:**  
  Pulls Google Search Console data for the defined weeks using the GSC API, requesting dimensions `page` and `query` with a row limit of 2500.

- **Nodes Involved:**  
  - priorWeek (HTTP Request)  
  - lastWeek (HTTP Request)  
  - label  
  - label1  
  - Flatten  
  - Flatten1

- **Node Details:**

  - **priorWeek / lastWeek**  
    - Type: HTTP Request  
    - Config: POST request to GSC Search Analytics endpoint with JSON body specifying date range, dimensions `[“page”, “query”]`, row limit 2500, and `dataState: "all"`.  
    - Authentication: Uses a predefined Google OAuth2 credential named "My Google Search Console Account".  
    - Inputs: From If node (filtered by label)  
    - Outputs: Raw GSC API response JSON.  
    - Edge Cases: Authentication failure, API rate limits, partial data returned, network timeouts.

  - **label / label1**  
    - Type: Code  
    - Config: Adds a `week` property to each item JSON (`"priorWeek"` or `"lastWeek"` depending on branch).  
    - Inputs: GSC API response  
    - Outputs: Items with week label included for downstream processing.  
    - Edge Cases: Empty or malformed input data.

  - **Flatten / Flatten1**  
    - Type: Code  
    - Config: Extracts and flattens the `rows` array from GSC response, mapping keys to `page` and `query`, and preserving metrics (`clicks`, `impressions`, `ctr`, `position`). Associates each row with the week label.  
    - Inputs: Labeled GSC data  
    - Outputs: One item per row with flattened JSON fields.  
    - Edge Cases: No `rows` field leads to empty output; incomplete rows may cause null fields.

---

#### 1.3 Data Merging and Delta Calculation

- **Overview:**  
  Merges prior and last week data sets by `page` and `query` keys, then computes absolute differences and percentage changes for clicks, CTR, impressions, and position.

- **Nodes Involved:**  
  - Merge  
  - Merge Weeks

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Config: Joins the flattened data from priorWeek and lastWeek branches into one stream with two inputs.  
    - Inputs: Flatten output from priorWeek and lastWeek  
    - Outputs: Combined items for comparison  
    - Edge Cases: Mismatched or missing data from either week would reduce matches.

  - **Merge Weeks**  
    - Type: Code  
    - Config:  
      - Separates items by week label.  
      - Indexes prior week data by concatenated key `page|||query`.  
      - For each last week item with matching key, calculates delta and percentage changes for:  
        - clicks  
        - ctr  
        - impressions  
        - position  
      - Formats percentage changes as strings with 1 decimal place and `%`.  
      - Outputs objects containing both current and prior week metrics plus delta values.  
    - Inputs: Merged data  
    - Outputs: Array of enriched comparison objects ready for segmentation.  
    - Edge Cases: Missing prior week data for a last week item is skipped; division by zero handled with null percentage change.

---

#### 1.4 Keyword Segmentation

- **Overview:**  
  Tags each keyword (query and page) into segments: brand, recipes, brand+recipes, or nonbrand. Also sorts data by click delta descending for prioritization.

- **Nodes Involved:**  
  - Tag Brand / Recipes / Nonbrand

- **Node Details:**

  - **Tag Brand / Recipes / Nonbrand**  
    - Type: Code  
    - Config:  
      - Converts query and page to lowercase.  
      - Checks if query contains any brand terms (placeholders "BRAND TERM 1", "BRAND TERM 2", "ETC.").  
      - Checks if page URL includes `/recipes`.  
      - Assigns `segment` string accordingly: `brand`, `recipes`, `brand+recipes`, or `nonbrand`.  
      - Sorts all items by absolute deltaClicks descending.  
    - Inputs: Merged Weeks output  
    - Outputs: Segmented and sorted items.  
    - Edge Cases: Brand terms must be customized to actual brand keywords; missing or malformed URLs/queries may misclassify segments.

---

#### 1.5 Filtering and Alerting per Segment

- **Overview:**  
  For each segment, filters the top movers by click delta and percentage change thresholds, sends Slack alerts on significant movements, and prepares HTML tables of top 25 positive and negative movers for email reporting.

- **Nodes Involved:**  
  - Switch  
  - Top Movers Filter (and 1, 2, 3)  
  - Top 25 Filter (and 1, 2, 3)  
  - Top WoW Movers Alert (and Alert1, Alert2, Alert3)

- **Node Details:**

  - **Switch**  
    - Type: Switch  
    - Config: Routes items based on `segment` property into one of four flows:  
      - Brand Flow  
      - Brand+Recipes Flow  
      - Recipes Flow  
      - Nonbrand Flow  
    - Inputs: Segmented items  
    - Outputs: Four separate paths matched by segment string  
    - Edge Cases: Unexpected segment values cause drop or misrouting.

  - **Top Movers Filter (0,1,2,3)**  
    - Type: Code  
    - Config:  
      - For each item, checks if absolute deltaClicks ≥ 200 AND absolute % change in clicks ≥ 30%.  
      - If so, creates a Slack alert text line with direction emoji and details.  
      - If no movers meet criteria, returns empty (no alert).  
      - Outputs message text for Slack.  
    - Inputs: Segment-specific data from Switch  
    - Outputs: Slack alert text or empty  
    - Edge Cases: No movers above threshold results in no alert sent.

  - **Top 25 Filter (0,1,2,3)**  
    - Type: Code  
    - Config:  
      - Splits items into "up" (positive deltaClicks) and "down" (negative deltaClicks).  
      - Sorts each list descending by absolute deltaClicks.  
      - Selects top 25 from each.  
      - Builds an HTML table with rows containing emoji, query, clickable page link, delta clicks, and percent change.  
      - Prepares subject and body JSON for email content.  
    - Inputs: Segment-specific data from Switch  
    - Outputs: Email content with subject and HTML body for reporting  
    - Edge Cases: Less than 25 movers in up/down handled gracefully by slice.

  - **Top WoW Movers Alert (0,1,2,3)**  
    - Type: Slack  
    - Config: Sends Slack message using webhook with text from corresponding Top Movers Filter.  
    - Credentials: Slack API Credential (named "Slack Account n8n-bot (YOUR.ACCOUNT)")  
    - Inputs: Alert text JSON  
    - Outputs: None (final action)  
    - Edge Cases: Slack webhook failures, message size limits.

---

#### 1.6 Reporting Consolidation and Email Delivery

- **Overview:**  
  Merges the four segment reports into a single email body with labeled sections and sends via Gmail.

- **Nodes Involved:**  
  - Merge4  
  - Code (titledTables)  
  - Top WoW Movers Email

- **Node Details:**

  - **Merge4**  
    - Type: Merge  
    - Config: Merges four inputs (one from each Top 25 Filter output) into one data stream.  
    - Inputs: Top 25 Filter outputs of all segments  
    - Outputs: Combined array of segment reports  
    - Edge Cases: Missing segment reports still allow merging.

  - **Code (titledTables)**  
    - Type: Code  
    - Config:  
      - Adds headings for each segment ("Brand KWs", "Brand+Recipes KWs", "Recipes KWs", "Nonbrand KWs").  
      - Concatenates HTML bodies from each segment with headings into a single email body.  
      - Sets email subject with current date.  
    - Inputs: Merged segment reports  
    - Outputs: Single email JSON with combined subject and body.  
    - Edge Cases: Missing or empty segment bodies result in empty sections.

  - **Top WoW Movers Email**  
    - Type: Gmail (Send Email)  
    - Config: Sends an email with subject and HTML body from previous node.  
    - Credential: Gmail OAuth2 Credential (named "Gmail Account 01 (YOUR.ACCOUNT)")  
    - Inputs: Email JSON with subject and body  
    - Outputs: None (final action)  
    - Edge Cases: SMTP or OAuth failures, email size limits.

---

#### 1.7 Sticky Notes (Setup & Documentation)

- **Overview:**  
  Provides inline documentation, reminders for credential setup, and configuration hints.

- **Nodes Involved:**  
  - Multiple Sticky Note nodes placed near relevant parts of the workflow.

- **Sticky Note Highlights:**

  - Connect to Google Search Console account for HTTP Request nodes.  
  - Connect to Slack account for Slack alert nodes.  
  - Connect to Gmail account or replace email node as needed.  
  - Instructions to customize brand terms and URL filters for segmentation.  
  - Threshold explanation: alerts trigger on absolute delta clicks ≥ 200 and percent change ≥ 30%.  
  - Detailed project description and usage notes.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                            | Input Node(s)           | Output Node(s)                                  | Sticky Note                                                                                     |
|----------------------------|----------------------|-------------------------------------------|-------------------------|------------------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger     | Trigger workflow weekly on Monday 15:00  | None                    | Define Weeks                                    |                                                                                                |
| Define Weeks              | Code                 | Calculate prior and last week date ranges| Schedule Trigger        | If                                             | Defines what the weeks are                                                                     |
| If                        | If                   | Branch by week label ("priorWeek" or "lastWeek") | Define Weeks            | priorWeek, lastWeek                             |                                                                                                |
| priorWeek                 | HTTP Request         | Fetch GSC data for prior week             | If (priorWeek branch)   | label                                           | Connect GSC account                                                                           |
| lastWeek                  | HTTP Request         | Fetch GSC data for last week              | If (lastWeek branch)    | label1                                          | Connect GSC account                                                                           |
| label                     | Code                 | Add "priorWeek" label to items            | priorWeek               | Flatten                                         |                                                                                                |
| label1                    | Code                 | Add "lastWeek" label to items              | lastWeek                | Flatten1                                        |                                                                                                |
| Flatten                   | Code                 | Flatten priorWeek GSC response rows       | label                   | Merge                                           |                                                                                                |
| Flatten1                  | Code                 | Flatten lastWeek GSC response rows        | label1                  | Merge                                           |                                                                                                |
| Merge                     | Merge                | Combine prior and last week flattened data| Flatten, Flatten1       | Merge Weeks                                     |                                                                                                |
| Merge Weeks               | Code                 | Calculate delta and % changes by page+query| Merge                   | Tag Brand / Recipes / Nonbrand                  |                                                                                                |
| Tag Brand / Recipes / Nonbrand | Code             | Segment keywords into brand/recipes/nonbrand | Merge Weeks             | Switch                                          | Create KW segments                                                                           |
| Switch                    | Switch               | Route items by segment                     | Tag Brand / Recipes / Nonbrand | Top Movers Filter / Top 25 Filter (4 branches) |                                                                                                |
| Top Movers Filter         | Code                 | Filter big movers (brand segment)          | Switch (Brand Flow)      | Top WoW Movers Alert                            |                                                                                                |
| Top Movers Filter1        | Code                 | Filter big movers (brand+recipes)           | Switch (Brand+Recipes)   | Top WoW Movers Alert1                           |                                                                                                |
| Top Movers Filter2        | Code                 | Filter big movers (recipes)                 | Switch (Recipes Flow)    | Top WoW Movers Alert2                           |                                                                                                |
| Top Movers Filter3        | Code                 | Filter big movers (nonbrand)                | Switch (Nonbrand Flow)   | Top WoW Movers Alert3                           |                                                                                                |
| Top 25 Filter             | Code                 | Prepare top 25 movers report (brand)       | Switch (Brand Flow)      | Merge4                                          |                                                                                                |
| Top 25 Filter1            | Code                 | Prepare top 25 movers report (brand+recipes)| Switch (Brand+Recipes)   | Merge4                                          |                                                                                                |
| Top 25 Filter2            | Code                 | Prepare top 25 movers report (recipes)     | Switch (Recipes Flow)    | Merge4                                          |                                                                                                |
| Top 25 Filter3            | Code                 | Prepare top 25 movers report (nonbrand)    | Switch (Nonbrand Flow)   | Merge4                                          |                                                                                                |
| Top WoW Movers Alert      | Slack                | Send Slack alert for big movers (brand)    | Top Movers Filter        | None                                            | Connect to Slack account                                                                    |
| Top WoW Movers Alert1     | Slack                | Send Slack alert (brand+recipes)            | Top Movers Filter1       | None                                            | Connect to Slack account                                                                    |
| Top WoW Movers Alert2     | Slack                | Send Slack alert (recipes)                   | Top Movers Filter2       | None                                            | Connect to Slack account                                                                    |
| Top WoW Movers Alert3     | Slack                | Send Slack alert (nonbrand)                  | Top Movers Filter3       | None                                            | Connect to Slack account                                                                    |
| Merge4                    | Merge                | Merge all segment reports for email         | Top 25 Filter, 1, 2, 3  | Code                                            |                                                                                                |
| Code                      | Code                 | Add section titles and concatenate reports  | Merge4                   | Top WoW Movers Email                            |                                                                                                |
| Top WoW Movers Email      | Gmail                | Send consolidated email report               | Code                     | None                                            | Connect to Gmail account or update to something else                                        |
| Sticky Note               | Sticky Note          | Documentation reminder to connect GSC       | None                     | None                                            | Connect to GSC account                                                                      |
| Sticky Note1              | Sticky Note          | Documentation reminder to connect Slack      | None                     | None                                            | Connect to Slack account                                                                    |
| Sticky Note2              | Sticky Note          | Documentation reminder to connect Gmail      | None                     | None                                            | Connect to Gmail account or update to something else                                        |
| Sticky Note3              | Sticky Note          | Documentation reminder about keyword segments| None                     | None                                            | Create KW segments                                                                         |
| Sticky Note4              | Sticky Note          | Threshold explanation for big movers alert  | None                     | None                                            | Current threshold is set to greater than 200 absolute delta clicks and 30% absolute change. |
| Sticky Note5              | Sticky Note          | Project overview and setup instructions      | None                     | None                                            | Detailed explanation of workflow purpose, how it works, and setup notes                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run every Monday at 15:00 (cron expression `0 15 * * 1`).  

2. **Add Code node "Define Weeks"**  
   - Purpose: Calculate `priorWeek` (14 to 8 days ago) and `lastWeek` (7 to 1 days ago) date ranges.  
   - Use JavaScript code to return two items with `label`, `startDate`, and `endDate` fields.  

3. **Add If node**  
   - Condition: Check if `$json.label` equals `"priorWeek"` or `"lastWeek"`.  
   - Two outputs correspond to the weeks.  

4. **Add two HTTP Request nodes: "priorWeek" and "lastWeek"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Google Search Console Search Analytics API endpoint (use appropriate GSC API URL)  
   - Body: JSON with `startDate`, `endDate`, `dimensions: ["page","query"]`, `rowLimit: 2500`, `dataState: "all"`  
   - Authentication: Google OAuth2 with valid GSC credentials.  

5. **Add two Code nodes "label" and "label1"**  
   - Purpose: Add `week` property (`"priorWeek"` or `"lastWeek"`) to each item.  
   - Code to map items and add the property accordingly.  

6. **Add two Code nodes "Flatten" and "Flatten1"**  
   - Purpose: Flatten the nested `rows` data from the GSC response into an array of objects with fields: `week`, `page`, `query`, `clicks`, `impressions`, `ctr`, `position`.  
   - Handle cases where `rows` is missing by returning an empty array.  

7. **Add Merge node "Merge"**  
   - Merge type: default (append)  
   - Inputs: Connect Flatten and Flatten1 outputs.  

8. **Add Code node "Merge Weeks"**  
   - Purpose:  
     - Separate items by week.  
     - Index priorWeek data by `page|||query`.  
     - For lastWeek items with matching priorWeek keys, compute deltas and percentage changes for clicks, CTR, impressions, position.  
   - Output enriched items with delta metrics and formatted percentages.  

9. **Add Code node "Tag Brand / Recipes / Nonbrand"**  
   - Purpose: Tag each item with segment labels based on query and page content.  
   - Customize brand term detection (replace placeholders with actual brand keywords).  
   - Sort all items by deltaClicks descending.  

10. **Add Switch node "Switch"**  
    - Route items based on `segment` property to four outputs:  
      - Brand  
      - Brand+Recipes  
      - Recipes  
      - Nonbrand  

11. **For each segment, add two Code nodes: "Top Movers Filter" and "Top 25 Filter"**  
    - "Top Movers Filter":  
      - Filter items where absolute deltaClicks ≥ 200 AND absolute % change ≥ 30%.  
      - Format Slack alert text with emoji and details.  
      - Return alert text or empty array if no movers.  
    - "Top 25 Filter":  
      - Split movers into up and down lists.  
      - Sort by absolute deltaClicks descending.  
      - Select top 25 from each.  
      - Generate HTML table with query, clickable page URL, delta clicks, and % change.  
      - Prepare JSON with subject and body for email.  

12. **Add Slack nodes for each "Top Movers Filter" output**  
    - Configure with Slack API credentials.  
    - Use incoming alert text as message content.  

13. **Add Merge node "Merge4"**  
    - Merge the four segment email report outputs from the "Top 25 Filter" nodes.  
    - Set to accept 4 inputs.  

14. **Add Code node "Code" after Merge4**  
    - Add headings for each segment report (Brand KWs, Brand+Recipes KWs, Recipes KWs, Nonbrand KWs).  
    - Concatenate HTML bodies with headings.  
    - Format final email subject with current date.  

15. **Add Gmail node "Top WoW Movers Email"**  
    - Configure with Gmail OAuth2 credentials.  
    - Send email with subject and HTML body from previous node.  

16. **Add Sticky Note nodes** as documentation reminders near:  
    - GSC HTTP Request nodes (remind to connect GSC account).  
    - Slack nodes (connect Slack account).  
    - Gmail node (connect Gmail account or alternative).  
    - Keyword segmentation code node (remind to customize brand terms and segments).  
    - Threshold explanation near filtering nodes.  
    - Overview and setup instructions near the start.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow tracks week-over-week changes in Google Search Console data and highlights significant keyword segment movements. It focuses on meaningful shifts rather than routine checks.                                         | General workflow purpose and design                                                             |
| Slack alerts only trigger if queries cross defined thresholds (±200 clicks and ±30% change), preventing alert fatigue.                                                                                                            | Alert thresholds                                                                                |
| Email report includes Top 25 positive and negative movers per segment with HTML tables and clickable page links for easy review.                                                                                                   | Email reporting details                                                                         |
| Customize brand terms and URL filters in the segmentation code node to adapt to your site's structure and brand vocabulary.                                                                                                        | Keyword segmentation customization                                                            |
| Setup requires Google Search Console OAuth2 credentials, Slack API webhook credentials, and Gmail OAuth2 credentials for email sending.                                                                                            | Credential setup                                                                                |
| Typical setup time: 15–25 minutes depending on segments and customizations.                                                                                                                                                         | Setup time estimate                                                                            |
| "Recipes" is an example content segment; adapt or add other categories as needed to fit your content taxonomy.                                                                                                                     | Content segmentation advice                                                                    |
| Slack nodes use webhook IDs that must be configured with your Slack workspace and bot user.                                                                                                                                         | Slack integration details                                                                      |
| For large GSC data sets, consider API limits and adjust `rowLimit` or implement pagination if necessary.                                                                                                                           | GSC API usage considerations                                                                  |
| This workflow is inactive by default; activate it after completing credential setup and testing.                                                                                                                                   | Workflow activation note                                                                       |
| Google Search Console API docs: https://developers.google.com/webmaster-tools/search-console-api-original/v3/searchanalytics/query                                                                                              | Official GSC API documentation                                                                 |
| n8n Slack node docs: https://docs.n8n.io/nodes/n8n-nodes-base.slack/                                                                                                                                                               | Slack node documentation                                                                       |
| n8n Gmail node docs: https://docs.n8n.io/nodes/n8n-nodes-base.gmail/                                                                                                                                                               | Gmail node documentation                                                                       |

---

**Disclaimer:** The text provided originates solely from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.