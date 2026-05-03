Research people with Perplexity AI and log results to Google Sheets

https://n8nworkflows.xyz/workflows/research-people-with-perplexity-ai-and-log-results-to-google-sheets-13937


# Research people with Perplexity AI and log results to Google Sheets

# 1. Workflow Overview

This workflow researches people listed in a Google Sheet using the Perplexity AI API, writes the results back into the same sheet, and sends Slack alerts when information is missing or when the API call fails.

Typical use case:
- A user enters a person’s name in a spreadsheet
- Sets the row status to `Pending`
- Runs the workflow manually
- The workflow asks Perplexity a configurable set of questions about that person
- It writes the answers into the proper columns
- It flags incomplete or failed runs

## 1.1 Manual Start and Sheet Intake

The workflow starts from a manual trigger, loads all rows from the main Google Sheet tab, and filters the data to keep only rows whose `Status` is `Pending`.

## 1.2 Row-by-Row Processing Control

Pending rows are processed one at a time through a loop. Each row is first marked as `Processing` in the sheet so the operator can see that work has started.

## 1.3 Dynamic Question Configuration

For each row, the workflow reads a separate Config tab from the same spreadsheet. That Config tab defines:
- the internal field key
- the prompt/question template
- the target sheet column header or mapping label

The workflow then builds a custom Perplexity prompt by replacing `[NAME]` with the person’s actual name.

## 1.4 AI Research Request

A structured HTTP request is sent to Perplexity’s chat completions endpoint. The prompt instructs Perplexity to return only valid JSON with exactly the expected keys.

## 1.5 Response Parsing and Sheet Mapping

The JSON response is parsed, normalized, and converted into a Google Sheets update payload. Missing values are replaced with `Not found`. The output row is marked `Done` if the request succeeds.

## 1.6 Missing-Field Notification

If some fields were not found, the workflow still writes the available results to the sheet, then sends a Slack warning containing the person name, row number, and missing fields.

## 1.7 API Error Handling

If the Perplexity API call fails entirely, the workflow writes `Error` into the row status, stores the error message in the `Error Log` column, and sends a separate Slack alert.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Manual Start and Pending Row Selection

### Overview
This block starts the workflow manually, reads all rows from the main Google Sheet, and filters the dataset so only rows with `Status = Pending` continue. It is the main intake gate for the workflow.

### Nodes Involved
- When clicking 'Execute workflow'
- Get All Rows
- Filter — Pending Only

### Node Details

#### 1) When clicking 'Execute workflow'
- **Type and technical role:** `manualTrigger`; starts the workflow when launched manually from the n8n editor.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Get All Rows`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None beyond standard manual execution context.
- **Sub-workflow reference:** None.

#### 2) Get All Rows
- **Type and technical role:** `googleSheets`; reads rows from the main spreadsheet tab.
- **Configuration choices:**
  - Uses Google Sheets OAuth2 credentials
  - Reads from `YOUR_GOOGLE_SHEET_ID`
  - Reads from tab `YOUR_SHEET_TAB_ID`
  - `alwaysOutputData` is enabled, which helps preserve execution continuity even if the sheet is empty
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**  
  - Input: `When clicking 'Execute workflow'`  
  - Output: `Filter — Pending Only`
- **Version-specific requirements:** Google Sheets node type version 4.5.
- **Edge cases or potential failure types:**
  - Invalid Google Sheets credentials
  - Sheet or tab ID mismatch
  - Empty sheet
  - Missing expected headers such as `Status`
- **Sub-workflow reference:** None.

#### 3) Filter — Pending Only
- **Type and technical role:** `filter`; allows only rows where `Status` equals `Pending`.
- **Configuration choices:**
  - One condition only
  - String equality comparison
  - Left value: `{{ $json['Status'] }}`
  - Right value: `Pending`
- **Key expressions or variables used:**
  - `$json['Status']`
- **Input and output connections:**  
  - Input: `Get All Rows`  
  - Output: `Loop Over Rows`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**
  - Header typo in Google Sheet (`Status` must match)
  - Status values with trailing spaces or different casing will not match
  - Blank rows are ignored
- **Sub-workflow reference:** None.

---

## 2.2 Block: Row Loop and Processing Status Update

### Overview
This block ensures rows are processed one by one. Before research starts, it updates the current row’s status to `Processing` in the sheet.

### Nodes Involved
- Loop Over Rows
- Set Status — Processing

### Node Details

#### 4) Loop Over Rows
- **Type and technical role:** `splitInBatches`; iterates through filtered rows one at a time.
- **Configuration choices:** Default batch behavior, used as a loop controller.
- **Key expressions or variables used:** Later nodes reference its current item using `$('Loop Over Rows').item.json`.
- **Input and output connections:**  
  - Input: `Filter — Pending Only`
  - Iteration output: `Set Status — Processing`
  - Loop continuation input comes back from `Any Fields Missing?`
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**
  - If no rows are pending, nothing proceeds
  - Downstream references to `$('Loop Over Rows').item...` depend on loop context being valid
- **Sub-workflow reference:** None.

#### 5) Set Status — Processing
- **Type and technical role:** `googleSheets`; updates the current row to show work has started.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Writes:
    - `Status = Processing`
    - `row_number = {{ $json['row_number'] }}`
- **Key expressions or variables used:**
  - `$json['row_number']`
- **Input and output connections:**  
  - Input: `Loop Over Rows`
  - Output: `Get Config Tab`
- **Version-specific requirements:** Google Sheets type version 4.5.
- **Edge cases or potential failure types:**
  - `row_number` missing from the input data
  - Spreadsheet permissions issue
  - If `row_number` is not preserved by the Google Sheets read operation, the update cannot target the row
- **Sub-workflow reference:** None.

---

## 2.3 Block: Config Retrieval and Prompt Construction

### Overview
This block reads the configuration tab and builds a dynamic Perplexity prompt for the current person. It converts spreadsheet-defined question templates into a strict JSON-answer request.

### Nodes Involved
- Get Config Tab
- Build Dynamic Prompt

### Node Details

#### 6) Get Config Tab
- **Type and technical role:** `googleSheets`; reads the configuration tab that defines research questions.
- **Configuration choices:**
  - Reads from the same document ID as the main sheet
  - Uses tab `YOUR_CONFIG_TAB_ID`
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**  
  - Input: `Set Status — Processing`
  - Output: `Build Dynamic Prompt`
- **Version-specific requirements:** Google Sheets type version 4.5.
- **Edge cases or potential failure types:**
  - Wrong config tab ID
  - Missing columns: `Field Key`, `Question Template`, `Column`
  - Empty config tab results in an empty prompt schema
- **Sub-workflow reference:** None.

#### 7) Build Dynamic Prompt
- **Type and technical role:** `code`; constructs the research prompt and metadata for downstream parsing/mapping.
- **Configuration choices:**
  - Reads the current person name from `$('Loop Over Rows').item.json['Name']`
  - Reads all config rows from `$input.all()`
  - Replaces `[NAME]` placeholders in each question template
  - Builds:
    - `field_map`
    - `field_keys`
    - `prompt`
    - `name`
    - `row_number`
- **Key expressions or variables used:**
  - `$('Loop Over Rows').item.json['Name']`
  - `$('Loop Over Rows').item.json['row_number']`
  - Config columns:
    - `Field Key`
    - `Column`
    - `Question Template`
- **Input and output connections:**  
  - Input: `Get Config Tab`
  - Output: `Perplexity — Research Call`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Missing `Name` value in current row
  - Empty config rows
  - Duplicate `Field Key` values will overwrite earlier entries in the `fieldMap`
  - If `Column` values do not match the target sheet’s incoming mapping strategy, results may not land in expected columns
- **Sub-workflow reference:** None.

**Important implementation note:**  
The code uses the `Column` value from the config tab as the output key later in the workflow. In practice, this value must align with what the Google Sheets update node expects during automapping. Although the sticky note example describes letters like `C`, the update node schema is based on column headers such as `Current Company`, `Location`, etc. To avoid mismatches, the config tab should use actual sheet header names unless you intentionally handle column-letter mapping elsewhere.

---

## 2.4 Block: Perplexity API Call and Error Branch

### Overview
This block sends the generated prompt to Perplexity and branches based on the result. A successful API response continues to parsing, while an HTTP failure goes to the dedicated error-writing path.

### Nodes Involved
- Perplexity — Research Call
- Write Error to Sheet
- Slack — API Error Alert

### Node Details

#### 8) Perplexity — Research Call
- **Type and technical role:** `httpRequest`; calls the Perplexity chat completions API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.perplexity.ai/chat/completions`
  - Timeout: 30000 ms
  - Body sent as JSON
  - Authentication: generic credential type using HTTP Header Auth
  - Model: `sonar`
  - Sends:
    - system message requiring JSON-only output
    - user message containing the generated prompt
  - `onError` is set to `continueErrorOutput`, enabling a second output path on failure
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.prompt) }}`
- **Input and output connections:**  
  - Input: `Build Dynamic Prompt`
  - Success output: `Parse Perplexity Response`
  - Error output: `Write Error to Sheet`
- **Version-specific requirements:** HTTP Request node version 4.2.
- **Edge cases or potential failure types:**
  - Invalid Perplexity API key
  - Header auth misconfigured
  - Rate limiting
  - Timeout
  - API schema changes
  - Model availability changes
- **Sub-workflow reference:** None.

#### 9) Write Error to Sheet
- **Type and technical role:** `googleSheets`; writes error status and error details when the API call itself fails.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Writes:
    - `Status = Error`
    - `Error Log = {{ $json.error?.message ?? 'Perplexity API call failed — check credentials or rate limits' }}`
    - `row_number = {{ $('Loop Over Rows').item.json['row_number'] }}`
  - Uses document ID directly
  - Sheet name is set by name as `Sheet1`
- **Key expressions or variables used:**
  - `$json.error?.message`
  - `$('Loop Over Rows').item.json['row_number']`
- **Input and output connections:**  
  - Input: error output of `Perplexity — Research Call`
  - Output: `Slack — API Error Alert`
- **Version-specific requirements:** Google Sheets type version 4.5.
- **Edge cases or potential failure types:**
  - Inconsistency risk: this node uses `Sheet1` by name, whereas other Google Sheets nodes use `YOUR_SHEET_TAB_ID`
  - If the actual tab is not named `Sheet1`, this node will fail
  - Missing `row_number`
- **Sub-workflow reference:** None.

#### 10) Slack — API Error Alert
- **Type and technical role:** `slack`; sends a message to Slack when the Perplexity API request fails.
- **Configuration choices:**
  - Sends to a selected channel by `YOUR_SLACK_CHANNEL_ID`
  - Message includes:
    - person name
    - row number
    - error message
    - current timestamp
- **Key expressions or variables used:**
  - `$('Loop Over Rows').item.json['Name']`
  - `$('Loop Over Rows').item.json['row_number']`
  - `$json.error?.message`
  - `$now.toISO()`
- **Input and output connections:**  
  - Input: `Write Error to Sheet`
  - Output: none
- **Version-specific requirements:** Slack node version 2.3.
- **Edge cases or potential failure types:**
  - Slack credentials invalid
  - Channel ID invalid or inaccessible
  - Error payload missing expected fields
- **Sub-workflow reference:** None.

---

## 2.5 Block: Response Parsing and Dynamic Sheet Update

### Overview
This block parses the Perplexity response, extracts values for the configured fields, maps them into a Google Sheets update payload, and writes the final results back into the correct row.

### Nodes Involved
- Parse Perplexity Response
- Map Fields to Sheet Columns
- Write Results to Sheet

### Node Details

#### 11) Parse Perplexity Response
- **Type and technical role:** `code`; extracts and validates the JSON content returned by Perplexity.
- **Configuration choices:**
  - Reads `choices[0].message.content`
  - Removes markdown fences if present
  - Parses JSON
  - Loads metadata from `Build Dynamic Prompt`
  - Creates:
    - row payload fields
    - `not_found_fields`
    - `has_missing`
    - metadata for notifications
- **Key expressions or variables used:**
  - `$input.first().json.choices[0].message.content`
  - `$('Build Dynamic Prompt').item.json.field_map`
  - `$('Build Dynamic Prompt').item.json.field_keys`
  - `$('Build Dynamic Prompt').item.json.name`
  - `$('Build Dynamic Prompt').item.json.row_number`
- **Input and output connections:**  
  - Input: success output of `Perplexity — Research Call`
  - Output: `Map Fields to Sheet Columns`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - API returns unexpected shape
  - Response is not valid JSON
  - Returned JSON omits some keys
  - If a returned value is not a string, `.toLowerCase()` may throw when checking `Not found`
- **Sub-workflow reference:** None.

#### 12) Map Fields to Sheet Columns
- **Type and technical role:** `code`; converts normalized field results into the payload that the Google Sheets node will write.
- **Configuration choices:**
  - Starts with:
    - `row_number`
    - `Status = Done`
    - `Error Log = ''`
  - Iterates configured field keys
  - Uses `field_map[key].column` as the target key name in the outgoing JSON
- **Key expressions or variables used:**
  - `$input.first().json`
  - `field_map`
  - `field_keys`
  - `row_number`
- **Input and output connections:**  
  - Input: `Parse Perplexity Response`
  - Output: `Write Results to Sheet`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - If `column` values do not match actual Google Sheets header names, auto-mapping may fail silently or partially
  - Missing `field_map` entries result in dropped fields
- **Sub-workflow reference:** None.

#### 13) Write Results to Sheet
- **Type and technical role:** `googleSheets`; updates the original row with research results.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Mapping mode: `autoMapInputData`
  - Explicit schema includes:
    - Name
    - Status
    - Current Company
    - Location
    - Current Title
    - Industry
    - Socials / Others
    - Error Log
    - row_number
- **Key expressions or variables used:**
  - Input-driven automapping from previous node
- **Input and output connections:**  
  - Input: `Map Fields to Sheet Columns`
  - Output: `Any Fields Missing?`
- **Version-specific requirements:** Google Sheets type version 4.5.
- **Edge cases or potential failure types:**
  - Auto-mapping depends on input keys matching sheet schema exactly
  - If config tab uses letters like `C`, `D`, etc. rather than actual column headers, these fields will not map to the listed schema
  - Sheet permissions or tab mismatch issues
- **Sub-workflow reference:** None.

---

## 2.6 Block: Missing Data Detection and Slack Notification

### Overview
This block checks whether any configured fields were returned as missing. If yes, it sends a Slack warning; regardless of the result, the loop continues to the next row.

### Nodes Involved
- Any Fields Missing?
- Slack — Missing Fields Alert

### Node Details

#### 14) Any Fields Missing?
- **Type and technical role:** `if`; branches based on whether any requested fields were unresolved.
- **Configuration choices:**
  - Condition checks boolean `true`
  - Left value: `{{ $json.has_missing }}`
- **Key expressions or variables used:**
  - `$json.has_missing`
- **Input and output connections:**  
  - Input: `Write Results to Sheet`
  - True output: `Slack — Missing Fields Alert`
  - False output: `Loop Over Rows`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**
  - The node expects `has_missing` on the incoming item, but `Write Results to Sheet` may not preserve all prior metadata depending on node behavior
  - If metadata is lost before this node, the condition may fail or always evaluate unexpectedly
- **Sub-workflow reference:** None.

**Important implementation note:**  
As currently wired, `Any Fields Missing?` receives data from `Write Results to Sheet`, not directly from `Parse Perplexity Response`. Depending on Google Sheets output behavior, fields like `has_missing`, `name`, and `not_found_fields` may not still be available. If they are not returned by the Sheets node, the missing-field Slack branch may not work reliably. A common fix is to merge or preserve the parse metadata before or after the sheet write.

#### 15) Slack — Missing Fields Alert
- **Type and technical role:** `slack`; alerts the team when Perplexity returned `Not found` for one or more fields.
- **Configuration choices:**
  - Sends to a selected Slack channel
  - Includes:
    - name
    - row number
    - list of missing fields
    - timestamp
- **Key expressions or variables used:**
  - `$json.name`
  - `$json.row_number`
  - `$json.not_found_fields.join(', ')`
  - `$now.toISO()`
- **Input and output connections:**  
  - Input: true branch from `Any Fields Missing?`
  - Output: none
- **Version-specific requirements:** Slack node version 2.3.
- **Edge cases or potential failure types:**
  - If metadata was not preserved through `Write Results to Sheet`, fields like `name` and `not_found_fields` may be unavailable
  - Slack credentials/channel issues
- **Sub-workflow reference:** None.

---

## 2.7 Block: Workflow Documentation Sticky Notes

### Overview
These nodes are non-executable annotations inside the canvas. They document usage, configuration placeholders, and error-handling expectations.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### 16) Sticky Note
- **Type and technical role:** `stickyNote`; visual documentation for overall purpose and setup.
- **Configuration choices:** Contains workflow summary, required credentials, and expected Google Sheet structure.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 17) Sticky Note1
- **Type and technical role:** `stickyNote`; usage instructions and placeholder values to replace.
- **Configuration choices:** Documents how to operate the sheet and which IDs/names to customize.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 18) Sticky Note2
- **Type and technical role:** `stickyNote`; explains API and missing-answer error behavior.
- **Configuration choices:** Describes result of failed API calls and missing fields.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 19) Sticky Note3
- **Type and technical role:** `stickyNote`; explains the config tab and question templating logic.
- **Configuration choices:** Documents use of `[NAME]` placeholder and flow from processing status to Perplexity query.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 20) Sticky Note4
- **Type and technical role:** `stickyNote`; explains parsing and safe row targeting.
- **Configuration choices:** Emphasizes correct column mapping and `row_number` continuity.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 21) Sticky Note5
- **Type and technical role:** `stickyNote`; explains the missing-fields Slack behavior.
- **Configuration choices:** Documents Slack warning intent.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual entry point for the workflow |  | Get All Rows | ## How To Use<br>Open your Google Sheet<br>Type a name in Column A<br>Set Column B status to Pending<br>Run the workflow — results appear in columns C through G automatically<br><br>To change what gets researched, edit the questions in the Config tab. No changes to the workflow needed.<br><br>## Placeholders To Replace<br><br>YOUR_GOOGLE_SHEET_ID<br>YOUR_GOOGLE_CREDENTIAL_ID<br>YOUR_PERPLEXITY_CREDENTIAL_ID<br>YOUR_SLACK_CREDENTIAL_ID<br>YOUR_SLACK_CHANNEL_ID<br>YOUR_SHEET_TAB_NAME<br>YOUR_CONFIG_TAB_NAME |
| Get All Rows | Google Sheets | Read all rows from the main data tab | When clicking 'Execute workflow' | Filter — Pending Only | ## How To Use<br>Open your Google Sheet<br>Type a name in Column A<br>Set Column B status to Pending<br>Run the workflow — results appear in columns C through G automatically<br><br>To change what gets researched, edit the questions in the Config tab. No changes to the workflow needed.<br><br>## Placeholders To Replace<br><br>YOUR_GOOGLE_SHEET_ID<br>YOUR_GOOGLE_CREDENTIAL_ID<br>YOUR_PERPLEXITY_CREDENTIAL_ID<br>YOUR_SLACK_CREDENTIAL_ID<br>YOUR_SLACK_CHANNEL_ID<br>YOUR_SHEET_TAB_NAME<br>YOUR_CONFIG_TAB_NAME |
| Filter — Pending Only | Filter | Keep only rows with Status = Pending | Get All Rows | Loop Over Rows | ## How To Use<br>Open your Google Sheet<br>Type a name in Column A<br>Set Column B status to Pending<br>Run the workflow — results appear in columns C through G automatically<br><br>To change what gets researched, edit the questions in the Config tab. No changes to the workflow needed.<br><br>## Placeholders To Replace<br><br>YOUR_GOOGLE_SHEET_ID<br>YOUR_GOOGLE_CREDENTIAL_ID<br>YOUR_PERPLEXITY_CREDENTIAL_ID<br>YOUR_SLACK_CREDENTIAL_ID<br>YOUR_SLACK_CHANNEL_ID<br>YOUR_SHEET_TAB_NAME<br>YOUR_CONFIG_TAB_NAME |
| Loop Over Rows | Split In Batches | Iterate through pending rows one by one | Filter — Pending Only, Any Fields Missing? | Set Status — Processing | ## How To Use<br>Open your Google Sheet<br>Type a name in Column A<br>Set Column B status to Pending<br>Run the workflow — results appear in columns C through G automatically<br><br>To change what gets researched, edit the questions in the Config tab. No changes to the workflow needed.<br><br>## Placeholders To Replace<br><br>YOUR_GOOGLE_SHEET_ID<br>YOUR_GOOGLE_CREDENTIAL_ID<br>YOUR_PERPLEXITY_CREDENTIAL_ID<br>YOUR_SLACK_CREDENTIAL_ID<br>YOUR_SLACK_CHANNEL_ID<br>YOUR_SHEET_TAB_NAME<br>YOUR_CONFIG_TAB_NAME |
| Set Status — Processing | Google Sheets | Mark current row as Processing | Loop Over Rows | Get Config Tab | ## Creating and formating questions<br><br>The next few nodes are grabbing the status of the spreadsheet from pending to processing.<br><br>Then taking the sheet called "config tab" which is where you will input your questions and you will use the place holder [NAME] as mentioned to plug in questions you want to ask perplexity.<br><br>Then in the Perplexity node it will use an API to ask and return the answeres it has to those questions about the person you are looking for. |
| Get Config Tab | Google Sheets | Read configurable research questions from config tab | Set Status — Processing | Build Dynamic Prompt | ## Creating and formating questions<br><br>The next few nodes are grabbing the status of the spreadsheet from pending to processing.<br><br>Then taking the sheet called "config tab" which is where you will input your questions and you will use the place holder [NAME] as mentioned to plug in questions you want to ask perplexity.<br><br>Then in the Perplexity node it will use an API to ask and return the answeres it has to those questions about the person you are looking for. |
| Build Dynamic Prompt | Code | Create strict JSON research prompt from config rows | Get Config Tab | Perplexity — Research Call | ## Creating and formating questions<br><br>The next few nodes are grabbing the status of the spreadsheet from pending to processing.<br><br>Then taking the sheet called "config tab" which is where you will input your questions and you will use the place holder [NAME] as mentioned to plug in questions you want to ask perplexity.<br><br>Then in the Perplexity node it will use an API to ask and return the answeres it has to those questions about the person you are looking for. |
| Perplexity — Research Call | HTTP Request | Send research prompt to Perplexity API | Build Dynamic Prompt | Parse Perplexity Response, Write Error to Sheet | ## Creating and formating questions<br><br>The next few nodes are grabbing the status of the spreadsheet from pending to processing.<br><br>Then taking the sheet called "config tab" which is where you will input your questions and you will use the place holder [NAME] as mentioned to plug in questions you want to ask perplexity.<br><br>Then in the Perplexity node it will use an API to ask and return the answeres it has to those questions about the person you are looking for. |
| Parse Perplexity Response | Code | Parse returned JSON and detect missing fields | Perplexity — Research Call | Map Fields to Sheet Columns | ## Parse and write answers<br><br>Those individual values are then mapped to their correct column headers, matching exactly what your sheet expects. The row number is carried through this entire process so the write always lands in the right place — never on the wrong person. |
| Map Fields to Sheet Columns | Code | Convert parsed values into a sheet update payload | Parse Perplexity Response | Write Results to Sheet | ## Parse and write answers<br><br>Those individual values are then mapped to their correct column headers, matching exactly what your sheet expects. The row number is carried through this entire process so the write always lands in the right place — never on the wrong person. |
| Write Results to Sheet | Google Sheets | Update the row with completed research results | Map Fields to Sheet Columns | Any Fields Missing? | ## Parse and write answers<br><br>Those individual values are then mapped to their correct column headers, matching exactly what your sheet expects. The row number is carried through this entire process so the write always lands in the right place — never on the wrong person. |
| Any Fields Missing? | If | Check whether any requested field was unresolved | Write Results to Sheet | Slack — Missing Fields Alert, Loop Over Rows | ## Error Handling<br>Similar to the other error log node this one will send a detailed slack message on what rows are experiencing the missing fields error. |
| Slack — Missing Fields Alert | Slack | Send warning when some fields are Not found | Any Fields Missing? |  | ## Error Handling<br>Similar to the other error log node this one will send a detailed slack message on what rows are experiencing the missing fields error. |
| Write Error to Sheet | Google Sheets | Mark row as Error and log API failure details | Perplexity — Research Call | Slack — API Error Alert | ## Error Handling<br>If Perplexity cannot find an answer for a field, the cell is written as "Not found" and a Slack alert is sent with the name and which fields were missing. If the API call fails entirely, the row status is set to Error, the error message is written to Column H, and a separate Slack alert fires. |
| Slack — API Error Alert | Slack | Send Slack alert for API-level failure | Write Error to Sheet |  | ## Error Handling<br>If Perplexity cannot find an answer for a field, the cell is written as "Not found" and a Slack alert is sent with the name and which fields were missing. If the API call fails entirely, the row status is set to Error, the error message is written to Column H, and a separate Slack alert fires. |
| Sticky Note | Sticky Note | Canvas documentation for setup and sheet structure |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for usage and placeholders |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for error handling |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for config question behavior |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for parsing/writing behavior |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation for missing-fields Slack alerts |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## 4.1 Prepare external systems

1. **Create the main Google Sheet**
   - Add a worksheet for the main data table.
   - Include at least these headers:
     - `Name`
     - `Status`
     - `Current Company`
     - `Location`
     - `Current Title`
     - `Industry`
     - `Socials / Others`
     - `Error Log`
   - n8n Google Sheets reads typically expose `row_number`, which is needed for updates.

2. **Create the Config tab**
   - Add another worksheet in the same spreadsheet.
   - Include these headers:
     - `Field Key`
     - `Question Template`
     - `Column`
   - Example rows:
     - `current_company | What company does [NAME] work for? | Current Company`
     - `location | Where is [NAME] located? | Location`
     - `current_title | What is [NAME]'s current job title? | Current Title`
     - `industry | What industry is [NAME] in? | Industry`
     - `socials | List notable profiles or links for [NAME]. | Socials / Others`

   **Recommended:** use actual sheet header names in the `Column` field, not letters like `C`, `D`, etc., because the write node relies on automapped field names.

3. **Create credentials in n8n**
   - **Google Sheets OAuth2**
     - Connect the Google account that owns or can edit the spreadsheet.
   - **HTTP Header Auth** for Perplexity
     - Header name typically: `Authorization`
     - Header value: `Bearer YOUR_API_KEY`
   - **Slack credential**
     - Use a Slack app/token with permission to post to the destination channel.

4. **Get identifiers ready**
   - Google Sheet document ID
   - Main tab ID or tab name
   - Config tab ID or tab name
   - Slack channel ID

---

## 4.2 Build the workflow nodes

### Step 1 — Add the trigger
1. Create a **Manual Trigger** node.
2. Keep the default name or rename it to:
   - `When clicking 'Execute workflow'`

### Step 2 — Add the main sheet reader
3. Create a **Google Sheets** node named `Get All Rows`.
4. Connect the Manual Trigger to it.
5. Configure:
   - Credential: your Google Sheets OAuth2 credential
   - Document: your target spreadsheet
   - Sheet: main data tab
   - Operation: read rows / get rows using default retrieval behavior
   - Enable `Always Output Data`

### Step 3 — Filter to pending rows only
6. Create a **Filter** node named `Filter — Pending Only`.
7. Connect `Get All Rows` to it.
8. Set one condition:
   - Left value: `{{ $json['Status'] }}`
   - Operator: `equals`
   - Right value: `Pending`

### Step 4 — Add the row loop
9. Create a **Split In Batches** node named `Loop Over Rows`.
10. Connect `Filter — Pending Only` to it.
11. Leave batch settings at default if processing one item at a time in loop style.

### Step 5 — Mark current row as Processing
12. Create a **Google Sheets** node named `Set Status — Processing`.
13. Connect the loop iteration output to it.
14. Configure:
   - Credential: Google Sheets OAuth2
   - Operation: `Update`
   - Document: same spreadsheet
   - Sheet: main tab
   - Matching column: `row_number`
   - Fields to update:
     - `Status` = `Processing`
     - `row_number` = `{{ $json['row_number'] }}`

### Step 6 — Read the config tab
15. Create another **Google Sheets** node named `Get Config Tab`.
16. Connect `Set Status — Processing` to it.
17. Configure:
   - Credential: Google Sheets OAuth2
   - Document: same spreadsheet
   - Sheet: config tab

### Step 7 — Build the dynamic prompt
18. Create a **Code** node named `Build Dynamic Prompt`.
19. Connect `Get Config Tab` to it.
20. Paste logic equivalent to:
   - Read current row name from `Loop Over Rows`
   - Read all config rows from current input
   - For each config row:
     - get `Field Key`
     - get `Question Template`
     - get `Column`
     - replace `[NAME]` with the person name
   - Build a JSON schema string with exactly those keys
   - Build a final prompt instructing Perplexity to return only JSON
   - Return:
     - `name`
     - `row_number`
     - `prompt`
     - `field_map`
     - `field_keys`

Core references:
- Current person name: `$('Loop Over Rows').item.json['Name']`
- Current row number: `$('Loop Over Rows').item.json['row_number']`

### Step 8 — Call Perplexity
21. Create an **HTTP Request** node named `Perplexity — Research Call`.
22. Connect `Build Dynamic Prompt` to it.
23. Configure:
   - Method: `POST`
   - URL: `https://api.perplexity.ai/chat/completions`
   - Authentication: Generic Credential Type
   - Generic auth type: HTTP Header Auth
   - Credential: your Perplexity header credential
   - Send body: enabled
   - Body content type: JSON
   - Timeout: `30000`
24. Set JSON body equivalent to:
   - `model: sonar`
   - `messages`:
     - system message saying to return only valid JSON
     - user message containing the generated prompt
25. Use the expression form for prompt content:
   - `{{ JSON.stringify($json.prompt) }}`
26. Set **On Error** to:
   - `Continue (using error output)`

### Step 9 — Parse Perplexity response
27. Create a **Code** node named `Parse Perplexity Response`.
28. Connect the **success output** of `Perplexity — Research Call` to it.
29. Add logic equivalent to:
   - Read `choices[0].message.content`
   - Remove markdown code fences if present
   - `JSON.parse()` the cleaned text
   - Load `field_map`, `field_keys`, `name`, `row_number` from `Build Dynamic Prompt`
   - For every expected key:
     - assign returned value
     - fallback to `Not found`
     - record missing keys in `not_found_fields`
   - Return:
     - all resolved values
     - `name`
     - `row_number`
     - `field_map`
     - `field_keys`
     - `not_found_fields`
     - `has_missing`

### Step 10 — Map fields to sheet columns
30. Create another **Code** node named `Map Fields to Sheet Columns`.
31. Connect `Parse Perplexity Response` to it.
32. Add logic equivalent to:
   - Start payload with:
     - `row_number`
     - `Status = Done`
     - `Error Log = ''`
   - For each configured field:
     - look up `field_map[key].column`
     - set that output property to the resolved value
   - Return the final payload

### Step 11 — Write results back to the sheet
33. Create a **Google Sheets** node named `Write Results to Sheet`.
34. Connect `Map Fields to Sheet Columns` to it.
35. Configure:
   - Credential: Google Sheets OAuth2
   - Operation: `Update`
   - Document: same spreadsheet
   - Sheet: main data tab
   - Matching column: `row_number`
   - Mapping mode: `Auto-map input data`
36. Make sure the sheet schema includes the real headers used in your main tab.

### Step 12 — Check for missing fields
37. Create an **If** node named `Any Fields Missing?`.
38. Connect `Write Results to Sheet` to it.
39. Configure one boolean condition:
   - Left value: `{{ $json.has_missing }}`
   - Operation: `is true`

**Recommended improvement:**  
Because the Google Sheets node may not preserve `has_missing`, `name`, and `not_found_fields`, a more robust design is:
- connect the If node before the Google Sheets write, or
- use a Merge node to preserve metadata, or
- configure downstream nodes based on pre-write data.

If reproducing exactly as JSON, keep the current structure. If reproducing for reliability, preserve metadata explicitly.

### Step 13 — Send Slack alert for missing fields
40. Create a **Slack** node named `Slack — Missing Fields Alert`.
41. Connect the **true** branch of `Any Fields Missing?` to it.
42. Configure:
   - Credential: Slack
   - Resource/action: send message to channel
   - Channel: your Slack channel ID
   - Text should include:
     - `{{ $json.name }}`
     - `{{ $json.row_number }}`
     - `{{ $json.not_found_fields.join(', ') }}`
     - `{{ $now.toISO() }}`

### Step 14 — Close the loop
43. Connect the **false** branch of `Any Fields Missing?` back to `Loop Over Rows`.
44. In the original JSON, only the false branch loops back directly.  
   For better operational reliability, you may also want the missing-fields branch to continue the loop after sending Slack. As provided, the JSON does not connect Slack missing alert back into the loop.

---

## 4.3 Build the API error branch

### Step 15 — Write API failure to sheet
45. Create a **Google Sheets** node named `Write Error to Sheet`.
46. Connect the **error output** of `Perplexity — Research Call` to it.
47. Configure:
   - Operation: `Update`
   - Document: same spreadsheet
   - Sheet: main tab
   - Matching column: `row_number`
   - Fields:
     - `Status = Error`
     - `Error Log = {{ $json.error?.message ?? 'Perplexity API call failed — check credentials or rate limits' }}`
     - `row_number = {{ $('Loop Over Rows').item.json['row_number'] }}`

**Important:**  
Use the same sheet/tab reference as the main update nodes. In the provided JSON this node points to `Sheet1` by name, which may not match the rest of the workflow.

### Step 16 — Send API error Slack message
48. Create a **Slack** node named `Slack — API Error Alert`.
49. Connect `Write Error to Sheet` to it.
50. Configure channel and text including:
   - `{{ $('Loop Over Rows').item.json['Name'] }}`
   - `{{ $('Loop Over Rows').item.json['row_number'] }}`
   - `{{ $json.error?.message ?? 'Unknown error — Perplexity API call failed' }}`
   - `{{ $now.toISO() }}`

---

## 4.4 Optional documentation nodes

51. Add **Sticky Note** nodes for:
   - overall workflow description
   - how to use it
   - error-handling notes
   - config tab explanation
   - parse/write explanation

These are optional for execution but useful for maintenance.

---

## 4.5 Required credentials summary

### Google Sheets OAuth2
- Used by:
  - `Get All Rows`
  - `Set Status — Processing`
  - `Get Config Tab`
  - `Write Results to Sheet`
  - `Write Error to Sheet`

### Perplexity HTTP Header Auth
- Used by:
  - `Perplexity — Research Call`
- Header pattern:
  - `Authorization: Bearer YOUR_API_KEY`

### Slack
- Used by:
  - `Slack — Missing Fields Alert`
  - `Slack — API Error Alert`

---

## 4.6 Expected data contract

### Main sheet input row
Must include at least:
- `Name`
- `Status`

Should support:
- `row_number` for update matching

### Config tab row
Must include:
- `Field Key`
- `Question Template`
- `Column`

### Perplexity expected output
A valid JSON object containing exactly the configured keys, such as:
- `current_company`
- `location`
- `current_title`
- `industry`
- `socials`

---

## 4.7 Known implementation caveats

1. **Config `Column` value mismatch**
   - The code comments suggest values like `C`, but the Google Sheets write node expects header-based automapping.
   - Best practice: set `Column` to exact sheet header names.

2. **Missing-field metadata may be lost**
   - `Any Fields Missing?` is placed after a Google Sheets write node.
   - Depending on node output behavior, `has_missing` and related fields may not still be present.

3. **Error branch tab inconsistency**
   - `Write Error to Sheet` uses `Sheet1` by name, while the rest of the workflow uses a tab ID placeholder.
   - Standardize this.

4. **Loop continuation on missing-fields true branch**
   - The false branch returns to `Loop Over Rows`.
   - The true branch sends Slack but does not explicitly reconnect to the loop in the JSON.
   - You may need to reconnect it to avoid stopping after a warning case.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Perplexity People Researcher: automatically researches people from a Google Sheet using the Perplexity AI API and writes the results back to the same sheet. | Workflow purpose |
| Required credentials: Google Sheets OAuth2, Perplexity API via HTTP Header Auth using `Bearer YOUR_API_KEY`, Slack API. | Setup |
| Main sheet structure: `Name | Status | Current Company | Location | Current Title | Industry | Socials / Others | Error Log` | Google Sheet design |
| Config tab structure: `Field Key | Question Template | Column` | Config design |
| In question templates, use `[NAME]` as a placeholder; it is replaced automatically with the person’s actual name. | Prompt templating |
| Operating steps: open the Google Sheet, type a name in Column A, set Column B to `Pending`, run the workflow, results appear in columns C through G automatically. | Usage |
| To change what gets researched, edit the questions in the Config tab. No workflow logic changes are required. | Maintenance |
| Placeholders to replace: `YOUR_GOOGLE_SHEET_ID`, `YOUR_GOOGLE_CREDENTIAL_ID`, `YOUR_PERPLEXITY_CREDENTIAL_ID`, `YOUR_SLACK_CREDENTIAL_ID`, `YOUR_SLACK_CHANNEL_ID`, `YOUR_SHEET_TAB_NAME`, `YOUR_CONFIG_TAB_NAME`. | Deployment customization |
| Missing-answer behavior: if Perplexity cannot find an answer, the sheet cell is written as `Not found` and a Slack warning is sent. | Error handling |
| API-failure behavior: if the API call fails, the row is marked `Error`, the error is written into `Error Log`, and a Slack alert is sent. | Error handling |