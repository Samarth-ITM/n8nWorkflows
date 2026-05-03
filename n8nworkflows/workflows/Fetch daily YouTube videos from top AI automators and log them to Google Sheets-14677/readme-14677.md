Fetch daily YouTube videos from top AI automators and log them to Google Sheets

https://n8nworkflows.xyz/workflows/fetch-daily-youtube-videos-from-top-ai-automators-and-log-them-to-google-sheets-14677


# Fetch daily YouTube videos from top AI automators and log them to Google Sheets

# 1. Workflow Overview

This workflow is designed to run on a schedule, fetch recently published YouTube videos from a curated list of AI automation creators, compare those videos against an existing Google Sheet log, remove already logged entries, and append only new video URLs into the sheet.

Its main use case is competitive monitoring or content tracking: each day, the workflow checks selected YouTube channels, gathers their video outputs, and keeps a deduplicated record in Google Sheets.

## 1.1 Scheduled Execution and Initial Context

The workflow starts from a scheduled trigger and sets a date field intended to represent the reference publication window for YouTube queries.

## 1.2 Multi-Channel YouTube Data Collection

A group of YouTube nodes queries several predefined channels. Each node fetches videos for one creator/channel.

## 1.3 Aggregation of New Video Results

All YouTube result streams are merged into one combined list of fetched videos.

## 1.4 Existing Google Sheets Data Retrieval and Normalization

The workflow reads the existing rows from Google Sheets, deduplicates them, extracts YouTube video IDs from stored URLs, and reshapes them into a comparable format.

## 1.5 New vs Existing Video ID Comparison

Newly fetched video records and existing sheet records are converted into separate ID sets, merged together, and processed by custom JavaScript to keep only unseen video IDs.

## 1.6 Append New Rows to Google Sheets

The workflow appends the newly identified videos to the target Google Sheet.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Execution and Initial Context

**Overview:**  
This block starts the workflow automatically and creates a date field used as the initial context for downstream nodes. In the current JSON, the date is hardcoded rather than dynamically generated.

**Nodes Involved:**  
- Scheduled Daily Trigger  
- Set Scheduled Date Field

### Node: Scheduled Daily Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry point node that launches the workflow on a recurring schedule.
- **Configuration choices:**  
  The rule uses `interval` with a default/empty object, which indicates a basic recurring schedule but does not clearly define a production-ready cadence from the JSON alone.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none, this is a trigger node
  - Output: Set Scheduled Date Field
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Misconfigured schedule may run too often or not as expected
  - Timezone assumptions may differ from the intended daily execution window
- **Sub-workflow reference:**  
  None

### Node: Set Scheduled Date Field
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds a `date` field to the current item for downstream use.
- **Configuration choices:**  
  Assigns a string field named `date` with a hardcoded value: `2026-04-05T16:00:53.949+02:00`
- **Key expressions or variables used:**  
  - `date`
- **Input and output connections:**  
  - Input: Scheduled Daily Trigger
  - Outputs:
    - Fetch Nate Herk Videos
    - Fetch Nick Saraev Videos
    - Fetch Jack Roberts Videos
    - Fetch Cole Medin Videos
    - Fetch Nick Puru Videos
    - Fetch Ed Hill Videos
    - Fetch Jason Cooperson Videos
    - Fetch Manthan Patel Videos
    - Fetch Nick Daily Videos
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - The field is currently not actually used by the YouTube nodes in their filters
  - Hardcoded date means the workflow does not behave as a true rolling daily fetch unless manually updated
- **Sub-workflow reference:**  
  None

---

## 2.2 Multi-Channel YouTube Data Collection

**Overview:**  
This block fetches video records from selected YouTube channels. Each node queries one creator’s channel using the YouTube node with a publication range configuration.

**Nodes Involved:**  
- Fetch Nate Herk Videos  
- Fetch Nick Saraev Videos  
- Fetch Jack Roberts Videos  
- Fetch Cole Medin Videos  
- Fetch Nick Puru Videos  
- Fetch Ed Hill Videos  
- Fetch Jason Cooperson Videos  
- Fetch Manthan Patel Videos  
- Fetch Nick Daily Videos

### Common behavior across all YouTube nodes
- **Type and technical role:** `n8n-nodes-base.youTube`  
  Queries the YouTube API for video resources.
- **Configuration choices:**  
  - `resource`: video
  - `returnAll`: true
  - each node uses a specific `channelId`
  - `publishedAfter` is set to an empty expression (`=`)
  - `publishedBefore` is hardcoded to `2026-04-06T16:00:53.949+02:00`
- **Key expressions or variables used:**  
  - `publishedAfter` is malformed/empty in practice
  - no visible reference to the `date` field from the previous Set node
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Missing or invalid YouTube credentials
  - API quota limits
  - Empty `publishedAfter` may cause unexpected results or API validation issues depending on node/runtime behavior
  - Hardcoded `publishedBefore` makes the workflow non-dynamic
  - If a channel has many matching videos, `returnAll` can increase API load and runtime

### Node: Fetch Nate Herk Videos
- **Configuration choices:** channel ID `UC2ojq-nuP8ceeHqiroeKhBA`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 0)
- **Sub-workflow reference:** None

### Node: Fetch Nick Saraev Videos
- **Configuration choices:** channel ID `UCbo-KbSjJDG6JWQ_MTZ_rNA`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 1)
- **Sub-workflow reference:** None

### Node: Fetch Jack Roberts Videos
- **Configuration choices:** channel ID `UCxVxcTULO9cFU6SB9qVaisQ`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 2)
- **Sub-workflow reference:** None

### Node: Fetch Cole Medin Videos
- **Configuration choices:** channel ID `UCMwVTLZIRRUyyVrkjDpn4pA`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 3)
- **Sub-workflow reference:** None

### Node: Fetch Nick Puru Videos
- **Configuration choices:** channel ID `UC4FK5DEcMLB3CyJcbJfZEJA`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 4)
- **Sub-workflow reference:** None

### Node: Fetch Ed Hill Videos
- **Configuration choices:** channel ID is set as expression-style value `=UC5_2We-HeVdEeHcIyfmMHOg`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 5)
- **Edge cases or potential failure types:**  
  - Leading `=` suggests expression mode; if misinterpreted, this may fail or evaluate incorrectly
- **Sub-workflow reference:** None

### Node: Fetch Jason Cooperson Videos
- **Configuration choices:** channel ID `UCN3tZfcySn-VVeSRdGFIYOA`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 7)
- **Sub-workflow reference:** None

### Node: Fetch Manthan Patel Videos
- **Configuration choices:** channel ID `UCV_xSDw2dg7gfNWLfoEZAzg`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 8)
- **Sub-workflow reference:** None

### Node: Fetch Nick Daily Videos
- **Configuration choices:** channel ID `UCCM_PkxUkhTVxH3IE1SYzlg`
- **Input and output connections:**  
  - Input: Set Scheduled Date Field
  - Output: Combine Video Lists (input 9)
- **Sub-workflow reference:** None

**Important structural note:**  
`Combine Video Lists` is configured for `numberInputs: 10`, but only 9 YouTube nodes feed it. Input index 6 is unused. Depending on merge behavior and n8n version/runtime, this can lead to waiting behavior or configuration ambiguity and should be corrected.

---

## 2.3 Aggregation of New Video Results

**Overview:**  
This block merges the outputs from all YouTube fetch nodes into a single stream so the workflow can treat all newly fetched videos uniformly.

**Nodes Involved:**  
- Combine Video Lists

### Node: Combine Video Lists
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines multiple incoming result streams from the YouTube nodes.
- **Configuration choices:**  
  - `numberInputs: 10`
  - no additional merge mode parameters are shown
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs:
    - Fetch Nate Herk Videos
    - Fetch Nick Saraev Videos
    - Fetch Jack Roberts Videos
    - Fetch Cole Medin Videos
    - Fetch Nick Puru Videos
    - Fetch Ed Hill Videos
    - Fetch Jason Cooperson Videos
    - Fetch Manthan Patel Videos
    - Fetch Nick Daily Videos
  - Outputs:
    - Read Rows in Sheets
    - Set New Video IDs
- **Version-specific requirements:**  
  Type version `3.2`
- **Edge cases or potential failure types:**  
  - Mismatch between configured input count and actual connected inputs
  - Merge behavior may depend on execution mode and input availability
  - If one YouTube branch errors, the merge may not receive all expected inputs
- **Sub-workflow reference:**  
  None

---

## 2.4 Existing Google Sheets Data Retrieval and Normalization

**Overview:**  
This block reads the current video log from Google Sheets, removes duplicate sheet rows, extracts the video ID from each stored YouTube URL, and converts the result into a minimal comparable structure.

**Nodes Involved:**  
- Read Rows in Sheets  
- Deduplicate by All Fields  
- Execute Custom Code  
- Set Existing Video IDs

### Node: Read Rows in Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the target spreadsheet.
- **Configuration choices:**  
  - Authentication: service account
  - Document ID: `YOUR_SPREADSHEET_ID`
  - Sheet: `gid=0` / cached name `Sheet1`
  - A filter is defined on lookup column `Status`
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: Combine Video Lists
  - Output: Deduplicate by All Fields
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - Invalid Google service account credentials
  - Spreadsheet not shared with the service account email
  - Placeholder spreadsheet ID must be replaced
  - The filter only specifies a lookup column and no visible value; behavior may not match intended filtering
- **Sub-workflow reference:**  
  None

### Node: Deduplicate by All Fields
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Removes duplicate rows from the data retrieved from Sheets.
- **Configuration choices:**  
  Uses default options, implying deduplication across the full row payload.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: Read Rows in Sheets
  - Output: Execute Custom Code
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - If rows differ in non-essential columns, duplicates may remain
  - If formatting varies, logically identical URLs may not be treated as duplicates
- **Sub-workflow reference:**  
  None

### Node: Execute Custom Code
- **Type and technical role:** `n8n-nodes-base.code`  
  Extracts a `videoId` from the `YT Url` field in each existing sheet row.
- **Configuration choices:**  
  The code:
  - reads `item.json['YT Url']`
  - finds the last `=`
  - stores the substring after the last `=` as `videoId`
  - sets `videoId` to `null` when parsing fails
- **Key expressions or variables used:**  
  - `item.json['YT Url']`
  - `item.json.videoId`
- **Input and output connections:**  
  - Input: Deduplicate by All Fields
  - Output: Set Existing Video IDs
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Shortened URLs like `youtu.be/...` will not parse correctly
  - URLs with extra query parameters may yield incorrect results if `v=` is not the final `=`
  - Non-string or empty `YT Url` becomes `null`
- **Sub-workflow reference:**  
  None

### Node: Set Existing Video IDs
- **Type and technical role:** `n8n-nodes-base.set`  
  Reshapes each existing row into a field called `existingVideoIds`.
- **Configuration choices:**  
  Assigns a string field named `existingVideoIds` with expression value `=`
- **Key expressions or variables used:**  
  Intended output field:
  - `existingVideoIds`
- **Input and output connections:**  
  - Input: Execute Custom Code
  - Output: Merge Video ID Sets
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - As configured, the expression is empty and likely does not actually map `videoId`
  - The intended mapping was probably something like the extracted `videoId`, but this is missing
  - This can cause the comparison logic to fail or produce empty results
- **Sub-workflow reference:**  
  None

---

## 2.5 New vs Existing Video ID Comparison

**Overview:**  
This block converts fetched videos and existing rows into comparable sets, merges them, and uses code to determine which fetched video IDs are not yet present in the sheet.

**Nodes Involved:**  
- Set New Video IDs  
- Merge Video ID Sets  
- Process Merged Data

### Node: Set New Video IDs
- **Type and technical role:** `n8n-nodes-base.set`  
  Intended to map the newly fetched YouTube records into a field called `newVideoIds`.
- **Configuration choices:**  
  Assigns `newVideoIds` with expression value `=`
- **Key expressions or variables used:**  
  Intended output field:
  - `newVideoIds`
- **Input and output connections:**  
  - Input: Combine Video Lists
  - Output: Merge Video ID Sets
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - The expression is empty, so the node currently does not visibly extract a real video ID
  - This is a critical configuration gap
  - The intended source may be something like the YouTube node’s video ID field, but it is not configured in the JSON
- **Sub-workflow reference:**  
  None

### Node: Merge Video ID Sets
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the stream of new video IDs and existing video IDs before comparison.
- **Configuration choices:**  
  Default merge configuration.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs:
    - Set New Video IDs
    - Set Existing Video IDs
  - Output: Process Merged Data
- **Version-specific requirements:**  
  Type version `3.2`
- **Edge cases or potential failure types:**  
  - Merge semantics may differ depending on node defaults
  - If one branch outputs empty or malformed items, comparison logic becomes incomplete
- **Sub-workflow reference:**  
  None

### Node: Process Merged Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Compares the two sets and returns only IDs that are new.
- **Configuration choices:**  
  The code:
  - collects `existingVideoIds`
  - collects `newVideoIds`
  - creates a `Set` from existing IDs
  - returns items containing only unseen `newVideoIds`
- **Key expressions or variables used:**  
  - `item.json.newVideoIds`
  - `item.json.existingVideoIds`
- **Input and output connections:**  
  - Input: Merge Video ID Sets
  - Output: Append Rows to Sheets
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - If upstream Set nodes are empty, this node will compare empty values rather than real IDs
  - Null or blank IDs may be appended unless filtered out
  - No internal deduplication of new IDs before append beyond the set comparison with existing IDs
- **Sub-workflow reference:**  
  None

---

## 2.6 Append New Rows to Google Sheets

**Overview:**  
This final block writes the unseen YouTube videos into the Google Sheet. It formats a YouTube watch URL prefix into the `YT Url` column before appending.

**Nodes Involved:**  
- Append Rows to Sheets

### Node: Append Rows to Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends rows to the configured spreadsheet.
- **Configuration choices:**  
  - Operation: `append`
  - Authentication: service account
  - Document ID: `YOUR_SPREADSHEET_ID`
  - Sheet: `gid=0`
  - Column mapping is defined manually
  - `YT Url` is set to `=https://www.youtube.com/watch?v=`
  - `Status` is present in the schema but not assigned a value
- **Key expressions or variables used:**  
  - `YT Url`
  - `Status`
- **Input and output connections:**  
  - Input: Process Merged Data
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - As configured, `YT Url` contains only the base URL and does not append the actual `newVideoIds`
  - This is another critical configuration issue
  - Missing spreadsheet sharing permissions or bad credentials will fail the append
  - If the sheet expects required columns not mapped here, append may fail
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Daily Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | Set Scheduled Date Field | ## Schedule workflow trigger<br>Initiates the workflow at the scheduled time and sets initial data fields. |
| Set Scheduled Date Field | Set | Adds an initial date field for downstream processing | Scheduled Daily Trigger | Fetch Nate Herk Videos; Fetch Nick Saraev Videos; Fetch Jack Roberts Videos; Fetch Cole Medin Videos; Fetch Nick Puru Videos; Fetch Ed Hill Videos; Fetch Jason Cooperson Videos; Fetch Manthan Patel Videos; Fetch Nick Daily Videos | ## Schedule workflow trigger<br>Initiates the workflow at the scheduled time and sets initial data fields. |
| Fetch Nate Herk Videos | YouTube | Fetches videos from Nate Herk’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Nick Saraev Videos | YouTube | Fetches videos from Nick Saraev’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Jack Roberts Videos | YouTube | Fetches videos from Jack Roberts’ channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Cole Medin Videos | YouTube | Fetches videos from Cole Medin’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Nick Puru Videos | YouTube | Fetches videos from Nick Puru’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Ed Hill Videos | YouTube | Fetches videos from Ed Hill’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Jason Cooperson Videos | YouTube | Fetches videos from Jason Cooperson’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Manthan Patel Videos | YouTube | Fetches videos from Manthan Patel’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Fetch Nick Daily Videos | YouTube | Fetches videos from Nick Daily’s channel | Set Scheduled Date Field | Combine Video Lists | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Combine Video Lists | Merge | Combines outputs from all YouTube fetch nodes | Fetch Nate Herk Videos; Fetch Nick Saraev Videos; Fetch Jack Roberts Videos; Fetch Cole Medin Videos; Fetch Nick Puru Videos; Fetch Ed Hill Videos; Fetch Jason Cooperson Videos; Fetch Manthan Patel Videos; Fetch Nick Daily Videos | Read Rows in Sheets; Set New Video IDs | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Merge YouTube data<br>Combines data from multiple channels into a single stream. |
| Read Rows in Sheets | Google Sheets | Reads existing log rows from Google Sheets | Combine Video Lists | Deduplicate by All Fields | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Prepare new and existing data<br>Prepares existing and new data formats for merging and processing. |
| Deduplicate by All Fields | Remove Duplicates | Removes duplicate existing sheet rows | Read Rows in Sheets | Execute Custom Code | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Prepare new and existing data<br>Prepares existing and new data formats for merging and processing. |
| Execute Custom Code | Code | Extracts video IDs from existing YouTube URLs | Deduplicate by All Fields | Set Existing Video IDs | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Prepare new and existing data<br>Prepares existing and new data formats for merging and processing. |
| Set Existing Video IDs | Set | Maps existing sheet rows to a comparable ID field | Execute Custom Code | Merge Video ID Sets | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Prepare new and existing data<br>Prepares existing and new data formats for merging and processing. |
| Set New Video IDs | Set | Maps new YouTube results to a comparable ID field | Combine Video Lists | Merge Video ID Sets | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Prepare new and existing data<br>Prepares existing and new data formats for merging and processing. |
| Merge Video ID Sets | Merge | Combines new and existing ID streams | Set New Video IDs; Set Existing Video IDs | Process Merged Data | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Merge all data<br>Combines newly prepared and existing data for final processing. |
| Process Merged Data | Code | Filters out already logged video IDs | Merge Video ID Sets | Append Rows to Sheets | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Process finalized data<br>Applies final processing logic via custom JavaScript before storage. |
| Append Rows to Sheets | Google Sheets | Appends new video rows to the sheet | Process Merged Data |  | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed.<br>## Store processed data<br>Appends the processed data into Google Sheets, ensuring no duplicates. |
| Sticky Note | Sticky Note | Canvas documentation for the full workflow |  |  | YouTube AI Automation Spy<br>### How it works<br>1. Triggers a scheduled workflow execution.<br>2. Collects data from multiple YouTube channels based on set parameters.<br>3. Merges and processes the received data with existing records.<br>4. Applies custom logic to filter and refine the data.<br>5. Appends the processed data into a Google Sheet.<br>6. Ensures no duplicate data is appended through a deduplication step.<br>### Setup steps<br>- [ ] Configure Google Sheets API credentials for data retrieval and append operations.<br>- [ ] Set up YouTube API credentials to access channel information.<br>- [ ] Define the schedule for the trigger to automate the workflow execution.<br>### Customization<br>Modify the JavaScript code nodes to alter data processing logic as needed. |
| Sticky Note1 | Sticky Note | Canvas note describing the schedule block |  |  | ## Schedule workflow trigger<br>Initiates the workflow at the scheduled time and sets initial data fields. |
| Sticky Note2 | Sticky Note | Canvas note describing YouTube collection |  |  | ## YouTube data collection<br>Fetches data from various specified YouTube channels. |
| Sticky Note3 | Sticky Note | Canvas note describing merge of channel outputs |  |  | ## Merge YouTube data<br>Combines data from multiple channels into a single stream. |
| Sticky Note4 | Sticky Note | Canvas note describing preparation of new/existing data |  |  | ## Prepare new and existing data<br>Prepares existing and new data formats for merging and processing. |
| Sticky Note5 | Sticky Note | Canvas note describing combined comparison inputs |  |  | ## Merge all data<br>Combines newly prepared and existing data for final processing. |
| Sticky Note6 | Sticky Note | Canvas note describing final code processing |  |  | ## Process finalized data<br>Applies final processing logic via custom JavaScript before storage. |
| Sticky Note7 | Sticky Note | Canvas note describing sheet append stage |  |  | ## Store processed data<br>Appends the processed data into Google Sheets, ensuring no duplicates. |

---

# 4. Reproducing the Workflow from Scratch

Below is a rebuild sequence that matches the workflow’s intended structure, while also pointing out where the provided JSON contains incomplete settings that should be corrected during recreation.

## 4.1 Prepare external services
1. Create or choose a Google Sheet with at least these columns:
   - `YT Url`
   - `Status`
2. Copy the spreadsheet ID from the sheet URL.
3. Create Google Sheets credentials in n8n:
   - Preferably a **Google Service Account**
   - Share the spreadsheet with the service account email
4. Create YouTube credentials in n8n:
   - Configure access for the YouTube Data API
   - Ensure the Google Cloud project has YouTube Data API enabled

## 4.2 Create the trigger block
5. Add a **Schedule Trigger** node named **Scheduled Daily Trigger**.
6. Configure the schedule to run once per day at your preferred time.
   - The source JSON uses a minimal interval configuration, but for a daily workflow you should explicitly define the cadence.
7. Add a **Set** node named **Set Scheduled Date Field**.
8. Connect **Scheduled Daily Trigger → Set Scheduled Date Field**.
9. In **Set Scheduled Date Field**, add a field:
   - Name: `date`
   - Type: `string`
   - In the source workflow this is hardcoded to `2026-04-05T16:00:53.949+02:00`
10. Recommended correction: replace the hardcoded value with a dynamic expression representing the start of your desired daily window.

## 4.3 Create the YouTube fetch block
11. Add nine **YouTube** nodes with these names:
   - Fetch Nate Herk Videos
   - Fetch Nick Saraev Videos
   - Fetch Jack Roberts Videos
   - Fetch Cole Medin Videos
   - Fetch Nick Puru Videos
   - Fetch Ed Hill Videos
   - Fetch Jason Cooperson Videos
   - Fetch Manthan Patel Videos
   - Fetch Nick Daily Videos
12. For each node:
   - Resource: `video`
   - Return All: enabled
   - Set the proper YouTube credentials
13. Configure each channel ID:
   - Nate Herk: `UC2ojq-nuP8ceeHqiroeKhBA`
   - Nick Saraev: `UCbo-KbSjJDG6JWQ_MTZ_rNA`
   - Jack Roberts: `UCxVxcTULO9cFU6SB9qVaisQ`
   - Cole Medin: `UCMwVTLZIRRUyyVrkjDpn4pA`
   - Nick Puru: `UC4FK5DEcMLB3CyJcbJfZEJA`
   - Ed Hill: `UC5_2We-HeVdEeHcIyfmMHOg`
   - Jason Cooperson: `UCN3tZfcySn-VVeSRdGFIYOA`
   - Manthan Patel: `UCV_xSDw2dg7gfNWLfoEZAzg`
   - Nick Daily: `UCCM_PkxUkhTVxH3IE1SYzlg`
14. Configure publication filters.
   - In the source workflow:
     - `publishedAfter` is empty
     - `publishedBefore` is hardcoded to `2026-04-06T16:00:53.949+02:00`
15. Recommended correction:
   - Set `publishedAfter` dynamically from the date field or from a calculated “24 hours ago”
   - Set `publishedBefore` dynamically to “now”
16. Connect **Set Scheduled Date Field** to all nine YouTube nodes.

## 4.4 Merge all YouTube results
17. Add a **Merge** node named **Combine Video Lists**.
18. Set **Number of Inputs** to match the number of connected YouTube nodes.
   - The source JSON uses `10`, but only 9 sources are connected.
   - Recommended correction: set it to `9` unless you add a tenth source.
19. Connect each YouTube node to **Combine Video Lists** on its own input.

## 4.5 Read and prepare existing Google Sheets data
20. Add a **Google Sheets** node named **Read Rows in Sheets**.
21. Connect **Combine Video Lists → Read Rows in Sheets**.
22. Configure:
   - Authentication: service account
   - Operation: read/get rows
   - Spreadsheet ID: your sheet ID
   - Sheet: target tab, equivalent to `gid=0` in the source workflow
23. If you need filtering, set it explicitly.
   - The source workflow contains a lookup column `Status` but no clear filter value.
24. Add a **Remove Duplicates** node named **Deduplicate by All Fields**.
25. Connect **Read Rows in Sheets → Deduplicate by All Fields**.
26. Keep default settings if full-row deduplication is acceptable.

## 4.6 Extract existing video IDs from stored URLs
27. Add a **Code** node named **Execute Custom Code**.
28. Connect **Deduplicate by All Fields → Execute Custom Code**.
29. Paste this logic conceptually:
   - Read the `YT Url` field
   - Extract the video ID from the URL
   - Save it as `videoId`
30. The source code uses the last `=` in the URL to parse the ID.
31. Recommended correction:
   - Prefer parsing the `v` query parameter explicitly
   - Also support `youtu.be` URLs if needed

## 4.7 Normalize existing IDs
32. Add a **Set** node named **Set Existing Video IDs**.
33. Connect **Execute Custom Code → Set Existing Video IDs**.
34. Create a field:
   - Name: `existingVideoIds`
   - Type: `string`
35. Recommended correction:
   - Map it to the parsed `videoId`
   - Example intent: use the extracted `videoId` from the previous node
36. Note: in the source JSON the expression is empty, so reproduction should fix this or the workflow will not work.

## 4.8 Normalize new IDs from YouTube results
37. Add a **Set** node named **Set New Video IDs**.
38. Connect **Combine Video Lists → Set New Video IDs**.
39. Create a field:
   - Name: `newVideoIds`
   - Type: `string`
40. Recommended correction:
   - Map this to the YouTube video ID field returned by the YouTube node
   - The exact property name depends on the node output structure in your n8n version
41. Note: the source JSON leaves this expression empty, so you must define it manually.

## 4.9 Merge the new and existing ID streams
42. Add a **Merge** node named **Merge Video ID Sets**.
43. Connect:
   - **Set New Video IDs → Merge Video ID Sets**
   - **Set Existing Video IDs → Merge Video ID Sets**
44. Keep the default merge settings unless you need a specific merge mode.

## 4.10 Filter out already logged IDs
45. Add a **Code** node named **Process Merged Data**.
46. Connect **Merge Video ID Sets → Process Merged Data**.
47. Use logic equivalent to:
   - collect all `existingVideoIds`
   - collect all `newVideoIds`
   - create a set of existing IDs
   - output only the new IDs not already present
48. Recommended enhancement:
   - ignore blank/null IDs
   - deduplicate new IDs among themselves before append

## 4.11 Append new rows to Google Sheets
49. Add a **Google Sheets** node named **Append Rows to Sheets**.
50. Connect **Process Merged Data → Append Rows to Sheets**.
51. Configure:
   - Authentication: same Google service account
   - Operation: `append`
   - Spreadsheet ID: same target sheet
   - Sheet/tab: same target tab
52. Define the column mapping manually.
53. Map:
   - `YT Url` to the full watch URL constructed from the new video ID
   - `Status` optionally to a default value such as blank, `new`, or `unreviewed`
54. Important correction:
   - The source workflow maps `YT Url` only to `https://www.youtube.com/watch?v=` without concatenating the actual ID
   - You should append the actual video ID from `newVideoIds`
55. Save and test the workflow with one recent known video.

## 4.12 Add optional canvas notes
56. Add sticky notes if desired for documentation:
   - schedule workflow trigger
   - YouTube data collection
   - merge YouTube data
   - prepare new and existing data
   - merge all data
   - process finalized data
   - store processed data

## 4.13 Recommended fixes before production use
57. Replace all hardcoded timestamps with dynamic expressions.
58. Fix empty expressions in:
   - **Set Existing Video IDs**
   - **Set New Video IDs**
59. Fix the append mapping so `YT Url` includes the actual video ID.
60. Align the **Combine Video Lists** input count with the actual number of connected YouTube nodes.
61. Add error handling or retry strategy for:
   - YouTube quota/API failures
   - Google Sheets permission errors
62. Consider logging additional fields such as:
   - channel name
   - video title
   - publish date
   - fetch timestamp

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| YouTube AI Automation Spy | Overall workflow branding |
| How it works: triggers scheduled execution, collects YouTube data, merges and processes records, appends to Google Sheets, and avoids duplicates | Overall workflow description |
| Setup steps: configure Google Sheets API credentials, set up YouTube API credentials, define the trigger schedule | Deployment/setup context |
| Customization: modify the JavaScript code nodes to alter data processing logic as needed | Maintenance and extension guidance |

## Additional implementation observations
- The workflow has **one entry point**: **Scheduled Daily Trigger**.
- The workflow contains **no sub-workflow nodes** and invokes **no child workflows**.
- Several fields in the provided JSON are placeholders or incomplete, so the current exported workflow is best treated as a structural template rather than a production-ready configuration.
- The deduplication strategy is split across:
  - Google Sheets row deduplication for existing records
  - custom code comparison between existing and newly fetched video IDs
- The current implementation logs only URLs and does not persist richer metadata unless you extend the schema.