Monitor lost backlinks with DataForSEO, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/monitor-lost-backlinks-with-dataforseo--google-sheets-and-gmail-13695


# Monitor lost backlinks with DataForSEO, Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the monitoring of lost backlinks using the **DataForSEO** API. It is designed to run on a daily schedule, identifying backlinks that have recently disappeared for a specific target domain. Once the data is retrieved, the workflow generates a formatted **Google Sheets** report and sends a notification via **Gmail** containing the link to the results.

The workflow is divided into four logical stages:
1.  **Data Acquisition & Pagination:** Triggers the process and loops through the DataForSEO API results to handle large datasets.
2.  **Report Initialization:** Checks if any backlinks were lost and creates a new, timestamped Google Spreadsheet with the correct headers.
3.  **Data Formatting:** Maps the technical API response fields (e.g., Spam Score, Domain Rating) into human-readable columns.
4.  **Distribution:** Aggregates the results and sends an HTML-formatted email to the stakeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition & Pagination
This block handles the scheduled trigger and ensures all available backlink data is fetched, even if it spans multiple API pages.

*   **Nodes Involved:** `Schedule Trigger`, `Initialize "items" field`, `Set "items" field`, `Get lost backlinks`, `Merge "items" with DFS response`, `Has more pages?`.
*   **Node Details:**
    *   **Schedule Trigger:** Triggers every day at 09:00 AM.
    *   **Get lost backlinks (DataForSEO):** Fetches backlinks for a target domain (default: `dataforseo.com`) filtered by `last_seen` within the last day. It uses a limit of 1000 and an offset calculated by `{{ $runIndex * 1000 }}`.
    *   **Has more pages? (If):** A loop control node. It continues the loop if the current `$runIndex` is less than the total count divided by 1000, with a safety cap of 5 pages (5,000 links).
    *   **Edge Cases:** API timeout or invalid credentials will stop the loop. If no backlinks are found, the workflow continues but is filtered in the next block.

#### 2.2 Report Initialization
Once data is collected, this block prepares the Google Sheets environment.

*   **Nodes Involved:** `Filter (has new backlinks)`, `Create spreadsheet`, `Prepare columns for Google Sheets`, `Append columns`.
*   **Node Details:**
    *   **Filter (has new backlinks):** Checks if `total_count` is greater than 0. If 0, the workflow terminates to avoid sending empty reports.
    *   **Create spreadsheet:** Generates a new file named "Lost Backlinks to [Domain] - [Date]".
    *   **Prepare columns for Google Sheets (Set):** Defines the raw JSON structure for headers (Date, Target, Backlink, Spam Score, etc.).
    *   **Append columns:** Writes the header row into the newly created sheet.

#### 2.3 Data Transformation & Mapping
This block converts the raw API array into individual rows for the spreadsheet.

*   **Nodes Involved:** `Set final "items" field`, `Split out items`, `Prepare data for Google Sheets`, `Append row in sheet`.
*   **Node Details:**
    *   **Split out items:** Breaks the combined array of backlinks into separate n8n items for individual processing.
    *   **Prepare data for Google Sheets (Set):** Maps API variables (e.g., `url_to`, `backlink_spam_score`, `page_from_rank`) to the spreadsheet headers.
    *   **Append row in sheet:** Batch inserts the mapped data into the Google Sheet.

#### 2.4 Distribution
The final stage notifies the user that the report is ready.

*   **Nodes Involved:** `Aggregate`, `Send a message`.
*   **Node Details:**
    *   **Aggregate:** Consolidates all processed items back into a single execution context to trigger only one email.
    *   **Send a message (Gmail):** Sends an HTML email including the total count of lost backlinks and a direct hyperlink to the Google Spreadsheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Workflow Start | None | Initialize "items" field | This workflow runs on a scheduled basis... |
| Initialize "items" field | Set | Array Setup | Schedule Trigger | Set "items" field | |
| Set "items" field | Set | Loop Entry | Init "items" / Has more pages? | Get lost backlinks | |
| Get lost backlinks | DataForSEO | API Data Fetch | Set "items" field | Merge "items"... | Create a DataForSEO connection... |
| Merge "items" with DFS response | Set | Accumulator | Get lost backlinks | Has more pages? | |
| Has more pages? | If | Loop Logic | Merge "items"... | Set "items" / Filter | |
| Filter (has new backlinks) | Filter | Data Validation | Has more pages? | Create spreadsheet | Create a Google Sheets connection. Create a Gmail connection... |
| Create spreadsheet | Google Sheets | File Creation | Filter | Prepare columns... | Create a Google Sheets connection. Create a Gmail connection... |
| Prepare columns for Google Sheets | Set | Header Definition | Create spreadsheet | Append columns | Create a Google Sheets connection. Create a Gmail connection... |
| Append columns | Google Sheets | Header Insertion | Prepare columns... | Set final "items" | Create a Google Sheets connection. Create a Gmail connection... |
| Set final "items" field | Set | Final Collection | Append columns | Split out items | Create a Google Sheets connection. Create a Gmail connection... |
| Split out items | Split Out | Data Unrolling | Set final "items" | Prepare data... | Create a Google Sheets connection. Create a Gmail connection... |
| Prepare data for Google Sheets | Set | Field Mapping | Split out items | Append row | Create a Google Sheets connection. Create a Gmail connection... |
| Append row in sheet | Google Sheets | Data Insertion | Prepare data... | Aggregate | Create a Google Sheets connection. Create a Gmail connection... |
| Aggregate | Aggregate | Item Grouping | Append row | Send a message | Create a Google Sheets connection. Create a Gmail connection... |
| Send a message | Gmail | Email Notification | Aggregate | None | Create a Google Sheets connection. Create a Gmail connection... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger & Loop Setup:**
    *   Add a **Schedule Trigger** node set to your preferred interval (e.g., 9:00 AM).
    *   Add a **Set** node ("Initialize items") to create an empty array `items`.
    *   Add another **Set** node ("Set items") to pass the `items` array forward.
2.  **DataForSEO Integration:**
    *   Add a **DataForSEO** node. Select the "Get Backlinks" operation.
    *   Configure the `Target` (domain) and `Filters`. Use an expression for `Offset`: `{{ $runIndex * 1000 }}`.
    *   Add a **Set** node to merge the current API response with the previously collected `items` array.
    *   Add an **If** node to check if more pages exist. Connect the "true" output back to the "Set items" node to form a loop.
3.  **Google Sheets Preparation:**
    *   Add a **Filter** node after the loop to ensure the list isn't empty.
    *   Add a **Google Sheets** node ("Create Spreadsheet"). Use an expression for the title including `$now.format("yyyy-MM-dd")`.
    *   Add a **Set** node to define a JSON object containing your column headers (Date, Target, Backlink, etc.).
    *   Add a **Google Sheets** node ("Append") to insert these headers.
4.  **Data Processing:**
    *   Add a **Split Out** node to turn the final array of backlinks into individual items.
    *   Add a **Set** node to map the DataForSEO fields (e.g., `url_to`, `rank`) to the keys defined in your headers.
    *   Add a **Google Sheets** node ("Append") to write these rows into the document created in step 3.
5.  **Notification:**
    *   Add an **Aggregate** node to wait for all rows to be written.
    *   Add a **Gmail** node. Set the body to "HTML" and use expressions to pull the total count and the `spreadsheetUrl` from the creation node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| DataForSEO API Credentials Setup | [DataForSEO API Access](https://app.dataforseo.com/api-access) |
| Requirements | DataForSEO account, Google Sheets OAuth2, Gmail OAuth2 |
| Customization Idea | Change the Filter node to alert via Slack instead of Gmail. |