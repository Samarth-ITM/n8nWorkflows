Optimize SEO Meta Tags in Google Sheets with Google Gemini

https://n8nworkflows.xyz/workflows/optimize-seo-meta-tags-in-google-sheets-with-google-gemini-5411


# Optimize SEO Meta Tags in Google Sheets with Google Gemini

### 1. Workflow Overview

This workflow automates the optimization of SEO meta tags (meta titles and descriptions) stored in a Google Sheets document using Google Gemini, an AI language model. It is designed to read rows from the sheet, process each row’s meta tags to ensure they meet SEO length constraints (titles ≤ 60 characters, descriptions ≤ 160 characters), preserve core keywords, and maintain original meaning while shortening only when necessary. The cleaned and optimized meta tags are then written back to the Google Sheet.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Periodically triggers the workflow and fetches current meta tag data from Google Sheets.
- **1.2 Batch Processing & Conditional Check:** Splits data into batches, processes each item, and filters out empty content rows.
- **1.3 AI Meta Tag Optimization:** Sends meta tags to Google Gemini for shortening and SEO optimization using a defined prompt.
- **1.4 Output Parsing & Formatting:** Parses, cleans, and flattens the AI response into a table-ready format.
- **1.5 Data Aggregation & Writing Back:** Aggregates processed results and updates the Google Sheet with optimized meta tags.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

**Overview:**  
This block initiates the workflow on a schedule and retrieves the current meta tag data from a specified Google Sheet.

**Nodes Involved:**  
- Schedule Trigger  
- get-current-tags

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at regular intervals (default is every minute as per empty interval configuration)  
  - Configuration: Interval trigger with default timing (can be customized)  
  - Inputs: None  
  - Outputs: Connects to `get-current-tags`  
  - Edge cases: Misconfiguration of schedule interval could cause unexpected execution frequency.

- **get-current-tags**  
  - Type: Google Sheets (OAuth2)  
  - Role: Reads all rows from a specific sheet identified by URL and sheet ID ("blog_content_sheet")  
  - Configuration: Reads all data without returning only the first match; uses OAuth2 credentials for Google Sheets access  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Passes data to `Loop Over Items`  
  - Edge cases: Authentication failure, API rate limits, missing or renamed sheet/tab could cause errors.

---

#### 2.2 Batch Processing & Conditional Check

**Overview:**  
Splits the retrieved rows into batches for individual processing and filters out rows where the 'Content' field is empty.

**Nodes Involved:**  
- Loop Over Items  
- If

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes rows in batches (default batch size, unspecified here)  
  - Configuration: Default batch size (assumed 1 for per-row processing)  
  - Inputs: Data from `get-current-tags`  
  - Outputs: Two outputs: one to `format-code` node (for further processing) and one to `If` node (for content validation)  
  - Edge cases: Large datasets could increase execution time; unexpected batch size changes may affect performance.

- **If**  
  - Type: Conditional (If)  
  - Role: Checks if the 'Content' field in the current item is empty  
  - Configuration: Uses version 2 conditions; condition checks if `{{ $json['Content'] }}` is empty (string empty)  
  - Inputs: From `Loop Over Items`  
  - Outputs:  
    - True: Sends to `update-tags` (process meta tags)  
    - False: Sends back to `Loop Over Items` for next batch  
  - Edge cases: If 'Content' field is missing, expression might fail; non-string types in 'Content' could cause condition to behave unexpectedly.

---

#### 2.3 AI Meta Tag Optimization

**Overview:**  
Sends meta title and description to the Google Gemini AI model for shortening while maintaining SEO quality and keyword integrity.

**Nodes Involved:**  
- update-tags  
- Google Gemini Chat Model

**Node Details:**

- **update-tags**  
  - Type: Langchain Agent (AI prompt executor)  
  - Role: Sends a formatted prompt to the AI model to shorten meta tags according to SEO rules  
  - Configuration:  
    - Prompt includes input meta_title and meta_description from the current item  
    - Task specifies max lengths: title ≤ 60 chars, description ≤ 160 chars  
    - Instructions to preserve keywords and original meaning, only shorten if necessary  
    - Output expected as flat JSON with keys: `Index`, `meta_titleFixed`, `meta_descriptionFixed`  
  - Inputs: From `If` (only items with non-empty content)  
  - Outputs: To `turn-into-table` for parsing AI output  
  - Credentials: Uses Google Gemini API key (Google PaLM API credentials)  
  - Edge cases: API quota or auth failures, prompt format errors, AI response not matching expected JSON schema, network timeouts.

- **Google Gemini Chat Model**  
  - Type: AI Language Model (Google Gemini)  
  - Role: Processes the AI prompt sent by `update-tags` node  
  - Configuration:  
    - Model: "models/gemini-2.0-flash"  
    - Temperature set to 0.4 for balanced creativity and determinism  
  - Inputs: From `update-tags` (as AI languageModel)  
  - Outputs: Back to `update-tags` node  
  - Credentials: Google Palm API credentials  
  - Edge cases: Model unavailability, API latency, malformed inputs.

---

#### 2.4 Output Parsing & Formatting

**Overview:**  
Cleans the AI response, extracts the JSON content, sanitizes it, flattens nested structures, and converts it into a table-friendly format.

**Nodes Involved:**  
- turn-into-table  
- format-code

**Node Details:**

- **turn-into-table**  
  - Type: Code (JavaScript)  
  - Role: Parses raw AI output, extracts JSON from code fences, removes HTML wrappers and control characters, sanitizes content, and returns a formatted JSON array  
  - Configuration:  
    - Handles multiple output formats (with or without code fences)  
    - Strips DOCTYPE, HTML tags, BOM, and control chars  
    - Attempts fallback parsing with escaping quotes in HTML text  
    - Converts multiline HTML content fields to single-line strings  
  - Inputs: From `update-tags`  
  - Outputs: To `Aggregate` node  
  - Edge cases: Parsing failures, malformed AI output, unexpected HTML content, JSON structure deviations.

- **format-code**  
  - Type: Code (JavaScript)  
  - Role: Flattens nested JSON objects into a flat key-value map suitable for tabular input into Google Sheets  
  - Configuration:  
    - Recursive flattening function removes nested structure prefixes  
    - Converts array of JSON objects into flat rows  
  - Inputs: From `Loop Over Items` (also potentially used after aggregation)  
  - Outputs: To `save-output` node  
  - Edge cases: Deeply nested objects, unexpected data types, empty arrays.

---

#### 2.5 Data Aggregation & Writing Back

**Overview:**  
Aggregates all processed rows into a single dataset and updates the Google Sheet with the optimized meta tags.

**Nodes Involved:**  
- Aggregate  
- save-output

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Collects all incoming items into a single combined data array for batch processing or output  
  - Configuration: Aggregates all item data without filters or transformations  
  - Inputs: From `turn-into-table`  
  - Outputs: To `Loop Over Items` for formatting  
  - Edge cases: Large datasets might hit memory limits; partial data aggregation on errors.

- **save-output**  
  - Type: Google Sheets (OAuth2)  
  - Role: Updates the original Google Sheet rows with the optimized meta title and description fields  
  - Configuration:  
    - Uses "update" operation matching on `row_index` column  
    - Writes to columns `meta_titleFixed` and `meta_descriptionFixed`  
    - Maps many other columns but only updates the fixed meta tags explicitly  
    - OAuth2 credentials for Google Sheets  
  - Inputs: From `format-code` (flattened and formatted output)  
  - Outputs: None (terminal node)  
  - Edge cases: Authentication failure, concurrent sheet edits causing conflicts, missing `row_index` key, API limits.

---

### 3. Summary Table

| Node Name            | Node Type                    | Functional Role                           | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                     |
|----------------------|------------------------------|------------------------------------------|------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger              | Periodic workflow start                   | None                   | get-current-tags               | ## How it works - Reads rows from a connected Google Sheet - Processes each row using Google Gemini to shorten meta titles and descriptions - Ensures SEO constraints (≤ 60 chars for titles, ≤ 160 chars for descriptions) - Preserves core keywords and original meaning - Cleans and flattens Gemini output - Writes results back to the sheet |
| get-current-tags      | Google Sheets                 | Fetches current meta tag data             | Schedule Trigger       | Loop Over Items                | ## Setup steps - Estimated setup time: 10–15 min - Google Sheets OAuth2 credentials - Google Gemini API key - Sheet columns: `row_index`, `meta_title`, `meta_description` - Output columns `meta_titleFixed` & `meta_descriptionFixed` - Error handling & formatting built‑in - Prompt editable in “update-tags” node |
| Loop Over Items       | Split In Batches              | Batch processes each sheet row            | get-current-tags       | format-code, If                |                                                                                               |
| If                   | Conditional                  | Filters out empty content rows            | Loop Over Items        | update-tags (True), Loop Over Items (False) |                                                                                               |
| update-tags           | Langchain Agent (AI prompt) | Sends meta tags to AI for optimization   | If                     | turn-into-table               |                                                                                               |
| Google Gemini Chat Model | AI Language Model            | Executes AI prompt using Gemini model    | update-tags (AI input) | update-tags (AI output)       |                                                                                               |
| turn-into-table       | Code                         | Parses and sanitizes AI JSON output       | update-tags            | Aggregate                     |                                                                                               |
| Aggregate             | Aggregate                    | Combines all processed items              | turn-into-table        | Loop Over Items (format-code) |                                                                                               |
| format-code           | Code                         | Flattens nested JSON to flat structure   | Loop Over Items (Aggregate output) | save-output                  |                                                                                               |
| save-output           | Google Sheets                 | Writes optimized meta tags back to sheet | format-code            | None                          |                                                                                                |
| Sticky — Overview     | Sticky Note                  | Workflow overview and explanation         | None                   | None                         | ## How it works - Reads rows from a connected Google Sheet - Processes each row using Google Gemini to shorten meta titles and descriptions - Ensures SEO constraints (≤ 60 chars for titles, ≤ 160 chars for descriptions) - Preserves core keywords and original meaning - Cleans and flattens Gemini output - Writes results back to the sheet |
| Sticky — Setup        | Sticky Note                  | Setup instructions and requirements       | None                   | None                         | ## Setup steps - Estimated setup time: 10–15 min - Google Sheets OAuth2 credentials - Google Gemini API key - Sheet columns: `row_index`, `meta_title`, `meta_description` - Output columns `meta_titleFixed` & `meta_descriptionFixed` - Error handling & formatting built‑in - Prompt editable in “update-tags” node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to trigger periodically (e.g., every 1 minute or as desired).

2. **Add a Google Sheets node ("get-current-tags"):**  
   - Operation: Read rows from the target Google Sheet.  
   - Configure OAuth2 credentials for Google Sheets access.  
   - Set the Document ID to your sheet URL.  
   - Set the Sheet Name to the tab containing meta tag data (e.g., "blog_content_sheet").  
   - Ensure it reads all rows, not just first match.

3. **Add a Split In Batches node ("Loop Over Items"):**  
   - Connect input from `get-current-tags`.  
   - Use default batch size (process one row at a time).

4. **Add an If node:**  
   - Connect input from `Loop Over Items`.  
   - Condition: Check if `Content` field is empty (expression: `{{$json["Content"]}} is empty`).  
   - True branch: Continue processing.  
   - False branch: Loop back or discard.

5. **Add a Langchain Agent node ("update-tags"):**  
   - Connect True output of `If` node here.  
   - Configure prompt to send meta_title and meta_description for shortening with SEO constraints:  
     - Title max 60 chars  
     - Description max 160 chars  
     - Preserve keywords and meaning, only shorten if necessary  
   - Use Google Gemini Chat Model as AI language model:  
     - Model: `models/gemini-2.0-flash`  
     - Temperature: 0.4  
   - Set Google Gemini API credentials (Google PaLM API key).

6. **Add a Code node ("turn-into-table"):**  
   - Connect from `update-tags`.  
   - Implement JavaScript to parse AI output, extract JSON within code fences, sanitize HTML and control characters, flatten output into an array.

7. **Add an Aggregate node:**  
   - Connect from `turn-into-table`.  
   - Aggregate all items into a single array for bulk processing.

8. **Add another Split In Batches node or reuse "Loop Over Items":**  
   - Process aggregated data batchwise for formatting.

9. **Add a Code node ("format-code"):**  
   - Connect from batch processing node.  
   - Implement JS code to recursively flatten nested JSON objects into flat key-value pairs suitable for Google Sheets.

10. **Add a Google Sheets node ("save-output"):**  
    - Connect from `format-code`.  
    - Operation: Update rows in the same Google Sheet.  
    - Match rows based on `row_index` column.  
    - Update `meta_titleFixed` and `meta_descriptionFixed` columns with AI-optimized values.  
    - Use same OAuth2 credentials as before.

11. **Add sticky notes for documentation:**  
    - One note explaining workflow overview and logic.  
    - Another note listing setup steps, required credentials, sheet schema, and prompt editing location.

12. **Connect nodes accordingly:**  
    - Schedule Trigger → get-current-tags → Loop Over Items → If  
    - If (True) → update-tags → turn-into-table → Aggregate → Loop Over Items (format-code) → save-output  
    - If (False) → Loop Over Items (continue batch processing)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                          | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow reads meta tags from a Google Sheet and uses Google Gemini to optimize SEO meta titles and descriptions by shortening while preserving keywords and meaning.                   | Workflow Overview sticky note                                                                           |
| Setup requires Google Sheets OAuth2 credentials and Google Gemini (Google PaLM) API key. The sheet must have columns `row_index`, `meta_title`, `meta_description`, and output columns. | Setup sticky note                                                                                       |
| The AI prompt is editable in the `update-tags` node for tailoring SEO rules or output format.                                                                                          | Setup sticky note                                                                                       |
| The workflow includes robust JSON parsing and sanitization to handle various AI output formats including embedded HTML and code fences.                                              | `turn-into-table` node JavaScript code comments                                                       |
| Google Gemini model used is `"models/gemini-2.0-flash"` with temperature 0.4 for balanced creativity and consistency.                                                                   | `Google Gemini Chat Model` node configuration                                                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.