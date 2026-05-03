Daily Google Search Console SEO Pulse: Catch Top Movers Across Keyword Segments

https://n8nworkflows.xyz/workflows/daily-google-search-console-seo-pulse--catch-top-movers-across-keyword-segments-6004


# Daily Google Search Console SEO Pulse: Catch Top Movers Across Keyword Segments

### 1. Workflow Overview

This workflow, titled **"Daily Google Search Console SEO Pulse: Catch Top Movers Across Keyword Segments"**, is designed for SEO teams to monitor daily changes in Google Search Console (GSC) performance data. It detects significant shifts in keyword and page performance across defined segments such as brand, nonbrand, and content categories like recipes. The workflow runs on weekdays, compares the last two days' data, calculates deltas in clicks, impressions, CTR, and position, and flags notable movers with alerts distributed via Slack.

**Use Cases:**
- Daily SEO performance monitoring to quickly identify major gains or losses in search traffic.
- Segmentation-based analysis to isolate brand vs. nonbrand or specific content categories.
- Automated alerting for SEO teams via Slack to prompt timely investigation and action.

**Logical Blocks:**
- **1.1 Scheduled Trigger & Date Definition:** Initiates the workflow on weekdays and defines the two prior dates for comparison.
- **1.2 Data Retrieval from Google Search Console:** Queries GSC API for performance data for each of the two days.
- **1.3 Data Labeling & Flattening:** Adds day labels and transforms nested GSC response rows into flat JSON structures.
- **1.4 Data Merging and Delta Calculation:** Combines prior and last day data, calculates differences and percentage changes.
- **1.5 Keyword Segmentation & Sorting:** Tags keywords/pages into segments (brand, recipes, etc.) and sorts by click delta.
- **1.6 Segment-Based Routing:** Uses a switch node to route data by segment to different alert filters.
- **1.7 Top Movers Filtering & Alerting:** Applies thresholds to identify significant movers and sends Slack alerts accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Date Definition

- **Overview:**  
  Triggers the workflow on weekdays at 15:00 UTC and defines the prior two days’ dates for data extraction.

- **Nodes Involved:**  
  - Schedule Trigger1  
  - Define Days  
  - If

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Configuration: Runs on Monday to Friday at 15:00 UTC (cron: `0 15 * * 1-5`)  
    - Inputs: None  
    - Outputs: Starts the workflow by sending empty data to "Define Days"  
    - Edge cases: Cron misconfiguration could cause missed or extra runs  

  - **Define Days**  
    - Type: Code  
    - Role: Calculates and outputs two date objects labeled "priorDay" (two days ago) and "lastDay" (one day ago) in ISO format for API queries  
    - Key expressions: Uses JavaScript Date object, formats to `YYYY-MM-DD`  
    - Inputs: Trigger from Schedule Trigger1  
    - Outputs: Two items with JSON `{label, startDate, endDate}` for priorDay and lastDay  
    - Edge cases: Timezone differences might affect date calculations if server not UTC  

  - **If**  
    - Type: If  
    - Role: Branches the flow based on the label field (`priorDay` or `lastDay`) to separate API calls  
    - Configuration: Checks if `$json.label == "priorDay"`  
    - Inputs: Items from Define Days  
    - Outputs: Two outputs—first for priorDay, second for lastDay  
    - Edge cases: If label missing or mismatched, may cause data loss or API calls not triggered  

---

#### 1.2 Data Retrieval from Google Search Console

- **Overview:**  
  Fetches performance data from Google Search Console for the two specified days separately.

- **Nodes Involved:**  
  - priorDay (HTTP Request)  
  - lastDay (HTTP Request)

- **Node Details:**

  - **priorDay**  
    - Type: HTTP Request  
    - Role: POST request to GSC API with JSON body containing startDate, endDate, dimensions ([page, query]), rowLimit (2500), and dataState (all)  
    - Credential: Google OAuth2 API (predefined)  
    - Inputs: Output from If node’s first branch (priorDay)  
    - Outputs: Raw GSC data JSON with rows of search performance metrics  
    - Edge cases: API rate limits, auth token expiration, empty or malformed responses  

  - **lastDay**  
    - Type: HTTP Request  
    - Role: Same as priorDay but triggered from the second output branch of the If node (lastDay)  
    - Credential: Same Google OAuth2 API credential  
    - Edge cases: Same as priorDay  

---

#### 1.3 Data Labeling & Flattening

- **Overview:**  
  Adds day labels explicitly to the API response data and transforms nested row arrays into flat JSON objects for easier processing.

- **Nodes Involved:**  
  - label (Code)  
  - label1 (Code)  
  - Flatten (Code)  
  - Flatten1 (Code)  
  - Merge (Merge)

- **Node Details:**

  - **label**  
    - Type: Code  
    - Role: Adds a property `"day": "priorDay"` to each item from priorDay HTTP response  
    - Input: priorDay HTTP request output  
    - Output: Items with day label and original JSON  
    - Edge cases: Empty input items cause no output  

  - **label1**  
    - Type: Code  
    - Role: Adds `"day": "lastDay"` label for lastDay HTTP response items  
    - Input: lastDay HTTP request output  
    - Output: Items labeled with lastDay  

  - **Flatten**  
    - Type: Code  
    - Role: Takes the nested `rows` array from priorDay response and maps each row into a flat JSON object containing day, page, query, clicks, impressions, ctr, position  
    - Input: Output from label node (priorDay)  
    - Output: Flat list of rows for priorDay  

  - **Flatten1**  
    - Type: Code  
    - Role: Same as Flatten but for lastDay data from label1 node  

  - **Merge**  
    - Type: Merge  
    - Role: Combines flattened priorDay and lastDay items into a single stream with two inputs  
    - Input: Flatten and Flatten1 outputs  
    - Output: Combined list of all rows  

---

#### 1.4 Data Merging and Delta Calculation

- **Overview:**  
  Separates data by day, indexes prior day by page+query, matches with last day data, and calculates deltas and percentage changes for key metrics.

- **Nodes Involved:**  
  - Merge Days (Code)

- **Node Details:**

  - **Merge Days**  
    - Type: Code  
    - Role:  
      - Splits combined items into arrays by `day`  
      - Creates a Map keyed by `page|||query` for priorDay data  
      - Loops lastDay data, finds matching priorDay entries  
      - Calculates deltaClicks, deltaCTR, deltaImpressions, deltaPosition and their percent changes  
      - Formats CTR and delta fields as percentages  
      - Outputs merged enriched JSON objects for each matched page+query pair  
    - Inputs: Output from Merge node (combined flattened rows)  
    - Outputs: List of objects with calculated delta metrics  
    - Edge cases:  
      - Missing prior day data for some keys causes skips  
      - Division by zero handled by null or skipped percent changes  

---

#### 1.5 Keyword Segmentation & Sorting

- **Overview:**  
  Tags each row with segment labels (brand, recipes, brand+recipes, nonbrand) based on keyword and page patterns, then sorts results by deltaClicks descending.

- **Nodes Involved:**  
  - Tag KWs by Category (Code)

- **Node Details:**

  - **Tag KWs by Category**  
    - Type: Code  
    - Role:  
      - Converts query and page to lowercase  
      - Checks if query contains specific brand terms (placeholders in code for "BRAND TERM 1", etc.)  
      - Checks if page path contains "/recipes" (can be customized)  
      - Assigns segment label accordingly: "brand", "recipes", "brand+recipes", or "nonbrand"  
      - Sorts all items descending by deltaClicks  
    - Inputs: Output from Merge Days node  
    - Outputs: Segmented and sorted items  
    - Edge cases: Brand term detection relies on string includes; incomplete brand list may misclassify  

---

#### 1.6 Segment-Based Routing

- **Overview:**  
  Routes data items into four separate flows based on their segment label for targeted alerting.

- **Nodes Involved:**  
  - Segment Switch (Switch)

- **Node Details:**

  - **Segment Switch**  
    - Type: Switch  
    - Role: Uses exact string matching on `$json.segment` to route items to one of four outputs:  
      - Brand Flow (segment == "brand")  
      - Brand+Recipes Flow (segment == "brand+recipes")  
      - Recipes Flow (segment == "recipes")  
      - Nonbrand Flow (segment == "nonbrand")  
    - Input: Output from Tag KWs by Category  
    - Outputs: Four separate streams for next filtering nodes  
    - Edge cases: Items with unexpected or missing segment labels will be dropped  

---

#### 1.7 Top Movers Filtering & Alerting

- **Overview:**  
  Filters each segment’s data to find top movers based on absolute delta clicks ≥ 100 and percent click change ≥ 30%. Sends formatted alerts via Slack for each segment flow.

- **Nodes Involved:**  
  - Top Movers Filter  
  - Top Movers Filter1  
  - Top Movers Filter2  
  - Top Movers Filter3  
  - Top DoD Movers Alert  
  - Top DoD Movers Alert1  
  - Top DoD Movers Alert2  
  - Top DoD Movers Alert3

- **Node Details:**

  - **Top Movers Filter (and Filter1, Filter2, Filter3)**  
    - Type: Code  
    - Role:  
      - Iterates items, extracts deltaClicks and percentChangeClicks  
      - Flags items where absolute deltaClicks ≥ 100 AND absolute percent change ≥ 30%  
      - Prepares alert message lines with emoji arrows, query, delta clicks with sign, percentage change, and page URL  
      - Returns a single JSON item with a text property containing a multi-line alert message per segment  
    - Notes:  
      - Filter node names correspond to segments: Brand, Brand+Recipes, Recipes, Nonbrand  
      - Alert header text adjusted per segment (e.g., "Big DoD Movers Alert - BRAND")  
    - Inputs: From Segment Switch outputs  
    - Outputs: Single alert message or empty (if no movers)  

  - **Top DoD Movers Alert (and Alert1, Alert2, Alert3)**  
    - Type: Slack  
    - Role: Sends Slack message with alert text from corresponding Top Movers Filter node  
    - Credential: Slack API (bot user)  
    - Inputs: Output from respective Top Movers Filter node  
    - Outputs: None (terminal node)  
    - Edge cases: Slack API errors, webhook misconfiguration, empty message input (no alerts sent)  

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                                  | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                      |
|------------------------|----------------------|-------------------------------------------------|-------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1       | Schedule Trigger     | Initiates workflow on weekdays at 15:00 UTC     | None                    | Define Days                 |                                                                                                                 |
| Define Days            | Code                 | Defines priorDay and lastDay dates for queries  | Schedule Trigger1       | If                         | Defines what the days are                                                                                        |
| If                     | If                   | Splits flow by day label (priorDay vs lastDay)  | Define Days             | priorDay, lastDay           | Splits the day flows                                                                                            |
| priorDay               | HTTP Request         | Queries GSC API for prior day data               | If (priorDay branch)    | label                      | Connect to GSC account                                                                                           |
| lastDay                | HTTP Request         | Queries GSC API for last day data                | If (lastDay branch)     | label1                     | Connect to GSC account                                                                                           |
| label                  | Code                 | Adds day label "priorDay"                        | priorDay                | Flatten                    |                                                                                                                 |
| label1                 | Code                 | Adds day label "lastDay"                         | lastDay                 | Flatten1                   |                                                                                                                 |
| Flatten                | Code                 | Flattens priorDay GSC rows                       | label                   | Merge                      |                                                                                                                 |
| Flatten1               | Code                 | Flattens lastDay GSC rows                        | label1                  | Merge                      |                                                                                                                 |
| Merge                  | Merge                | Combines flattened priorDay and lastDay data    | Flatten, Flatten1       | Merge Days                 |                                                                                                                 |
| Merge Days             | Code                 | Merges prior and last day data, calculates deltas and % changes | Merge                   | Tag KWs by Category        |                                                                                                                 |
| Tag KWs by Category    | Code                 | Tags keywords/pages into segments and sorts     | Merge Days              | Segment Switch             | Create KW segment                                                                                                |
| Segment Switch         | Switch               | Routes items by segment to separate flows       | Tag KWs by Category     | Top Movers Filter nodes     | Divide segments into flows                                                                                       |
| Top Movers Filter      | Code                 | Filters Brand segment top movers & formats alert| Segment Switch (Brand)  | Top DoD Movers Alert       | Set alert threshold by DoD delta clicks & % change                                                              |
| Top Movers Filter1     | Code                 | Filters Brand+Recipes segment top movers         | Segment Switch (Brand+Recipes) | Top DoD Movers Alert1  | Set alert threshold by DoD delta clicks & % change                                                              |
| Top Movers Filter2     | Code                 | Filters Recipes segment top movers                | Segment Switch (Recipes)| Top DoD Movers Alert2      | Set alert threshold by DoD delta clicks & % change                                                              |
| Top Movers Filter3     | Code                 | Filters Nonbrand segment top movers               | Segment Switch (Nonbrand)| Top DoD Movers Alert3     | Set alert threshold by DoD delta clicks & % change                                                              |
| Top DoD Movers Alert   | Slack                | Sends Slack alert for Brand movers                | Top Movers Filter       | None                      | Connect to Slack account or update to email/etc.                                                                |
| Top DoD Movers Alert1  | Slack                | Sends Slack alert for Brand+Recipes movers        | Top Movers Filter1      | None                      | Connect to Slack account or update to email/etc.                                                                |
| Top DoD Movers Alert2  | Slack                | Sends Slack alert for Recipes movers               | Top Movers Filter2      | None                      | Connect to Slack account or update to email/etc.                                                                |
| Top DoD Movers Alert3  | Slack                | Sends Slack alert for Nonbrand movers              | Top Movers Filter3      | None                      | Connect to Slack account or update to email/etc.                                                                |
| Sticky Note            | Sticky Note          | Explains workflow purpose and setup instructions | None                    | None                      | See detailed note in "General Notes & Resources"                                                                |
| Sticky Note1           | Sticky Note          | Notes current alert thresholds                    | None                    | None                      | Current threshold is set to greater than 100 absolute delta clicks (i.e. positive or negative) and 30% absolute change. |
| Sticky Note2           | Sticky Note          | Reminder to connect GSC account                    | None                    | None                      | Connect to GSC account                                                                                           |
| Sticky Note3           | Sticky Note          | Reminder to connect Slack account or update alert channel | None                    | None                      | Connect to Slack account or update to email/etc.                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set Cron Expression to `0 15 * * 1-5` (15:00 UTC Monday to Friday)  

2. **Create Code Node "Define Days"**  
   - JavaScript code to generate two dates: priorDay (two days ago) and lastDay (one day ago) in `YYYY-MM-DD` format with labels  
   - Connect Schedule Trigger output to this node  

3. **Create If Node "If"**  
   - Condition: `$json.label` equals `"priorDay"`  
   - Connect Define Days output to If node input  

4. **Create HTTP Request Node "priorDay"**  
   - Method: POST  
   - URL: Google Search Console API endpoint for performance data (set accordingly)  
   - Body (JSON):  
     ```json
     {
       "startDate": "{{ $json.startDate }}",
       "endDate": "{{ $json.endDate }}",
       "dimensions": ["page", "query"],
       "rowLimit": 2500,
       "dataState": "all"
     }
     ```  
   - Authentication: Google OAuth2 API credential connected and authorized  
   - Connect If node's first output (priorDay) to this node  

5. **Create HTTP Request Node "lastDay"**  
   - Same configuration as priorDay node  
   - Connect If node's second output (lastDay) to this node  

6. **Create Code Node "label"**  
   - JavaScript: Add `"day": "priorDay"` to each item  
   - Connect priorDay HTTP node to this node  

7. **Create Code Node "label1"**  
   - JavaScript: Add `"day": "lastDay"` to each item  
   - Connect lastDay HTTP node to this node  

8. **Create Code Node "Flatten"**  
   - JavaScript: Map `rows` array to flat JSON objects containing day, page, query, clicks, impressions, ctr, position  
   - Connect "label" node to Flatten  

9. **Create Code Node "Flatten1"**  
   - Same as Flatten but for lastDay  
   - Connect "label1" node to Flatten1  

10. **Create Merge Node "Merge"**  
    - Mode: Merge Inputs (default)  
    - Connect Flatten (input 1) and Flatten1 (input 2) to Merge node  

11. **Create Code Node "Merge Days"**  
    - JavaScript:  
      - Separate items by day  
      - Index priorDay by page+query  
      - Match lastDay entries, calculate deltas and percent changes  
      - Format results with all metrics  
    - Connect Merge node to Merge Days  

12. **Create Code Node "Tag KWs by Category"**  
    - JavaScript:  
      - Lowercase queries and pages  
      - Define brand terms and recipe path filters  
      - Assign segment labels: brand, brand+recipes, recipes, nonbrand  
      - Sort by deltaClicks descending  
    - Connect Merge Days to this node  

13. **Create Switch Node "Segment Switch"**  
    - Add outputs for segments: "brand", "brand+recipes", "recipes", "nonbrand"  
    - Condition: `$json.segment == "segment_name"` for each output  
    - Connect Tag KWs by Category to Segment Switch  

14. **Create Code Nodes for Top Movers Filtering:**  
    - Four separate nodes, one for each segment (named Top Movers Filter, Top Movers Filter1, Top Movers Filter2, Top Movers Filter3)  
    - JavaScript:  
      - Iterate items, check if `|deltaClicks| ≥ 100` and `|percentChangeClicks| ≥ 30%`  
      - Format alert lines with arrows, query, delta, percent change, page URL  
      - Return a single item with a `text` property containing the alert message, or empty if none  
    - Connect each Segment Switch output to corresponding Top Movers Filter  

15. **Create Slack Nodes for Alerts:**  
    - Four Slack nodes corresponding to each Top Movers Filter node (named Top DoD Movers Alert, Alert1, Alert2, Alert3)  
    - Configure Slack credentials with OAuth or webhook for your workspace  
    - Message text: `={{$json.text}}`  
    - Connect each Top Movers Filter output to respective Slack node  

16. **Optionally Add Sticky Note Nodes:**  
    - Add explanatory sticky notes for workflow overview, thresholds, and connection reminders  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow helps SEO teams catch top movers in Google Search Console by comparing daily performance across keyword segments like brand, nonbrand, and content categories. It highlights biggest jumps/drops to spot wins or losses early. | Detailed functional description in Sticky Note node at workflow start |
| Current alert threshold is set to absolute delta clicks ≥ 100 and absolute percent change ≥ 30%. | Sticky Note near Top Movers Filter nodes |
| Connect your Google Search Console account with OAuth2 credentials. Use Slack API credentials for alert notifications, or customize for email/webhooks as needed. | Sticky Notes near respective nodes |
| The “recipes” segment is an example and can be customized to other content categories (blog, FAQ, product pages, etc.). | Sticky Note overview |
| For Google Search Console API details, refer to: https://developers.google.com/webmaster-tools/search-console-api-original/v3/searchanalytics/query | Official documentation |
| For Slack API webhook setup, refer to: https://api.slack.com/messaging/webhooks | Official documentation |

---

**Disclaimer:**  
The text above is exclusively derived from an automated n8n workflow. All data processed and referenced is legal and public. No illegal or protected content is included.