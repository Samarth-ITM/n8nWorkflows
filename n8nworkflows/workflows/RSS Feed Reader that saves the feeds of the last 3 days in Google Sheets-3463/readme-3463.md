RSS Feed Reader that saves the feeds of the last 3 days in Google Sheets

https://n8nworkflows.xyz/workflows/rss-feed-reader-that-saves-the-feeds-of-the-last-3-days-in-google-sheets-3463


# RSS Feed Reader that saves the feeds of the last 3 days in Google Sheets

### 1. Workflow Overview

This workflow automates the process of reading RSS feeds listed in a Google Sheets file, filtering entries from the last 3 days, saving these recent entries into another Google Sheets file, and cleaning up older entries from that second sheet. It is designed to run periodically (every 24 hours) and handles Google Sheets API rate limits by introducing wait times between writes and deletions.

**Target Use Cases:**  
- Aggregating recent news or blog updates from multiple RSS feeds  
- Maintaining a Google Sheets document with only the latest feed items (last 3 days)  
- Automating feed updates with controlled API usage to avoid Google Sheets blocking  

**Logical Blocks:**  
- **1.1 Schedule and Trigger**: Initiates the workflow periodically.  
- **1.2 Reading RSS Feed URLs**: Reads a list of RSS feed URLs from a Google Sheets file.  
- **1.3 Fetching and Filtering RSS Items**: Iterates over each RSS feed URL, fetches feed items, and filters them for the last 3 days.  
- **1.4 Formatting and Saving Recent Items**: Prepares the filtered items and saves them individually to a target Google Sheets file with pauses to avoid API rate limits.  
- **1.5 Cleaning Up Old Entries**: Reads existing entries from the target sheet, identifies entries older than 3 days, and deletes them with controlled timing.  
- **1.6 Rate Limiting Waits**: Implements wait nodes to pace the Google Sheets API calls to prevent blocking.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Trigger

- **Overview:**  
  Starts the workflow every 24 hours to perform the full RSS reading and updating process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger at a 24-hour interval (daily)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Read Links" node to start processing RSS feed URLs  
    - Edge Cases: Workflow will not run if the n8n instance is offline or credentials are invalid.

---

#### 2.2 Reading RSS Feed URLs

- **Overview:**  
  Reads RSS feed URLs from a Google Sheets file ("RSS-Links") to process each link individually.

- **Nodes Involved:**  
  - Read Links  
  - Loop Over Items1

- **Node Details:**  
  - **Read Links**  
    - Type: Google Sheets (Read operation)  
    - Configuration: Reads all rows from the first sheet (gid=0) of the Google Sheets document with ID `12p3M0Umh_Xlpm4Y04IpOFqE8YOJcCd97wPJNv80X8u4` ("RSS-Links")  
    - Credentials: Google Sheets OAuth2  
    - Output: Array of RSS feed links under the field `Links`  
    - Edge Cases: Authentication failures, empty or malformed sheet data, API rate limits  
  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Configuration: Processes each RSS feed URL one by one  
    - Inputs: Connected from "Read Links"  
    - Outputs: Two outputs, one to "Edit Fields" and one to "RSS" node  
    - Edge Cases: Empty input array, batch size defaults to 1 (assumed)

---

#### 2.3 Fetching and Filtering RSS Items

- **Overview:**  
  For each RSS feed URL, retrieves RSS items, formats fields, filters only items published within the last 3 days.

- **Nodes Involved:**  
  - RSS  
  - Edit Fields  
  - Code  
  - Loop Over Items  
  - Wait2  
  - Markdown

- **Node Details:**  
  - **RSS**  
    - Type: RSS Feed Read  
    - Configuration: Reads feed from URL provided by each item (`$json.Links`)  
    - Inputs: From "Loop Over Items1"  
    - Outputs: Raw RSS feed items with fields like `link`, `title`, `content`, `pubDate`, `categories`  
    - Edge Cases: Invalid URL, feed unreachable, malformed feed, empty feed  
  - **Edit Fields**  
    - Type: Set  
    - Configuration: Maps RSS item fields to normalized fields:  
      - `id` = link  
      - `title` = title  
      - `output` = content  
      - `pubDate` = pubDate  
      - `tags` = categories  
    - Inputs: From "Loop Over Items1" (likely from RSS output)  
    - Outputs: Standardized JSON structure for next filtering step  
    - Edge Cases: Missing fields in RSS items, null values  
  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration: Filters items to keep only those with `pubDate` within last 3 days  
    - Key Expression:  
      ```js
      const now = new Date();
      const setdays = 3;
      const cutoffDate = new Date();
      cutoffDate.setDate(now.getDate() - setdays);
      
      return $input.all().filter(item => {
        const pubDate = new Date(Date.parse(item.json.pubDate));
        return !isNaN(pubDate.getTime()) && pubDate >= cutoffDate;
      });
      ```
    - Inputs: From "Edit Fields"  
    - Outputs: Filtered list of recent RSS items  
    - Edge Cases: Invalid date formats, empty input  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Configuration: Processes each filtered RSS item individually  
    - Inputs: From "Code"  
    - Outputs: Two outputs, one to "Wait2" and one to "Markdown"  
    - Edge Cases: Empty input arrays  
  - **Wait2**  
    - Type: Wait  
    - Configuration: Waits for 1 minute before proceeding  
    - Inputs: From "Loop Over Items"  
    - Outputs: To "Read News" node  
    - Edge Cases: Workflow pausing too long or timing out  
  - **Markdown**  
    - Type: Markdown  
    - Configuration: Converts the `output` HTML content of RSS items into markdown format, stored back into `output` field  
    - Inputs: From "Loop Over Items"  
    - Outputs: To "Save News" node  
    - Edge Cases: Malformed HTML, conversion errors  

---

#### 2.4 Formatting and Saving Recent Items

- **Overview:**  
  Saves each filtered and formatted RSS item into the target Google Sheets file ("RSS-Feeds"), appending new entries or updating existing ones, with wait times to avoid Google API limits.

- **Nodes Involved:**  
  - Save News  
  - Wait

- **Node Details:**  
  - **Save News**  
    - Type: Google Sheets (AppendOrUpdate)  
    - Configuration:  
      - Document ID: `1iFwBIRDfUEZFACoL4bXfeT4Ot2i5vWfEew69fYRfz0A` ("RSS-Feeds")  
      - Sheet: gid=0  
      - Columns mapped:  
        - id (link)  
        - title  
        - output (markdown content)  
        - pubDate  
        - Category (tags)  
      - Matching column: id (to avoid duplicates)  
      - Append or update mode  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From "Markdown"  
    - Outputs: To "Wait"  
    - Edge Cases: API rate limits, write conflicts, invalid data  
  - **Wait**  
    - Type: Wait  
    - Configuration: Waits 2.5 seconds between each save operation to avoid Google Sheets API blocking  
    - Inputs: From "Save News"  
    - Outputs: Back to "Loop Over Items" for next item processing  
    - Edge Cases: Workflow delays or timeouts  

---

#### 2.5 Cleaning Up Old Entries

- **Overview:**  
  Reads all entries from the target "RSS-Feeds" Google Sheets file, identifies entries older than 3 days, and deletes them one by one with delays to avoid API blocking.

- **Nodes Involved:**  
  - Read News  
  - Code1  
  - Loop Over Items2  
  - Delete News  
  - Wait1

- **Node Details:**  
  - **Read News**  
    - Type: Google Sheets (Read)  
    - Configuration: Reads all entries from "RSS-Feeds" document ID `1iFwBIRDfUEZFACoL4bXfeT4Ot2i5vWfEew69fYRfz0A` sheet gid=0  
    - Credentials: Google Sheets OAuth2  
    - Outputs: Full rows including `pubDate` and `row_number`  
    - Edge Cases: Empty sheet, API errors  
  - **Code1**  
    - Type: Code (JavaScript)  
    - Configuration: Filters entries older than 3 days and sorts them by row number descending, returns rows for deletion  
    - Key Expression:  
      ```js
      const now = new Date();
      const setdays = 3;
      const cutoffDate = new Date();
      cutoffDate.setDate(now.getDate() - setdays);
      
      const oldRows = $input.all().filter(item => {
        const pubDate = new Date(item.json.pubDate);
        return pubDate < cutoffDate;
      });
      oldRows.sort((a, b) => b.json.row_number - a.json.row_number);
      return oldRows.map(item => ({ json: { rowNumber: item.json.row_number } }));
      ```
    - Inputs: From "Read News"  
    - Outputs: List of rows to delete  
    - Edge Cases: Invalid dates, empty input  
  - **Loop Over Items2**  
    - Type: SplitInBatches  
    - Configuration: Processes each old entry individually for deletion  
    - Inputs: From "Code1"  
    - Outputs: To "Delete News" on second output path, no first output used  
    - Edge Cases: Empty input  
  - **Delete News**  
    - Type: Google Sheets (Delete)  
    - Configuration: Deletes row from "RSS-Feeds" sheet based on rowNumber from input  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From "Loop Over Items2"  
    - Outputs: To "Wait1"  
    - Edge Cases: Row not found, API errors  
  - **Wait1**  
    - Type: Wait  
    - Configuration: Waits 25 seconds between each deletion to prevent Google Sheets API blocking  
    - Inputs: From "Delete News"  
    - Outputs: Back to "Loop Over Items2" for next deletion  
    - Edge Cases: Delay causing workflow timeout  

---

#### 2.6 Rate Limiting Waits

- **Overview:**  
  Implements wait periods after each write or delete to avoid Google Sheets API blocking due to rapid consecutive calls.

- **Nodes Involved:**  
  - Wait (2.5 seconds between save operations)  
  - Wait1 (25 seconds between delete operations)  
  - Wait2 (1 minute wait before reading news)

- **Node Details:**  
  - These nodes are configured with fixed wait durations to pace API calls.  
  - They are critical to preventing Google Sheets API rate limit errors or access blocking.  
  - Edge Cases: Excessive total workflow duration or timeouts if too many items are processed.

---

### 3. Summary Table

| Node Name       | Node Type             | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                           |
|-----------------|-----------------------|-------------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger       | Initiate workflow every 24 hours          | —                      | Read Links             | Timer starts the Update every 24 hours and Read the Links out of a Google Sheets File (RSS-Links)  |
| Read Links      | Google Sheets (Read)   | Reads RSS feed URLs from Google Sheets    | Schedule Trigger       | Loop Over Items1        | Each individual link is scanned and retrieved                                                      |
| Loop Over Items1| SplitInBatches         | Processes each RSS feed URL in batches    | Read Links              | Edit Fields, RSS        | Each individual link is scanned and retrieved                                                      |
| RSS             | RSS Feed Read          | Reads RSS items from each feed URL        | Loop Over Items1        | Loop Over Items         |                                                                                                     |
| Edit Fields     | Set                    | Normalizes RSS item fields                 | Loop Over Items1        | Code                   | Everything older than 3 days is removed                                                             |
| Code            | Code                   | Filters RSS items to last 3 days           | Edit Fields             | Loop Over Items         | Everything older than 3 days is removed                                                             |
| Loop Over Items | SplitInBatches         | Processes each filtered RSS item           | Code                    | Wait2, Markdown         | Each entry is saved individually with a waiting time in the Google Sheets file (RSS-Feeds), ...     |
| Markdown        | Markdown                | Converts HTML content to markdown          | Loop Over Items          | Save News               | Each entry is saved individually with a waiting time in the Google Sheets file (RSS-Feeds), ...     |
| Save News       | Google Sheets (AppendOrUpdate) | Saves or updates RSS items in Google Sheets | Markdown               | Wait                   | Each entry is saved individually with a waiting time in the Google Sheets file (RSS-Feeds), ...     |
| Wait            | Wait                   | Waits 2.5 seconds to avoid API blocking    | Save News               | Loop Over Items         | Each entry is saved individually with a waiting time in the Google Sheets file (RSS-Feeds), ...     |
| Wait2           | Wait                   | Waits 1 minute before reading news         | Loop Over Items          | Read News               | Reading the saved entries in the Google Sheets file (RSS-Feeds)                                     |
| Read News       | Google Sheets (Read)   | Reads existing RSS entries from Google Sheets | Wait2                  | Code1                   | Reading the saved entries in the Google Sheets file (RSS-Feeds)                                     |
| Code1           | Code                   | Filters old entries older than 3 days      | Read News               | Loop Over Items2         | Everything that is younger than 3 days will be removed, as we only want to delete the older entries!|
| Loop Over Items2| SplitInBatches         | Processes each old entry for deletion      | Code1                   | Delete News             | All entries older than 3 days are deleted here, again with a timer to prevent a Google API block!   |
| Delete News     | Google Sheets (Delete) | Deletes old entries from Google Sheets     | Loop Over Items2         | Wait1                   | All entries older than 3 days are deleted here, again with a timer to prevent a Google API block!   |
| Wait1           | Wait                   | Waits 25 seconds between delete operations | Delete News              | Loop Over Items2         | All entries older than 3 days are deleted here, again with a timer to prevent a Google API block!   |
| Sticky Note     | Sticky Note            | Infos on Schedule Trigger                   | —                       | —                       | Timer starts the Update every 24 hours and Read the Links out of a Google Sheets File (RSS-Links)  |
| Sticky Note1    | Sticky Note            | Infos on Loop Over Items1                   | —                       | —                       | Each individual link is scanned and retrieved                                                      |
| Sticky Note2    | Sticky Note            | Infos on Edit Fields and Code node filtering| —                       | —                       | Everything older than 3 days is removed                                                             |
| Sticky Note3    | Sticky Note            | Infos on Loop Over Items, Markdown, Save News | —                       | —                       | Each entry is saved individually with a waiting time in the Google Sheets file (RSS-Feeds), ...     |
| Sticky Note4    | Sticky Note            | Infos on Wait2 and Read News                | —                       | —                       | Reading the saved entries in the Google Sheets file (RSS-Feeds)                                     |
| Sticky Note5    | Sticky Note            | Infos on Code1 filtering old entries       | —                       | —                       | Everything that is younger than 3 days will be removed, as we only want to delete the older entries!|
| Sticky Note6    | Sticky Note            | Infos on Delete News and Wait1 node         | —                       | —                       | All entries older than 3 days are deleted here, again with a timer to prevent a Google API block!   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger every 24 hours (interval: daily).

2. **Add a Google Sheets node ("Read Links")**  
   - Operation: Read  
   - Google Sheets Document ID: `12p3M0Umh_Xlpm4Y04IpOFqE8YOJcCd97wPJNv80X8u4` (RSS-Links)  
   - Sheet Name: First sheet (gid=0)  
   - Credentials: Configure Google Sheets OAuth2 API credentials.

3. **Connect Schedule Trigger → Read Links**

4. **Add SplitInBatches node ("Loop Over Items1")**  
   - Splits incoming RSS feed URLs into individual items for processing.

5. **Connect Read Links → Loop Over Items1**

6. **Add RSS Feed Read node ("RSS")**  
   - URL: Use expression `{{$json["Links"]}}` to fetch feed URL from input JSON.  
   - No special options needed.

7. **Connect Loop Over Items1 (second output) → RSS**

8. **Add Set node ("Edit Fields")**  
   - Map RSS feed item fields to normalized fields:  
     - `id` = `{{$json["link"]}}`  
     - `title` = `{{$json["title"]}}`  
     - `output` = `{{$json["content"]}}`  
     - `pubDate` = `{{$json["pubDate"]}}`  
     - `tags` = `{{$json["categories"]}}`

9. **Connect Loop Over Items1 (first output) → Edit Fields**

10. **Add Code node ("Code")**  
    - JavaScript to filter items from the last 3 days:  
      ```js
      const now = new Date();
      const setdays = 3;
      const cutoffDate = new Date();
      cutoffDate.setDate(now.getDate() - setdays);
      
      return $input.all().filter(item => {
        const pubDate = new Date(Date.parse(item.json.pubDate));
        return !isNaN(pubDate.getTime()) && pubDate >= cutoffDate;
      });
      ```
11. **Connect Edit Fields → Code**

12. **Add SplitInBatches node ("Loop Over Items")**  
    - Processes each filtered RSS item one by one.

13. **Connect Code → Loop Over Items**

14. **Add Markdown node ("Markdown")**  
    - Converts `output` HTML content to markdown format, output key: `output`.  
    - Input expression for HTML content: `{{$json["output"]}}`

15. **Connect Loop Over Items (second output) → Markdown**

16. **Add Google Sheets node ("Save News")**  
    - Operation: AppendOrUpdate  
    - Document ID: `1iFwBIRDfUEZFACoL4bXfeT4Ot2i5vWfEew69fYRfz0A` (RSS-Feeds)  
    - Sheet: gid=0  
    - Mapping columns:  
      - id: `{{$json["id"]}}`  
      - title: `{{$json["title"]}}`  
      - output: `{{$json["output"]}}`  
      - pubDate: `{{$json["pubDate"]}}`  
      - Category: `{{$json["tags"]}}`  
    - Matching columns: `id` to avoid duplicates  
    - Credentials: Google Sheets OAuth2

17. **Connect Markdown → Save News**

18. **Add Wait node ("Wait")**  
    - Wait time: 2.5 seconds

19. **Connect Save News → Wait**

20. **Connect Wait → Loop Over Items (first output)**  
    - Loops back for next item processing.

21. **Connect Loop Over Items (first output) → Wait2**

22. **Add Wait node ("Wait2")**  
    - Wait time: 1 minute

23. **Connect Loop Over Items (first output) → Wait2**

24. **Add Google Sheets node ("Read News")**  
    - Operation: Read  
    - Document ID: `1iFwBIRDfUEZFACoL4bXfeT4Ot2i5vWfEew69fYRfz0A` (RSS-Feeds)  
    - Sheet: gid=0  
    - Credentials: Google Sheets OAuth2

25. **Connect Wait2 → Read News**

26. **Add Code node ("Code1")**  
    - Filters entries older than 3 days and sorts by row number descending:  
      ```js
      const now = new Date();
      const setdays = 3;
      const cutoffDate = new Date();
      cutoffDate.setDate(now.getDate() - setdays);
      
      const oldRows = $input.all().filter(item => {
        const pubDate = new Date(item.json.pubDate);
        return pubDate < cutoffDate;
      });
      oldRows.sort((a, b) => b.json.row_number - a.json.row_number);
      return oldRows.map(item => ({ json: { rowNumber: item.json.row_number } }));
      ```
27. **Connect Read News → Code1**

28. **Add SplitInBatches node ("Loop Over Items2")**  
    - Processes each old entry individually for deletion.

29. **Connect Code1 → Loop Over Items2**

30. **Add Google Sheets node ("Delete News")**  
    - Operation: Delete  
    - Document ID: `1iFwBIRDfUEZFACoL4bXfeT4Ot2i5vWfEew69fYRfz0A` (RSS-Feeds)  
    - Sheet: gid=0  
    - Uses input field `rowNumber` to delete the corresponding row.  
    - Credentials: Google Sheets OAuth2

31. **Connect Loop Over Items2 (second output) → Delete News**

32. **Add Wait node ("Wait1")**  
    - Wait time: 25 seconds

33. **Connect Delete News → Wait1**

34. **Connect Wait1 → Loop Over Items2**  
    - Loop back for the next deletion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The retrieval can take a while due to Google API block prevention depending on the number of feeds processed.                   | Workflow description                                                                                         |
| Detailed description is in the sticky notes attached to the workflow nodes.                                                     | Sticky Notes in the workflow                                                                                 |
| Timer starts the update every 24 hours and reads the links out of a Google Sheets file (RSS-Links).                             | Sticky Note on Schedule Trigger node                                                                         |
| Each individual link is scanned and retrieved.                                                                                 | Sticky Note on Loop Over Items1 node                                                                          |
| Everything older than 3 days is removed.                                                                                       | Sticky Note on Edit Fields and Code nodes                                                                     |
| Each entry is saved individually with a waiting time in the Google Sheets file (RSS-Feeds) to avoid Google API blocking.        | Sticky Note on Loop Over Items, Markdown, and Save News nodes                                                 |
| Reading the saved entries in the Google Sheets file (RSS-Feeds).                                                                | Sticky Note on Wait2 and Read News nodes                                                                      |
| Everything younger than 3 days will be removed, as only older entries are deleted.                                               | Sticky Note on Code1 node                                                                                      |
| All entries older than 3 days are deleted here with a timer to prevent Google API blocking.                                      | Sticky Note on Delete News and Wait1 nodes                                                                    |

---

This document provides a full structured analysis, enabling reproduction, modification, and troubleshooting of the RSS Feed Reader workflow that manages recent RSS feed entries in Google Sheets with rate-limit-aware pacing.