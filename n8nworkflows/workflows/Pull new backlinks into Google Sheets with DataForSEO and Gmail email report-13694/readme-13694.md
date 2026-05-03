Pull new backlinks into Google Sheets with DataForSEO and Gmail email report

https://n8nworkflows.xyz/workflows/pull-new-backlinks-into-google-sheets-with-dataforseo-and-gmail-email-report-13694


# Pull new backlinks into Google Sheets with DataForSEO and Gmail email report

This document provides a technical breakdown of the n8n workflow designed to automate the monitoring of new backlinks using the DataForSEO API, logging them into Google Sheets, and notifying a recipient via Gmail.

### 1. Workflow Overview
The workflow is designed to run on a daily schedule. It performs a paginated fetch of new backlinks for a specific target domain, filters the results to only include those discovered in the last 24 hours, formats this data into a structured report, and saves it to a newly created Google Sheet. Finally, it sends an email summary with a link to the spreadsheet.

**Logical Blocks:**
*   **1.1 Data Fetching Loop:** Triggers the process, initializes an array, and loops through DataForSEO API pages to collect all recent backlinks.
*   **1.2 Data Validation:** Checks if any new backlinks were found before proceeding to document generation.
*   **1.3 Spreadsheet Initialization:** Creates a new Google Sheet and sets up the header columns.
*   **1.4 Data Processing & Storage:** flattens the nested backlink data and appends it row-by-row into the spreadsheet.
*   **1.5 Notification:** Aggregates the results to trigger a single summary email via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Fetching Loop
*   **Overview:** This block manages the schedule and the recursive logic needed to handle pagination from the DataForSEO API.
*   **Nodes Involved:** `Schedule Trigger`, `Initialize "items" field`, `Set "items" field`, `Get new backlinks`, `Merge "items" with DFS response`, `Has more pages?`.
*   **Node Details:**
    *   **Schedule Trigger:** Fires every day at 09:00.
    *   **Initialize/Set "items" field:** Uses the `Set` node to maintain a persistent array across loop iterations.
    *   **Get new backlinks (DataForSEO):** Calls the `get-backlinks` operation. It uses an expression for `offset` (`$runIndex * 1000`) and a filter to only grab links where `first_seen` is greater than yesterday’s date.
    *   **Has more pages? (IF):** Checks if the current `$runIndex` is less than the total count divided by the limit (1000). It includes a safety cap of 5 pages (5000 links).

#### 1.2 Data Validation
*   **Overview:** Prevents the workflow from creating empty spreadsheets if no backlinks were found.
*   **Nodes Involved:** `Filter (has new backlinks)`.
*   **Node Details:**
    *   **Filter:** Checks if `total_count` from the DataForSEO response is greater than 0. If false, the workflow stops.

#### 1.3 Spreadsheet Initialization
*   **Overview:** Prepares the destination file.
*   **Nodes Involved:** `Create spreadsheet`, `Prepage columns for Google Sheets`, `Append columns`.
*   **Node Details:**
    *   **Create spreadsheet:** Creates a file named "New Backlinks to [Target] - [Date]".
    *   **Prepage columns:** A `Set` node in "Raw" mode that defines the header keys (Date, Target, Backlink, Spam Score, etc.) with empty values.
    *   **Append columns:** Inserts the headers into the newly created sheet.

#### 1.4 Data Processing & Storage
*   **Overview:** Prepares the raw API items for Google Sheets compatibility.
*   **Nodes Involved:** `Set final "items" field`, `Split out items`, `Prepare data for Google Sheets`, `Append row in sheet`.
*   **Node Details:**
    *   **Split out items:** Converts the array of backlink objects into individual n8n items.
    *   **Prepare data for Google Sheets:** Maps specific API fields (e.g., `url_to`, `backlink_spam_score`, `domain_from_rank`) to the column names created in section 1.3.

#### 1.5 Notification
*   **Overview:** Sends the final report link.
*   **Nodes Involved:** `Aggregate`, `Send a message`.
*   **Node Details:**
    *   **Aggregate:** Wait for all rows to be appended so that only one email is sent.
    *   **Send a message (Gmail):** Sends an HTML email containing the total backlink count and a hyperlink to the `spreadsheetUrl`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Workflow Entry | (None) | Initialize "items" field | |
| Initialize "items" field | Set | Variable Init | Schedule Trigger | Set "items" field | |
| Set "items" field | Set | Loop Buffer | Init / Has more pages? | Get new backlinks | |
| Get new backlinks | DataForSEO | API Data Fetch | Set "items" field | Merge "items" | Get new backlinks with DataForSEO |
| Merge "items"... | Set | Data Merging | Get new backlinks | Has more pages? | |
| Has more pages? | IF | Loop Logic | Merge "items" | Set "items" / Filter | |
| Filter (has new backlinks) | Filter | Condition Check | Has more pages? | Create spreadsheet | |
| Create spreadsheet | Google Sheets | File Creation | Filter | Prepage columns... | Create a Google Sheet with new backlinks... |
| Prepage columns... | Set | Header Mapping | Create spreadsheet | Append columns | Create a Google Sheet with new backlinks... |
| Append columns | Google Sheets | Header Setup | Prepage columns... | Set final "items" | Create a Google Sheet with new backlinks... |
| Set final "items" field | Set | Data Preparation | Append columns | Split out items | Create a Google Sheet with new backlinks... |
| Split out items | Split Out | Item Unnesting | Set final "items" | Prepare data... | Create a Google Sheet with new backlinks... |
| Prepare data... | Set | Field Mapping | Split out items | Append row in sheet | Create a Google Sheet with new backlinks... |
| Append row in sheet | Google Sheets | Data Logging | Prepare data... | Aggregate | Create a Google Sheet with new backlinks... |
| Aggregate | Aggregate | Flow Control | Append row in sheet | Send a message | Create a Google Sheet with new backlinks... |
| Send a message | Gmail | Notification | Aggregate | (None) | Create a Google Sheet with new backlinks... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Schedule Trigger** set to daily.
2.  **Pagination Setup:**
    *   Add a **Set** node (`Initialize "items"`) and create an empty array parameter `items`.
    *   Connect it to another **Set** node (`Set "items" field`) that passes `$json.items` forward.
3.  **API Integration:**
    *   Add a **DataForSEO** node. Select the Backlinks API and the "Get Backlinks" operation.
    *   **Target:** Set your domain (e.g., `example.com`).
    *   **Filters:** Use an expression: `["first_seen", ">", "{{ $now.minus(1, "days").format("yyyy-MM-dd") }}"]`.
    *   **Offset:** Set to `{{ $runIndex * 1000 }}`.
4.  **Looping Logic:**
    *   Add a **Set** node to merge the previous `items` with the new result: `{{ [...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`.
    *   Add an **If** node to check if `$runIndex` is less than `(total_count / 1000) - 1`. If true, loop back to `Set "items" field`.
5.  **Data Validation:** Connect the **If** (False branch) to a **Filter** node checking if `total_count > 0`.
6.  **Sheet Setup:**
    *   Add a **Google Sheets** node (Create) with a dynamic title using `$now`.
    *   Add a **Set** node to define your column headers as keys with empty string values.
    *   Add a **Google Sheets** node (Append) to write these headers.
7.  **Data Processing:**
    *   Add a **Split Out** node to break the `items` array into individual rows.
    *   Add a **Set** node to map DataForSEO fields (like `url_from`, `rank`) to your Sheet headers.
    *   Add a **Google Sheets** node (Append) to write the backlink data.
8.  **Notification:**
    *   Add an **Aggregate** node.
    *   Add a **Gmail** node. In the message body, use expressions to reference the `total_count` from the DataForSEO node and the `spreadsheetUrl` from the Create Spreadsheet node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Access DataForSEO API login and password | [DataForSEO API Access](https://app.dataforseo.com/api-access) |
| Target Domain Configuration | Update the 'Target' field in the 'Get new backlinks' node for your specific site. |
| Pagination Limit | The workflow is capped at 5 iterations (5000 backlinks) to prevent timeout in the 'Has more pages?' node. |