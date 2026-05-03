Generate Leads with Google Maps

https://n8nworkflows.xyz/workflows/generate-leads-with-google-maps-2605


# Generate Leads with Google Maps

### 1. Workflow Overview

This workflow, titled **Generate Leads with Google Maps**, automates the extraction of location data from the Google Maps API based on predefined ZIP codes and subcategories. It efficiently manages data retrieval, filtering, deduplication, and storage in Google Sheets, while incorporating robust error handling through exponential backoff retries to gracefully handle API rate limits.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Settings Initialization**: Handles manual or scheduled workflow execution and loads configuration settings.
- **1.2 Data Retrieval Preparation**: Fetches ZIP codes and categories to be queried and filters relevant entries.
- **1.3 Google Maps API Query**: Queries Google Maps API for place data based on ZIP code and category combinations.
- **1.4 Data Processing & Deduplication**: Processes API responses, splits place arrays into individual items, and removes duplicates.
- **1.5 Data Storage & Status Update**: Appends or updates place data into Google Sheets and updates status for ZIP codes/categories.
- **1.6 Error Handling & Retry Logic**: Implements exponential backoff retries with limits and stops workflow on repeated failures.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Settings Initialization

**Overview:**  
This block initiates the workflow either manually or by schedule and loads essential configuration parameters such as Google Sheets URL and sheet names.

**Nodes Involved:**  
- When clicking "Execute Workflow"  
- Run workflow every hours (disabled)  
- Execute Workflow Trigger (disabled)  
- Settings

**Node Details:**  
- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Starts workflow manually on demand  
  - Config: No parameters, single trigger  
  - Output: Proceeds to load settings  
  - Failures: None typical, manual start

- **Run workflow every hours**  
  - Type: Schedule Trigger  
  - Role: Enables scheduled runs every 15 minutes (disabled by default)  
  - Config: Interval set to 15 minutes  
  - Edge Cases: Disabled, so no executions unless enabled

- **Execute Workflow Trigger**  
  - Type: Execute Workflow Trigger (for triggering from other workflows)  
  - Role: Allows external workflow invocation (disabled)  
  - Config: None

- **Settings**  
  - Type: Set Node  
  - Role: Defines key configuration values: Google Sheets URL (`gs_url`), category sheet name (`catSheet`), and ZIP code sheet name (`sheet`)  
  - Config: Hardcoded values for these parameters  
  - Output: Provides settings to subsequent nodes  
  - Edge Cases: Hardcoded URLs require updates for different use cases

---

#### 1.2 Data Retrieval Preparation

**Overview:**  
Retrieves ZIP codes and subcategories from Google Sheets, filters out entries with undesired statuses, and prepares batch processing.

**Nodes Involved:**  
- GS - Get Zip Codes  
- Filter Zips  
- Set Row Number  
- Split Out  
- Limit  
- Loop Zips  
- GS - Get Subcategory  
- Filter Subcategories  
- Subcategory  
- Loop Subcats  
- Set Zip  
- Zips

**Node Details:**  
- **GS - Get Zip Codes**  
  - Type: Google Sheets Read  
  - Role: Reads ZIP codes from the sheet defined in Settings  
  - Config: Uses the sheet name from `Settings.sheet` and document URL from `Settings.gs_url`  
  - Output: Array of ZIP code data objects

- **Filter Zips**  
  - Type: Filter  
  - Role: Filters out ZIP codes with non-empty `status` field (only empty status passes)  
  - Config: Condition: `status` is empty string  
  - Edge Cases: If `status` field missing or irregular, filtering may fail

- **Set Row Number**  
  - Type: Set  
  - Role: Assigns `row_number` field from input JSON for tracking  
  - Config: Copies `row_number` from input

- **Split Out**  
  - Type: Split Out  
  - Role: Splits data by `row_number` field, isolating items for batch processing

- **Limit**  
  - Type: Limit  
  - Role: Restricts the number of processed ZIP code items per run (maxItems=3)  
  - Config: Limits to 3 items for manageable batch size

- **Loop Zips**  
  - Type: Split In Batches  
  - Role: Processes ZIP codes in batches (default batch size)  
  - Output: Two outputs; one proceeds to get subcategories, the other used in nested loops

- **GS - Get Subcategory**  
  - Type: Google Sheets Read  
  - Role: Retrieves subcategories from the configured category sheet  
  - Config: Reads from `Settings.catSheet` sheet and document URL

- **Filter Subcategories**  
  - Type: Filter  
  - Role: Filters out subcategories with `STATUS` set to "Ignore"  
  - Config: Condition: `STATUS` not equals "Ignore"  
  - Edge Cases: Case sensitivity and missing fields may affect filtering

- **Subcategory**  
  - Type: Set  
  - Role: Sets `Subcategory` field for downstream use

- **Loop Subcats**  
  - Type: Split In Batches  
  - Role: Processes subcategories in batches, looping back into ZIP codes

- **Set Zip**  
  - Type: Set  
  - Role: Sets the current `zip` value from the iterated ZIP code batch for the API query

- **Zips**  
  - Type: Set  
  - Role: Assigns `zip` field for filtering and processing

---

#### 1.3 Google Maps API Query

**Overview:**  
This block performs the Google Maps text search API query using combined ZIP code and subcategory strings, handling empty results and preparing data for processing.

**Nodes Involved:**  
- GMaps API  
- If Empty  
- Place Array

**Node Details:**  
- **GMaps API**  
  - Type: HTTP Request  
  - Role: Calls Google Maps Places `searchText` API endpoint via POST  
  - Config:  
    - URL: https://places.googleapis.com/v1/places:searchText  
    - Authentication: Google OAuth2 credential `KBB Google OAuth`  
    - Body: `textQuery` formed by concatenating subcategory and ZIP (e.g., "Subcategory zip")  
    - Headers: `X-Goog-FieldMask` requesting detailed place fields  
  - Output: JSON response with places array  
  - Edge Cases: API rate limits, authentication errors, malformed queries

- **If Empty**  
  - Type: If  
  - Role: Checks if API response `body` is empty or missing  
  - Logic: If empty, loops back to process next subcategory; else proceeds  
  - Edge Cases: Missing or malformed response body

- **Place Array**  
  - Type: Code  
  - Role: Extracts `places` array from API response and outputs each place as a separate item for processing  
  - Logic: Iterates over places array, returns individual place objects with any additional data  
  - Edge Cases: Empty places array logs a message but proceeds gracefully

---

#### 1.4 Data Processing & Deduplication

**Overview:**  
Processes individual place items by setting place IDs, removing duplicates, and preparing data for insertion or update in Google Sheets.

**Nodes Involved:**  
- Set Place ID  
- Remove Duplicates

**Node Details:**  
- **Set Place ID**  
  - Type: Set  
  - Role: Creates a field `places.id` copying from `place.id` for comparison and deduplication  
  - Config: Simple assignment from nested JSON

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate places based on `place.id` or `places.id` fields  
  - Config: Compares on selected fields `"place.id,places.id"`  
  - Edge Cases: Variations in ID formatting may affect deduplication accuracy

---

#### 1.5 Data Storage & Status Update

**Overview:**  
Appends or updates place data into the "Results" Google Sheet and updates the status of ZIP code and subcategory combinations in the original sheet.

**Nodes Involved:**  
- Add rows in Google Sheets  
- GS - Get Status  
- Update Status to Success

**Node Details:**  
- **Add rows in Google Sheets**  
  - Type: Google Sheets AppendOrUpdate  
  - Role: Inserts or updates place data rows in the "Results" sheet  
  - Config:  
    - Document URL from `Settings.gs_url`  
    - Sheet name: "Results"  
    - Matching column: `place_id` to avoid duplicates  
    - Mapped fields: Place attributes such as title, phone, address, rating, GPS coordinates, and many other metadata fields  
  - Edge Cases: Google Sheets API limits, data mapping errors

- **GS - Get Status**  
  - Type: Google Sheets Read  
  - Role: Reads status rows for ZIP and subcategory tracking from configured sheet  
  - Config: Uses same sheet as `Settings.sheet` and URL

- **Update Status to Success**  
  - Type: Google Sheets Update  
  - Role: Updates status to `"scraped"` for the processed ZIP and subcategory  
  - Config: Matches on `zip` column to update row status  
  - Edge Cases: Row matching failure, API limits

---

#### 1.6 Error Handling & Retry Logic

**Overview:**  
Implements exponential backoff retry mechanism to handle API or Google Sheets rate limit errors and stops the workflow after exceeding retry limits.

**Nodes Involved:**  
- Exponential Backoff  
- Exponential Backoff1  
- Exponential Backoff2  
- Wait  
- Wait1  
- Wait2  
- Check Max Retries  
- Check Max Retries1  
- Check Max Retries2  
- Stop and Error  
- Stop and Error1  
- Stop and Error2

**Node Details:**  
- **Exponential Backoff, Exponential Backoff1, Exponential Backoff2**  
  - Type: Code  
  - Role: Calculates retry delay based on retry count using exponential backoff (delay doubles on each retry)  
  - Logic:  
    - Max retries set to 5  
    - Initial delay of 1 second doubled exponentially per retry count  
    - Returns JSON with updated retryCount, waitTimeInSeconds, and status  
  - Edge Cases: Retry count missing or malformed JSON

- **Wait, Wait1, Wait2**  
  - Type: Wait  
  - Role: Pauses workflow execution for the calculated backoff delay before retrying

- **Check Max Retries, Check Max Retries1, Check Max Retries2**  
  - Type: If  
  - Role: Checks if retry count exceeds 10 (double the max retries code), then stops workflow  
  - Edge Cases: Incorrect retry count values may cause premature stops

- **Stop and Error, Stop and Error1, Stop and Error2**  
  - Type: Stop and Error  
  - Role: Stops workflow execution with message indicating Google Sheets API limit exceeded  
  - Edge Cases: Triggered only when retry limits exceeded

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                                  | Input Node(s)                        | Output Node(s)                   | Sticky Note                          |
|----------------------------|----------------------------|-------------------------------------------------|------------------------------------|---------------------------------|------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger            | Manual workflow start                            | None                               | Settings                        | # Triggers                         |
| Run workflow every hours    | Schedule Trigger           | Scheduled workflow start (disabled)             | None                               | Settings                        | # Triggers                         |
| Execute Workflow Trigger    | Execute Workflow Trigger   | External workflow trigger (disabled)            | None                               | Settings                        | # Triggers                         |
| Settings                   | Set                        | Load configuration parameters                    | When clicking..., Run workflow...  | GS - Get Zip Codes              | # Settings                        |
| GS - Get Zip Codes          | Google Sheets              | Read ZIP codes from sheet                         | Settings                          | Zips                           | # Data Prep                       |
| Zips                       | Set                        | Assign ZIP code field                            | GS - Get Zip Codes                | Filter Zips                    | # Data Prep                       |
| Filter Zips                | Filter                     | Filter ZIP codes with empty status               | Zips                             | Set Row Number                 | # Data Prep                       |
| Set Row Number             | Set                        | Assign row number                                | Filter Zips                      | Split Out                     | # Data Prep                       |
| Split Out                  | Split Out                  | Split items by row_number                         | Set Row Number                   | Limit                         | # Data Prep                       |
| Limit                      | Limit                      | Limit items processed per run                     | Split Out                       | Loop Zips                     | # Data Prep                       |
| Loop Zips                  | Split In Batches           | Batch process ZIP codes                           | Limit                           | GS - Get Subcategory, (Loop Subcats) | # Data Prep                       |
| GS - Get Subcategory       | Google Sheets              | Read subcategories                                | Loop Zips                      | Filter Subcategories           | # Data Prep                       |
| Filter Subcategories       | Filter                     | Filter out ignored subcategories                  | GS - Get Subcategory             | Subcategory                   | # Data Prep                       |
| Subcategory                | Set                        | Assign subcategory                                | Filter Subcategories             | Loop Subcats                  | # Data Prep                       |
| Loop Subcats               | Split In Batches           | Batch process subcategories                        | Subcategory                    | Loop Zips, Set Zip            | # Data Prep                       |
| Set Zip                    | Set                        | Assign ZIP code for API query                     | Loop Subcats                   | GMaps API                    | # Data Prep                       |
| GMaps API                  | HTTP Request               | Query Google Maps Places API                       | Set Zip                        | If Empty                     | # Google Maps API                 |
| If Empty                  | If                         | Check if API response is empty                     | GMaps API                     | Loop Subcats (empty), Place Array (not empty) | # Google Maps API                 |
| Place Array                | Code                       | Extract and split places array                      | If Empty                     | Set Place ID                 | # Google Maps API                 |
| Set Place ID               | Set                        | Assign `places.id` from place data                  | Place Array                   | Remove Duplicates            | # Google Maps API                 |
| Remove Duplicates          | Remove Duplicates           | Remove duplicate places based on place IDs          | Set Place ID                  | Add rows in Google Sheets    | # Google Maps API                 |
| Add rows in Google Sheets  | Google Sheets AppendOrUpdate | Append or update place data in "Results" sheet      | Remove Duplicates             | GS - Get Status              | # Google Maps API                 |
| GS - Get Status            | Google Sheets              | Read status from ZIP code sheet                      | Add rows in Google Sheets     | Update Status to Success, Exponential Backoff | # Google Maps API                 |
| Update Status to Success   | Google Sheets Update       | Update status to "scraped" for ZIP and subcategory   | GS - Get Status               | Loop Subcats, Exponential Backoff1 | # Google Maps API                 |
| Exponential Backoff        | Code                       | Calculate exponential backoff delay                 | Add rows in Google Sheets     | Wait                        | # Google Maps API                 |
| Wait                      | Wait                       | Wait for backoff delay                               | Exponential Backoff           | Check Max Retries1           | # Google Maps API                 |
| Check Max Retries1         | If                         | Check if retries exceed limit                         | Wait                         | Stop and Error1, Add rows in Google Sheets | # Google Maps API                 |
| Stop and Error1            | Stop and Error             | Stop workflow on retry limit exceeded                 | Check Max Retries1            | None                        | # Google Maps API                 |
| Exponential Backoff1       | Code                       | Retry delay calculation for status update            | Update Status to Success      | Wait1                       | # Google Maps API                 |
| Wait1                     | Wait                       | Wait for retry delay                                  | Exponential Backoff1          | Check Max Retries            | # Google Maps API                 |
| Check Max Retries          | If                         | Check retry limit for status update                    | Wait1                        | Stop and Error, Update Status to Success | # Google Maps API                 |
| Stop and Error             | Stop and Error             | Stop workflow on retry limit exceeded                   | Check Max Retries             | None                        | # Google Maps API                 |
| Exponential Backoff2       | Code                       | Retry delay calculation for GS - Get Status            | GS - Get Status              | Wait2                       | # Google Maps API                 |
| Wait2                     | Wait                       | Wait for retry delay                                  | Exponential Backoff2          | Check Max Retries2           | # Google Maps API                 |
| Check Max Retries2         | If                         | Check retry limit for GS - Get Status                   | Wait2                        | Stop and Error2             | # Google Maps API                 |
| Stop and Error2            | Stop and Error             | Stop workflow on retry limit exceeded                   | Check Max Retries2            | None                        | # Google Maps API                 |
| Sticky Note20             | Sticky Note                | Author info and product links                           | None                         | None                        | # AlexK1919                      |
| Sticky Note8              | Sticky Note                | Settings title                                         | None                         | None                        | # Settings                       |
| Sticky Note15             | Sticky Note                | Google Sheets setup instructions                        | None                         | None                        | # Settings                       |
| Sticky Note               | Sticky Note                | Triggers title                                        | None                         | None                        | # Triggers                      |
| Sticky Note1              | Sticky Note                | Google Maps API block title                            | None                         | None                        | # Google Maps API               |
| Sticky Note2              | Sticky Note                | Data preparation block title                           | None                         | None                        | # Data Prep                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers**  
   - Add a **Manual Trigger** node named "When clicking \"Execute Workflow\"" with no parameters.  
   - (Optional) Add a **Schedule Trigger** node "Run workflow every hours" with a 15-minute interval, disabled by default.  
   - (Optional) Add an **Execute Workflow Trigger** node "Execute Workflow Trigger", disabled by default.

2. **Settings Node**  
   - Add a **Set** node named "Settings".  
   - Define three string fields:  
     - `gs_url`: Google Sheets URL (e.g., `https://docs.google.com/spreadsheets/d/...`)  
     - `catSheet`: Sheet name for categories (e.g., `"Google Maps Categories"`)  
     - `sheet`: Sheet name for ZIP codes (e.g., `"AZ Zips"`)

3. **Retrieve ZIP Codes**  
   - Add a **Google Sheets** node "GS - Get Zip Codes".  
   - Configure to read from `Settings.sheet` and `Settings.gs_url` with Google OAuth2 credentials.  
   - Set to execute once.

4. **Filter ZIP Codes**  
   - Add a **Filter** node "Filter Zips".  
   - Condition: Field `status` is empty string (`""`).

5. **Assign Row Number**  
   - Add a **Set** node "Set Row Number".  
   - Assign `row_number` field from the input JSON's `row_number`.

6. **Split Out Rows**  
   - Add a **Split Out** node "Split Out" splitting by `row_number`.

7. **Limit Processed ZIPs**  
   - Add a **Limit** node "Limit".  
   - Set `maxItems` to 3 (or desired batch size).

8. **Batch Process ZIPs**  
   - Add a **Split In Batches** node "Loop Zips".

9. **Retrieve Subcategories**  
   - Add a **Google Sheets** node "GS - Get Subcategory".  
   - Configure to read from `Settings.catSheet` and `Settings.gs_url`.

10. **Filter Subcategories**  
    - Add a **Filter** node "Filter Subcategories".  
    - Condition: `STATUS` not equal to `"Ignore"`.

11. **Set Subcategory Field**  
    - Add a **Set** node "Subcategory".  
    - Assign `Subcategory` from input JSON.

12. **Batch Process Subcategories**  
    - Add a **Split In Batches** node "Loop Subcats".

13. **Assign ZIP Code for Query**  
    - Add a **Set** node "Set Zip".  
    - Assign `zip` from the current ZIP code batch.

14. **Google Maps API Query**  
    - Add an **HTTP Request** node "GMaps API".  
    - Configure:  
      - Method: POST  
      - URL: `https://places.googleapis.com/v1/places:searchText`  
      - Authentication: Google OAuth2 (credential with Maps API access)  
      - Headers: `X-Goog-FieldMask` with detailed place fields as per workflow  
      - Body Parameters: `textQuery` set to `={{ $('Subcategory').item.json.Subcategory }} {{ $json.zip }}`

15. **Check Empty Response**  
    - Add an **If** node "If Empty".  
    - Condition: Check if `body` field in response is empty or missing.

16. **Place Array Split**  
    - Add a **Code** node "Place Array".  
    - JavaScript to iterate over `body.places` array and output each place as separate item.

17. **Set Place ID**  
    - Add a **Set** node "Set Place ID".  
    - Assign `places.id` from `place.id`.

18. **Remove Duplicates**  
    - Add a **Remove Duplicates** node "Remove Duplicates".  
    - Compare fields `place.id,places.id`.

19. **Add or Update Google Sheets Rows**  
    - Add a **Google Sheets** node "Add rows in Google Sheets".  
    - Operation: Append or Update  
    - Document URL: From `Settings.gs_url`  
    - Sheet: `"Results"`  
    - Matching column: `place_id`  
    - Map all relevant place fields (title, phone, rating, address, etc.)

20. **Get Status Sheet Data**  
    - Add a **Google Sheets** node "GS - Get Status".  
    - Read from `Settings.sheet` and `Settings.gs_url`.

21. **Update Status to Success**  
    - Add a **Google Sheets** node "Update Status to Success".  
    - Operation: Update  
    - Sheet: From `Settings.sheet`  
    - Matching column: `zip`  
    - Update `status` to `"scraped"` and `subcat` accordingly.

22. **Exponential Backoff Logic for Retries**  
    - Add three separate **Code** nodes ("Exponential Backoff", "Exponential Backoff1", "Exponential Backoff2") with JavaScript implementing exponential backoff delay calculation (max 5 retries, initial delay 1 second, doubling delay).

23. **Wait Nodes for Delays**  
    - Add **Wait** nodes ("Wait", "Wait1", "Wait2") configured to wait for the calculated backoff delay (`waitTimeInSeconds`).

24. **Check Max Retries and Stop on Failure**  
    - Add **If** nodes ("Check Max Retries", "Check Max Retries1", "Check Max Retries2") that stop workflow with **Stop and Error** nodes ("Stop and Error", "Stop and Error1", "Stop and Error2") if retry count exceeds 10.

25. **Connect nodes as per described flow**:  
    - Trigger → Settings → GS - Get Zip Codes → Filter Zips → Set Row Number → Split Out → Limit → Loop Zips → GS - Get Subcategory → Filter Subcategories → Subcategory → Loop Subcats → Set Zip → GMaps API → If Empty → (if empty) Loop Subcats, else Place Array → Set Place ID → Remove Duplicates → Add rows in Google Sheets → GS - Get Status → Update Status to Success → Exponential Backoff1 → Wait1 → Check Max Retries → Stop and Error or Update Status to Success → Loop Subcats and so forth.  
    - Exponential backoff and retry logic is connected after Google Sheets operations and API calls to handle failures.

26. **Credentials Setup**  
    - Configure Google OAuth2 credentials in n8n for Google Sheets and Google Maps API access, ensuring Maps API has access to `places.searchText`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Author: Alex Kim, AI-Native Workflow Automation Architect                                                                                           | https://beacons.ai/alexk1919                                                                                 |
| Product used: Google Maps API via Google Cloud Account                                                                                            | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal |
| Workflow includes robust exponential backoff retry mechanism for API rate limits and Google Sheets API limits                                     | Internal workflow design                                                                                    |
| Google Sheets document must have sheets named as configured in `Settings` node, with ZIP codes, categories, and results sheets                    | User responsibility                                                                                          |
| To modify query parameters, edit `textQuery` field in the GMaps API HTTP Request node                                                           | Workflow customization                                                                                       |
| Scheduled trigger disabled by default; enable and adjust interval as needed                                                                       | Operational flexibility                                                                                       |

---

This comprehensive reference document provides a detailed understanding of the workflow structure, node roles, configurations, and instructions for building or modifying the workflow in n8n for automated lead generation leveraging Google Maps and Google Sheets.