Track Weekly Keyword Rankings with Google SERP, Serper API, and Google Sheets

https://n8nworkflows.xyz/workflows/track-weekly-keyword-rankings-with-google-serp--serper-api--and-google-sheets-8225


# Track Weekly Keyword Rankings with Google SERP, Serper API, and Google Sheets

### 1. Workflow Overview

This workflow is designed to **track weekly keyword rankings** for specified keywords by querying Google Search Engine Results Pages (SERP) using the Serper API, processing the results to find ranking positions and URLs, and then updating a Google Sheet with this ranking data for historical tracking.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Scheduling**: Initiates the workflow every Monday and manually if needed, fetching the list of target keywords from a Google Sheet.
- **1.2 Batch Processing Loop**: Processes keywords one by one with optional delay to avoid API throttling.
- **1.3 SERP Query via Serper API**: Sends keyword search requests to Serper API to retrieve Google SERP results.
- **1.4 Ranking Extraction & Logic**: Parses SERP results to find the domain/page ranking and decides the appropriate date slot to update.
- **1.5 Data Merging & Formatting**: Combines extracted ranking data with existing sheet rows to prepare update payloads.
- **1.6 Google Sheets Update**: Writes back the processed ranking data into the Google Sheet, maintaining historical weekly records.

Sticky notes are included throughout the workflow to provide setup guidance and explanations for each block.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Scheduling

- **Overview:**  
  Triggers the workflow every Monday at midnight and also supports manual execution. Reads the list of target keywords from a specified Google Sheet using service account authentication.

- **Nodes Involved:**  
  - Run Every Monday (Schedule Trigger)  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Targeted Keywords (Google Sheets Read)  
  - Sticky Note (Setup Checklist)  
  - Sticky Note (Keywords)  
  - Sticky Note (Cron)

- **Node Details:**

  - **Run Every Monday**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression set to run at 00:00 every Monday (`0 0 * * MON`)  
    - Input: None  
    - Output: Triggers downstream nodes  
    - Edge cases: Cron misconfiguration could cause missed runs; ensure timezone consistency.

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Configuration: No parameters; allows manual starting of the flow for testing or on-demand runs.  
    - Input: None  
    - Output: Triggers downstream nodes.

  - **Get Targeted Keywords**  
    - Type: Google Sheets (Read)  
    - Configuration:  
      - Document ID and Sheet Name set to the sheet containing keywords.  
      - Authentication via Google Service Account.  
      - Reads all rows including columns such as `Keyword`, `Target Page`, and `Sr.no`.  
    - Input: Trigger from schedule/manual trigger  
    - Output: Outputs all keyword rows as JSON items for further processing.  
    - Edge cases:  
      - Google API auth errors (service account misconfiguration).  
      - Empty or malformed sheet data could cause downstream errors.

  - **Sticky Notes**  
    - Provide important setup instructions such as credential setup, sheet ID/name replacement, API key setup, and workflow trigger info.

---

#### 1.2 Batch Processing Loop

- **Overview:**  
  Processes each keyword row individually by splitting the input array into batches of one, allowing for paced sequential processing and controlled delays between API calls.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Delay Between Keyword Queries (Wait)  
  - Sticky Note (Serper)

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Default batch size of 1 (process one keyword at a time). No options set.  
    - Input: Array of keyword rows from "Get Targeted Keywords"  
    - Output: Single keyword row per execution cycle.  
    - Edge cases: Large keyword lists may increase total execution time.

  - **Delay Between Keyword Queries**  
    - Type: Wait  
    - Configuration: Waits 10 seconds between batches to avoid API throttling or rate limits.  
    - Input: Output from Loop Over Items (second output stream)  
    - Output: Continues workflow after delay.  
    - Edge cases: Delay duration too short may cause API rate limit errors; too long increases runtime.

  - **Sticky Note (Serper)**  
    - Explains the Serper API usage, required API key credential, and parameters for the query.

---

#### 1.3 SERP Query via Serper API

- **Overview:**  
  For each keyword, submits a POST request to Serper API's Google SERP endpoint with parameters to retrieve up to 100 organic results localized to India (`gl=in`).

- **Nodes Involved:**  
  - Keyword Searching on Google (HTTP Request)

- **Node Details:**

  - **Keyword Searching on Google**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://google.serper.dev/search`  
      - Body Parameters:  
        - `q`: Keyword field from current input item  
        - `gl`: Geographic location code, set to "in" (India)  
        - `num`: Number of results, set to 100  
      - Headers:  
        - `X-API-KEY`: Fetched from n8n credentials (`serperApiKey`)  
        - `Content-Type`: `application/json`  
    - Input: Single keyword item from Loop Over Items  
    - Output: SERP JSON response including organic results to be parsed downstream.  
    - Edge cases:  
      - API authentication errors (invalid or missing API key).  
      - API rate limits or request failures (network issues or Serper service downtime).  
      - Unexpected response format changes.

---

#### 1.4 Ranking Extraction & Logic

- **Overview:**  
  Parses SERP organic results to find the ranking position and URL of the specified target domain/page. Updates the Google Sheet data structure to store weekly ranking snapshots in one of 12 date slots.

- **Nodes Involved:**  
  - Extracting Ranking & URL (Code)

- **Node Details:**

  - **Extracting Ranking & URL**  
    - Type: Code (JavaScript)  
    - Configuration Details:  
      - Retrieves the row data from "Get Targeted Keywords".  
      - Uses IST timezone to determine today's date.  
      - Extracts target domain from the "Target Page" column or uses a hardcoded fallback domain.  
      - Iterates over SERP organic results to find first occurrence of target domain or exact URL.  
      - Assigns position rank (1-based) or defaults to 100 if not found.  
      - Finds an available slot (1 to 12) in the sheet columns where today's date can be recorded, or updates existing slot if today’s date is already present.  
      - Optionally overwrites last slot if all are filled (configurable via `OVERWRITE_IF_ALL_FILLED`).  
      - Returns an object with rowNumber and updated ranking, URL, and date fields for the correct slot.  
    - Inputs: HTTP response from Serper API, and original sheet row data via node reference.  
    - Outputs: JSON object with update payload for the Google Sheet.  
    - Edge cases:  
      - Missing `rowNumber` in input row data causes error output.  
      - No domain match found sets position to 100 and URL to null.  
      - All date slots filled without overwrite enabled returns error object.  
      - Date/timezone mismatches affecting slot assignment.

---

#### 1.5 Data Merging & Formatting

- **Overview:**  
  Merges updated ranking data from extraction node with the original keyword row data to produce a full updated row payload for Google Sheets update.

- **Nodes Involved:**  
  - Merging Result (Merge)  
  - Formating output (Code)  
  - Sticky Note (Formatting)

- **Node Details:**

  - **Merging Result**  
    - Type: Merge  
    - Configuration: Defaults (merges two input streams by index).  
    - Inputs:  
      - Update payload from "Extracting Ranking & URL"  
      - Original row from "Get Targeted Keywords" (passed through delay node)  
    - Outputs: Combined data to formatting node.

  - **Formating output**  
    - Type: Code (JavaScript)  
    - Configuration Details:  
      - Accepts two inputs: update payload and original row.  
      - Validates presence of `rowNumber`.  
      - Copies all existing columns from the original row.  
      - Overlays only updated ranking, URL, and date fields from the update payload.  
      - Preserves null values to maintain link absence if domain not found.  
      - Returns a single consolidated JSON object representing the full row update.  
    - Edge cases: Missing inputs or malformed data cause error output.

---

#### 1.6 Google Sheets Update

- **Overview:**  
  Updates the Google Sheet rows with the newly computed ranking data, preserving all existing columns and updating only the relevant ranking and date columns.

- **Nodes Involved:**  
  - Update Rank and URLs (Google Sheet)  
  - Sticky Note (Update Sheet)

- **Node Details:**

  - **Update Rank and URLs (Google Sheet)**  
    - Type: Google Sheets (Update)  
    - Configuration:  
      - Document ID and Sheet Name matching the source sheet (to update the same rows).  
      - Authentication via Google Service Account.  
      - Mapping: Matches rows by `Sr.no` column.  
      - Updates columns: `Ranking Date 1` to `Ranking Date 12`, `Ranking URL Date 1` to `Ranking URL Date 12`, `Date 1` to `Date 12`, along with all other columns from the row.  
      - Operation: Update existing rows based on `Sr.no`.  
    - Input: Consolidated row JSON from "Formating output".  
    - Output: Confirmation or error from Google Sheets API.  
    - Edge cases:  
      - Incorrect sheet ID or name causes failure.  
      - Authentication errors.  
      - Mapping errors if `Sr.no` is missing or duplicated.

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                     | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                     |
|-------------------------------|-----------------------|-----------------------------------|-----------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Run Every Monday               | Schedule Trigger      | Weekly Cron Trigger                | None                        | Get Targeted Keywords           | ## Cron Trigger - Runs automatically every Monday at 00:00 (midnight) - Adjust CRON if needed  |
| When clicking ‘Execute workflow’ | Manual Trigger       | Manual workflow start              | None                        | Get Targeted Keywords           |                                                                                                |
| Get Targeted Keywords          | Google Sheets (Read)  | Read keywords and target pages     | Run Every Monday, Manual     | Loop Over Items                 | ## Google Sheets Setup (Read Keywords) - Replace your Google Sheet ID and sheet name - Connect Google Service Account - Sheet must include at least: `Keyword`, `Target Page`, `Sr.no` |
| Loop Over Items               | Split In Batches      | Process keywords one by one        | Get Targeted Keywords        | Delay Between Keyword Queries, Loop Over Items (2nd output) |                                                                                                |
| Delay Between Keyword Queries | Wait                  | Wait 10 seconds between queries    | Loop Over Items (2nd output) | Keyword Searching on Google     |                                                                                                |
| Keyword Searching on Google    | HTTP Request          | Query Serper API for Google SERP   | Delay Between Keyword Queries | Extracting Ranking & URL        | ## Serper API Setup - Queries Google SERP results using Serper API - Add your Serper API key in n8n credentials (`serperApiKey`) - Docs: https://serper.dev - Parameters: q=Keyword, gl=country code, num=100 |
| Extracting Ranking & URL       | Code (JavaScript)     | Parse SERP results and find rank   | Keyword Searching on Google  | Merging Result                 | ## Extracting Ranking & URL - Parses SERP results and finds your domain/page - If found → saves rank + URL in sheet - If not found → sets position=100 - Default domain is set in `HARDCODED_DOMAIN` |
| Merging Result                | Merge                 | Combine update payload and row     | Extracting Ranking & URL, Delay Between Keyword Queries | Formating output               |                                                                                                |
| Formating output              | Code (JavaScript)     | Merge updated data with original row | Merging Result              | Update Rank and URLs (Google Sheet) | ## Formatting Output - Merges SERP ranking results with the existing Google Sheet row - Ensures only updated rank/date/url fields overwrite - Keeps other columns intact |
| Update Rank and URLs (Google Sheet) | Google Sheets (Update) | Update Google Sheet with new rankings | Formating output            | Loop Over Items                | ## Google Sheets Setup (Write Rankings) - Replace your Google Sheet ID and sheet name - Connect Google Service Account credentials - Workflow updates columns Ranking Date/URL/Date 1..12 |
| Sticky Note (Setup Checklist) | Sticky Note           | Setup instructions                 | None                        | None                          | ## Setup Checklist - Add your Google Service Account credentials in n8n - Replace `your-google-sheet-id` and `your-sheet-name` - Add Serper API key - Optional: update `HARDCODED_DOMAIN` - Run manually or schedule |
| Sticky Note (Keywords)        | Sticky Note           | Notes on Google Sheet setup        | None                        | None                          | ## Google Sheets Setup (Read Keywords) - Replace Google Sheet ID and name - Connect service account - Required columns: `Keyword`, `Target Page`, `Sr.no` |
| Sticky Note (Serper)          | Sticky Note           | Notes on Serper API usage          | None                        | None                          | ## Serper API Setup - Add Serper API key in credentials - Docs: https://serper.dev - Parameters q, gl, num |
| Sticky Note (Extract)         | Sticky Note           | Explanation of ranking extraction  | None                        | None                          | ## Extracting Ranking & URL - Parses SERP to find domain/page - Position=100 if not found - `HARDCODED_DOMAIN` used for domain detection |
| Sticky Note (Formatting)      | Sticky Note           | Explains output formatting logic   | None                        | None                          | ## Formatting Output - Merges SERP results with sheet row - Overwrites only updated fields - Preserves null values |
| Sticky Note (Update Sheet)    | Sticky Note           | Notes on Google Sheet update       | None                        | None                          | ## Google Sheets Setup (Write Rankings) - Replace Google Sheet ID and name - Connect service account - Updates Ranking Date/URL/Date columns |
| Sticky Note (Cron)            | Sticky Note           | Notes on cron trigger              | None                        | None                          | ## Cron Trigger - Runs every Monday at midnight - Adjust as needed |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Run Every Monday`  
   - Set Cron expression: `0 0 * * MON` (runs every Monday at midnight)  
   - No credentials needed.

2. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Used for manual execution.

3. **Create a Google Sheets node (Read operation)**  
   - Name: `Get Targeted Keywords`  
   - Operation: Read rows  
   - Credentials: Connect Google Service Account credentials  
   - Document ID: Set your Google Sheet ID containing keywords  
   - Sheet Name: Set sheet name containing the data  
   - Ensure the sheet has columns: `Keyword`, `Target Page`, `Sr.no`  
   - Connect outputs of `Run Every Monday` and `When clicking ‘Execute workflow’` to this node.

4. **Create a SplitInBatches node**  
   - Name: `Loop Over Items`  
   - Batch Size: 1 (process one keyword at a time)  
   - Connect `Get Targeted Keywords` output to this node input.

5. **Create a Wait node**  
   - Name: `Delay Between Keyword Queries`  
   - Wait for 10 seconds (to prevent API overload)  
   - Connect second output (next batch) of `Loop Over Items` to this node.

6. **Create an HTTP Request node**  
   - Name: `Keyword Searching on Google`  
   - Method: POST  
   - URL: `https://google.serper.dev/search`  
   - Authentication: None (use API key in headers)  
   - Headers:  
     - `X-API-KEY`: Use credential variable `{{ $credentials.serperApiKey }}`  
     - `Content-Type`: `application/json`  
   - Body Parameters (JSON):  
     - `q`: Expression `{{$json.Keyword}}`  
     - `gl`: `"in"` (or your country code)  
     - `num`: `"100"`  
   - Connect output of `Delay Between Keyword Queries` to this node.

7. **Create a Code node**  
   - Name: `Extracting Ranking & URL`  
   - Paste the provided JavaScript code that:  
     - Extracts today's date in IST  
     - Finds rank and URLs from SERP results matching target domain or URL  
     - Chooses the correct date slot in the sheet (1 to 12)  
     - Returns update payload with rank, URL, and date fields for that slot.  
   - Ensure the node references the original sheet row via node reference (`$('Get Targeted Keywords').first().json`)  
   - Connect output of `Keyword Searching on Google` to this node.

8. **Create a Merge node**  
   - Name: `Merging Result`  
   - Mode: Merge by index (default)  
   - Connect output of `Extracting Ranking & URL` to input A  
   - Connect output of `Delay Between Keyword Queries` (original row) to input B

9. **Create a Code node**  
   - Name: `Formating output`  
   - Paste the JavaScript code that merges the update payload with the original sheet row:  
     - Copies all original columns  
     - Overwrites only updated rank/date/url fields  
     - Preserves null values  
   - Connect output of `Merging Result` to this node.

10. **Create a Google Sheets node (Update operation)**  
    - Name: `Update Rank and URLs (Google Sheet)`  
    - Operation: Update  
    - Credentials: Use same Google Service Account credentials  
    - Document ID and Sheet Name: Same as in step 3  
    - Set matching columns: `Sr.no` (must exist and be unique in the sheet)  
    - Map all columns including `Ranking Date 1..12`, `Ranking URL Date 1..12`, `Date 1..12`, and other sheet columns from the incoming data  
    - Connect output of `Formating output` to this node.

11. **Connect the output of** `Update Rank and URLs (Google Sheet)` **back to** `Loop Over Items` **to continue processing next batch.**

12. **Add Sticky Notes** at positions corresponding to each block with the contents:  
    - Setup details (Google credentials, API keys, sheet ID/name replacements)  
    - Google Sheet reading and writing instructions  
    - Serper API usage and documentation link: https://serper.dev  
    - Explanation of code nodes and cron trigger

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                      |
|--------------------------------------------------------------------------------------------------------------|------------------------------------|
| Workflow tracks keyword rankings weekly via Google SERP and logs data in Google Sheets for historical tracking. | Overall workflow purpose            |
| Serper API documentation and usage details: https://serper.dev                                               | Serper API reference and setup     |
| Use Google Service Account credentials for secure and scalable Google Sheets API access.                      | Google Sheets node authentication  |
| Ensure `Sr.no` column in the Google Sheet is unique and used as matching key for updates.                     | Google Sheets update node matching  |
| Timezone is set to IST (+5:30) for date slot assignment; adjust code if your timezone differs.                 | Code node timezone handling         |
| The workflow avoids overwriting previous weeks unless the date matches today; can be configured to overwrite. | Ranking extraction logic            |
| Delay of 10 seconds between API calls prevents hitting Serper API rate limits. Adjust as per your quota.      | Delay node configuration           |
| The workflow runs automatically every Monday at midnight, but can be executed manually for testing.           | Schedule trigger configuration     |
| You may customize the `HARDCODED_DOMAIN` variable in the extraction code for domain detection fallback.       | Code node variable customization   |

---

**Disclaimer:**  
The text provided is extracted solely from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.