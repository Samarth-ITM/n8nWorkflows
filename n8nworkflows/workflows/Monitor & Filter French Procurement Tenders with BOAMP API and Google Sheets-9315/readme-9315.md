Monitor & Filter French Procurement Tenders with BOAMP API and Google Sheets

https://n8nworkflows.xyz/workflows/monitor---filter-french-procurement-tenders-with-boamp-api-and-google-sheets-9315


# Monitor & Filter French Procurement Tenders with BOAMP API and Google Sheets

### 1. Workflow Overview

This n8n workflow automates the monitoring and filtering of French public procurement tenders using the BOAMP API and Google Sheets. It is designed to fetch tender data, filter it based on user-configured market types and keywords, and store relevant tenders in a Google Sheets document for further review.

The workflow is composed of two major logical blocks:

- **1.1 Big Phase 1: Tender Retrieval and Storage**  
  This phase retrieves batches of tenders from the BOAMP API with pagination, filters them by configured market types (Works, Services, Supplies), formats the data, and saves it into a Google Sheet. It manages offset pagination to retrieve all relevant tenders from the specified time period.

- **1.2 Big Phase 2: Tender Keyword Filtering and Targeting**  
  This phase reads all retrieved tenders from the Google Sheet, filters out already processed tenders, downloads and extracts text from tender PDFs, performs keyword matching based on configured search terms, and appends matched tenders to a target Google Sheet for curated opportunities. It marks tenders as processed to avoid duplicate analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Big Phase 1: Tender Retrieval and Storage

**Overview:**  
Fetches tenders from the BOAMP API in paginated batches according to configured time period and market types, formats the data, stores all tenders in a Google Sheet, and manages pagination offsets.

**Nodes Involved:**  
- Schedule Trigger1  
- Get config  
- Get Offset  
- HTTP Request  
- Tenders sorting  
- Format Results  
- Append row in sheet  
- Process Response  
- Update offset  
- Check Continue Loop  
- Wait  
- Reset Offset  

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow monthly at 8:00 AM using cron expression `0 8 1 * *`.  
  - Input: None  
  - Output: Triggers "Get config" node.  
  - Edge cases: Misconfigured cron can cause missed runs.  
  - Sticky Note: Phase 0 configuration instructions provided.

- **Get config**  
  - Type: Google Sheets (Read)  
  - Role: Reads user configuration from the "Config" sheet (market types to monitor, search period in days).  
  - Input: Trigger  
  - Output: Configuration JSON object to "Get Offset".  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Missing or malformed config rows can cause errors.  
  - Sticky Note: Explains purpose and expected config structure.

- **Get Offset**  
  - Type: Google Sheets (Read)  
  - Role: Retrieves current pagination offset from "Offset" sheet to continue fetching batches.  
  - Input: Config data  
  - Output: Offset number to "HTTP Request" node.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Missing or invalid offset value may cause incorrect pagination.  
  - Sticky Note: Describes pagination control via offset.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls BOAMP API to fetch tenders with parameters: select fields, order by publication date descending, date filter based on config period, limit 100, offset from Google Sheet.  
  - Input: Offset, Config  
  - Output: Raw API response JSON to "Tenders sorting".  
  - Headers: Accept application/json, User-Agent n8n workflow.  
  - Edge cases: API downtime, rate limiting, request timeouts, malformed responses.  
  - Sticky Note: Details API call parameters and expected data.

- **Tenders sorting**  
  - Type: Code  
  - Role: Filters API results to keep only tenders matching configured market types (Travaux, Services, Fournitures).  
  - Input: API response JSON, Config  
  - Output: Filtered results JSON to "Format Results".  
  - Logic: Checks if tender market types exist and match enabled config types, throws error if no type selected.  
  - Edge cases: Missing or unexpected data structure, empty results.  
  - Sticky Note: Explains filtering logic and config dependency.

- **Format Results**  
  - Type: Code  
  - Role: Formats filtered tenders for Google Sheets: formats dates (dd/MM/yyyy), generates PDF URLs from tender web IDs and dates, maps fields to sheet columns.  
  - Input: Filtered tenders  
  - Output: Items array with formatted JSON to "Append row in sheet".  
  - Edge cases: Invalid dates or URLs fallback gracefully.  
  - Sticky Note: Describes data formatting and URL generation.

- **Append row in sheet**  
  - Type: Google Sheets (Append)  
  - Role: Appends formatted tender records into the "All" sheet for storage.  
  - Input: Formatted tender data  
  - Output: Confirmation to "Process Response".  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: API quota limits, sheet access permissions.  
  - Sticky Note: Marks data persistence step.

- **Process Response**  
  - Type: Code  
  - Role: Processes API response metadata to calculate next offset for pagination, checks if more data is available, prepares control variables for loop.  
  - Input: API response  
  - Output: Offset and control flags to "Update offset".  
  - Edge cases: Missing total_count or results fields.  
  - Sticky Note: Explains pagination calculation.

- **Update offset**  
  - Type: Google Sheets (Update)  
  - Role: Updates the "Offset" sheet with the new offset value for the next API call batch.  
  - Input: Next offset from "Process Response"  
  - Output: Confirmation to "Check Continue Loop".  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Update failure due to connectivity or permission issues.  
  - Sticky Note: Offset update description.

- **Check Continue Loop**  
  - Type: If  
  - Role: Decision node to check if more tenders are to be fetched based on `hasMoreData` boolean.  
  - Input: Control flags from "Update offset"  
  - Output:  
    - True: Triggers "Wait" (pause before next API call).  
    - False: Triggers "Reset Offset" (pagination complete).  
  - Edge cases: Incorrect flag could cause infinite loops or premature termination.  
  - Sticky Note: Loop control explanation.

- **Wait**  
  - Type: Wait  
  - Role: Delays execution by 10 seconds to avoid hitting API rate limits between paginated calls.  
  - Input: Continue loop trigger  
  - Output: Loops back to "Get Offset" for next batch.  
  - Edge cases: Workflow pause may be interrupted if n8n is restarted.  
  - Sticky Note: Rate limit handling.

- **Reset Offset**  
  - Type: Google Sheets (Update)  
  - Role: Resets pagination offset to 0 in the "Offset" sheet after all data is retrieved.  
  - Input: End of loop trigger  
  - Output: Workflow ends or idle until next schedule.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Update failure prevents fresh start for next run.  
  - Sticky Note: Marks completion of Big Phase 1.

---

#### 2.2 Big Phase 2: Tender Keyword Filtering and Targeting

**Overview:**  
Processes tenders retrieved and stored in Big Phase 1 by filtering out already processed entries, downloading tender PDFs, extracting text, matching keywords, and saving matches to a target sheet.

**Nodes Involved:**  
- Schedule Trigger  
- Get keyword  
- Get All  
- Filter  
- Loop Over offres  
- HTTP Request4  
- Extract from File  
- Get query  
- Query match ?  
- Target offre  
- Ok  
- Wait1  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates keyword filtering phase monthly on the 1st at 10:00 AM (`0 10 1 * *`).  
  - Input: None  
  - Output: Triggers "Get keyword" node.  
  - Edge cases: Misconfigured cron causes skipped runs.  
  - Sticky Note: Phase 2 overview.

- **Get keyword**  
  - Type: Google Sheets (Read)  
  - Role: Reads configured keyword list from the "Config" sheet.  
  - Input: Schedule Trigger  
  - Output: Keyword JSON array to "Get All".  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Empty or malformed keyword list disables filtering.  
  - Sticky Note: Keywords loading description.

- **Get All**  
  - Type: Google Sheets (Read)  
  - Role: Retrieves all tenders from the "All" sheet populated by Big Phase 1.  
  - Input: Keywords  
  - Output: Tender list to "Filter".  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Large data sets may cause performance issues.  
  - Sticky Note: Retrieves all tenders for processing.

- **Filter**  
  - Type: If / Filter  
  - Role: Filters tenders to exclude those marked as already processed (where "Ok" column ≠ "Ok").  
  - Input: Tender list  
  - Output: New tenders to "Loop Over offres".  
  - Edge cases: Incorrect filtering can cause reprocessing or misses.  
  - Sticky Note: Avoids duplicate processing.

- **Loop Over offres**  
  - Type: Split In Batches  
  - Role: Processes tenders one by one sequentially for stability and API limits.  
  - Input: Filtered tender list  
  - Output: Individual tender items to "HTTP Request4" (PDF download).  
  - Parameters: Batch size implicitly 1.  
  - Edge cases: Lengthy processing time for large datasets.  
  - Sticky Note: Sequential processing explanation.

- **HTTP Request4**  
  - Type: HTTP Request  
  - Role: Downloads the PDF document for the current tender using the generated URL.  
  - Input: Single tender with URL  
  - Output: Binary PDF data to "Extract from File".  
  - Edge cases: Broken URLs, network failures, 404 errors.  
  - Sticky Note: Tender PDF download.

- **Extract from File**  
  - Type: Extract From File  
  - Role: Extracts text content from downloaded PDF to prepare for keyword matching.  
  - Input: Binary PDF data  
  - Output: Text content JSON to "Get query".  
  - Edge cases: PDF parsing failures, encrypted or malformed PDFs.  
  - Sticky Note: Text extraction from tender PDFs.

- **Get query**  
  - Type: Code  
  - Role: Normalizes extracted tender text and keywords, performs keyword matching, logs match details including position and context.  
  - Input: Extracted text, Keywords from config  
  - Output: Matching result JSON to "Query match ?".  
  - Logic: Uses accent removal, lowercasing, substring search.  
  - Edge cases: Empty text or keywords disables matching.  
  - Sticky Note: Keyword matching logic.

- **Query match ?**  
  - Type: If  
  - Role: Routes tender based on match presence:  
    - If matched keywords found → "Target offre" (save to target sheet).  
    - Else → "Wait1" (short delay before next tender).  
  - Input: Matching result  
  - Output: Conditional routing  
  - Edge cases: False negatives/positives due to text normalization.  
  - Sticky Note: Tender relevance decision.

- **Target offre**  
  - Type: Google Sheets (Append)  
  - Role: Appends matched tender data with matched keywords and current date to the "Target" sheet for curated opportunities.  
  - Input: Matched tender data  
  - Output: Confirmation back to "Loop Over offres" (to continue processing).  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Sheet quota limits, data format issues.  
  - Sticky Note: Saves relevant tenders for review.

- **Ok**  
  - Type: Google Sheets (Update)  
  - Role: Updates "All" sheet to mark the tender as processed ("Ok" status) to prevent reprocessing.  
  - Input: Matching result from "Get query"  
  - Output: Confirmation to "Query match ?" (loop continuation).  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Update failure causes reprocessing or data inconsistency.  
  - Sticky Note: Processing status tracking.

- **Wait1**  
  - Type: Wait  
  - Role: Short wait (default no explicit delay) after processing a tender without match before continuing loop.  
  - Input: No match path from "Query match ?"  
  - Output: Loops back to "Loop Over offres" for next tender.  
  - Edge cases: None significant.  
  - Sticky Note: Flow control delay for stability.

---

### 3. Summary Table

| Node Name          | Node Type                | Functional Role                                  | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                   |
|--------------------|--------------------------|-------------------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger1   | Schedule Trigger         | Initiates Big Phase 1 monthly at 8:00 AM        | None                   | Get config              | Phase 0: Initial configuration setup instructions                                            |
| Get config         | Google Sheets (Read)     | Reads user market type and period configuration | Schedule Trigger1      | Get Offset              | Loads configuration parameters and search period                                             |
| Get Offset         | Google Sheets (Read)     | Retrieves current pagination offset              | Get config             | HTTP Request            | Controls pagination offset retrieval                                                         |
| HTTP Request       | HTTP Request             | Calls BOAMP API to fetch tenders batch           | Get Offset             | Tenders sorting         | Fetches tenders with filters: date, offset, limit                                            |
| Tenders sorting    | Code                     | Filters tenders by market types from config      | HTTP Request           | Format Results          | Keeps only tenders matching selected market types                                            |
| Format Results     | Code                     | Formats tenders for sheet, generates PDF URLs    | Tenders sorting        | Append row in sheet     | Formats data, date conversion, URL generation                                                |
| Append row in sheet| Google Sheets (Append)   | Appends formatted tenders into "All" sheet       | Format Results         | Process Response        | Saves tenders for full dataset storage                                                       |
| Process Response   | Code                     | Prepares pagination variables for next batch     | Append row in sheet    | Update offset           | Calculates next offset, total counts, hasMore flag                                           |
| Update offset      | Google Sheets (Update)   | Updates offset in sheet for pagination            | Process Response       | Check Continue Loop     | Saves next offset for continuation                                                          |
| Check Continue Loop| If                       | Decides if more data to fetch or reset offset    | Update offset          | Wait / Reset Offset     | Controls loop continuation or termination                                                   |
| Wait               | Wait                     | Delays before next API call                        | Check Continue Loop    | Get Offset              | Prevents API rate limit issues                                                               |
| Reset Offset       | Google Sheets (Update)   | Resets offset to zero after full retrieval        | Check Continue Loop    | None                    | Completes Big Phase 1                                                                        |
| Schedule Trigger   | Schedule Trigger         | Initiates Big Phase 2 monthly at 10:00 AM         | None                   | Get keyword             | Starts keyword filtering phase                                                               |
| Get keyword        | Google Sheets (Read)     | Reads tender filtering keywords                    | Schedule Trigger       | Get All                 | Loads keywords for content matching                                                         |
| Get All            | Google Sheets (Read)     | Retrieves all tenders from "All" sheet             | Get keyword            | Filter                  | Provides tenders for analysis                                                                |
| Filter             | If                       | Filters out already processed tenders              | Get All                | Loop Over offres        | Prevents duplicate processing                                                                |
| Loop Over offres   | Split In Batches         | Processes tenders sequentially                      | Filter                 | HTTP Request4           | Manages stable per-tender processing                                                        |
| HTTP Request4      | HTTP Request             | Downloads tender PDF                                | Loop Over offres       | Extract from File       | Retrieves tender PDF document                                                                |
| Extract from File  | Extract From File        | Extracts text from PDF                              | HTTP Request4          | Get query               | Prepares text for keyword matching                                                          |
| Get query          | Code                     | Normalizes and matches keywords                     | Extract from File      | Query match ?           | Performs keyword matching with context                                                      |
| Query match ?      | If                       | Routes tenders by match presence                     | Get query              | Target offre / Wait1    | Decision on tender relevance                                                                 |
| Target offre       | Google Sheets (Append)   | Saves matched tenders to "Target" sheet             | Query match ? (true)   | Loop Over offres        | Stores relevant tenders                                                                      |
| Ok                 | Google Sheets (Update)   | Marks tender as processed ("Ok")                     | Get query              | Query match ?           | Tracks processing status                                                                     |
| Wait1              | Wait                     | Waits before next tender processing                   | Query match ? (false)  | Loop Over offres        | Flow control delay                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger1 node**  
   - Type: Schedule Trigger  
   - Cron Expression: `0 8 1 * *` (runs monthly at 8:00 AM)  
   - Connect output to "Get config".

2. **Create Get config node**  
   - Type: Google Sheets (Read)  
   - Document URL: Your Google Sheets URL for configuration  
   - Sheet name: "Config" (or as configured)  
   - Read all rows or specific range containing market type booleans and period  
   - Credentials: Google Sheets OAuth2  
   - Connect output to "Get Offset".

3. **Create Get Offset node**  
   - Type: Google Sheets (Read)  
   - Document URL: Same spreadsheet, sheet "Offset"  
   - Reads current offset number from a designated cell (e.g., row 2, column "Offset")  
   - Credentials: Google Sheets OAuth2  
   - Connect output to "HTTP Request".

4. **Create HTTP Request node**  
   - Type: HTTP Request  
   - URL: `https://boamp-datadila.opendatasoft.com/api/explore/v2.1/catalog/datasets/boamp/records`  
   - Query parameters:  
     - `select`: idweb,dateparution,objet,nomacheteur,datelimitereponse,type_marche,url_avis,code_departement  
     - `order_by`: dateparution desc  
     - `where`: `=dateparution >= "{{ $now.minus({days: $('Get config').item.json['Période']}).format('yyyy-MM-dd') }}"`  
     - `limit`: 100  
     - `offset`: `={{ $json.Offset }}` (from offset node)  
   - Headers: Accept: application/json; User-Agent: n8n-workflow-veille  
   - Connect output to "Tenders sorting".

5. **Create Tenders sorting node**  
   - Type: Code  
   - Script: Filter tenders by market types enabled in config (Travaux, Services, Fournitures)  
   - Throw error if no types selected  
   - Connect output to "Format Results".

6. **Create Format Results node**  
   - Type: Code  
   - Script: Format dates (dd/MM/yyyy), generate PDF URLs from idweb and dateparution  
   - Output fields: Parution, Objet, URL, Acheteur, Date limite  
   - Connect output to "Append row in sheet".

7. **Create Append row in sheet node**  
   - Type: Google Sheets (Append)  
   - Document URL: Your Google Sheets URL for data storage  
   - Sheet name: "All" (or as configured)  
   - Map columns to formatted tender fields  
   - Credentials: Google Sheets OAuth2  
   - Connect output to "Process Response".

8. **Create Process Response node**  
   - Type: Code  
   - Script: Extract total_count, results count, calculate nextOffset = currentOffset + 100, set hasMoreData boolean  
   - Connect output to "Update offset".

9. **Create Update offset node**  
   - Type: Google Sheets (Update)  
   - Document URL: Same as Get Offset  
   - Update offset cell with `nextOffset` value  
   - Credentials: Google Sheets OAuth2  
   - Connect output to "Check Continue Loop".

10. **Create Check Continue Loop node**  
    - Type: If  
    - Condition: Check if `hasMoreData` is true  
    - True output → "Wait" node  
    - False output → "Reset Offset" node

11. **Create Wait node**  
    - Type: Wait  
    - Duration: 10 seconds  
    - Connect output back to "Get Offset" to continue pagination loop

12. **Create Reset Offset node**  
    - Type: Google Sheets (Update)  
    - Reset offset back to 0 in "Offset" sheet after full data retrieval  
    - Credentials: Google Sheets OAuth2

13. **Create Schedule Trigger node**  
    - Type: Schedule Trigger  
    - Cron Expression: `0 10 1 * *` (monthly at 10:00 AM)  
    - Connect output to "Get keyword".

14. **Create Get keyword node**  
    - Type: Google Sheets (Read)  
    - Document URL: Your configuration spreadsheet  
    - Sheet name: "Config"  
    - Reads keywords column (comma separated)  
    - Credentials: Google Sheets OAuth2  
    - Connect output to "Get All".

15. **Create Get All node**  
    - Type: Google Sheets (Read)  
    - Document URL: Your data spreadsheet  
    - Sheet name: "All"  
    - Reads all tender records  
    - Credentials: Google Sheets OAuth2  
    - Connect output to "Filter".

16. **Create Filter node**  
    - Type: If  
    - Condition: `$json.Ok` not equals `"Ok"`  
    - Filters out processed tenders  
    - True output → "Loop Over offres"  
    - False output discarded

17. **Create Loop Over offres node**  
    - Type: Split In Batches  
    - Batch size: 1 (process sequentially)  
    - Connect output to "HTTP Request4".

18. **Create HTTP Request4 node**  
    - Type: HTTP Request  
    - URL: Tender PDF URL field from current item  
    - Download PDF binary  
    - Connect output to "Extract from File".

19. **Create Extract from File node**  
    - Type: Extract From File (PDF)  
    - Extract text content from downloaded PDF  
    - Connect output to "Get query".

20. **Create Get query node**  
    - Type: Code  
    - Script:  
      - Normalize both PDF text and keywords (lowercase, remove accents)  
      - Search for keyword matches  
      - Output match flag, matched terms, context snippet  
    - Connect output to "Query match ?".

21. **Create Query match ? node**  
    - Type: If  
    - Condition: `hasMatch` boolean true  
    - True output → "Target offre"  
    - False output → "Wait1"

22. **Create Target offre node**  
    - Type: Google Sheets (Append)  
    - Document URL: Your data spreadsheet  
    - Sheet name: "Target"  
    - Append matched tenders with matched keywords and current date  
    - Credentials: Google Sheets OAuth2  
    - Connect output back to "Loop Over offres".

23. **Create Ok node**  
    - Type: Google Sheets (Update)  
    - Document URL: "All" sheet  
    - Update current tender row with "Ok" in status column to mark as processed  
    - Credentials: Google Sheets OAuth2  
    - Connect output to "Query match ?" (to continue loop).

24. **Create Wait1 node**  
    - Type: Wait  
    - Short or default delay  
    - Connect output back to "Loop Over offres" for next tender

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Duplicate the Google Sheets template for configuration and data storage before using the workflow. Replace spreadsheet URLs in nodes accordingly.              | https://docs.google.com/spreadsheets/d/1wapLLWjwzo7SfG_YEsUlFaPRs1MmjxPRhRc6BlwBUAY/edit#gid=966659321#gid=966659321 |
| The workflow is designed to run monthly on the 1st day at 8:00 AM (data retrieval) and 10:00 AM (keyword filtering), but cron schedules can be adjusted as needed.| Adjust schedule triggers in n8n nodes "Schedule Trigger1" and "Schedule Trigger".                                   |
| The BOAMP API is used under its public data access policy; ensure you comply with their terms regarding request frequency and data usage.                       | https://www.boamp.fr/                                                                                               |
| PDF extraction depends on the quality and format of tender PDFs; encrypted or malformed PDFs may cause extraction failures.                                    | Consider adding error handling or fallback mechanisms in production.                                                |
| Keyword matching normalizes text by removing accents and converting to lowercase to improve match reliability.                                                  | Adjust normalization logic in "Get query" node as needed for language or domain specificity.                         |

---

**Disclaimer:**  
The provided description and analysis are based exclusively on the provided n8n workflow JSON. The workflow processes only legal and public data from the BOAMP API and Google Sheets. It respects all content and data policies. No illegal, offensive, or protected content is included.