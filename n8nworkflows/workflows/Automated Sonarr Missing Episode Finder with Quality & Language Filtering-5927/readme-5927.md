Automated Sonarr Missing Episode Finder with Quality & Language Filtering

https://n8nworkflows.xyz/workflows/automated-sonarr-missing-episode-finder-with-quality---language-filtering-5927


# Automated Sonarr Missing Episode Finder with Quality & Language Filtering

---

### 1. Workflow Overview

This workflow automates the process of identifying missing TV show episodes in Sonarr and queues them for download based on specific quality and language filters. It targets users who manage media libraries with Sonarr and want to ensure missing episodes are automatically found and downloaded only if they match desired criteria like video quality (e.g., WEBDL-1080p) and language (e.g., Portuguese (Brazil)).

**Logical blocks are:**

- **1.1 Scheduled Trigger & Initialization:** Starts the workflow periodically and sets static configuration parameters such as Sonarr URL, API key, desired quality, and languages.
- **1.2 Fetch Missing Episodes:** Retrieves the list of missing episodes from Sonarr’s API.
- **1.3 Data Preparation & Filtering:** Processes the missing episodes list to extract unique series-season combinations and splits the data for batch processing.
- **1.4 Episode Search & Validation:** For each batch, searches for all episodes in the season and filters releases by quality and language to match user preferences.
- **1.5 Download Queue Override:** For validated episodes, sends a request to Sonarr to override and add the release to the download queue.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Initialization

- **Overview:**  
This block triggers the workflow every 12 hours and initializes key parameters such as Sonarr URL, API key, desired quality, and language.

- **Nodes Involved:**  
  - Schedule Trigger  
  - info

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Triggers every 12 hours  
    - Inputs: None (start node)  
    - Outputs: info  
    - Failure Modes: Misconfiguration could cause no triggering or too frequent runs.

  - **info**  
    - Type: Set  
    - Configuration: Sets static variables:  
      - urlSonar: Sonarr base URL (http://192.168.31.204:8989)  
      - apikey: Sonarr API key  
      - quality: Desired quality filter ("WEBDL-1080p")  
      - languages: Desired language filter ("Portuguese (Brazil)")  
    - Inputs: Schedule Trigger  
    - Outputs: Check for Missing Episodes  
    - Failure Modes: Hardcoded API key and URL; if Sonarr URL or key changes, this node must be updated.

---

#### 2.2 Fetch Missing Episodes

- **Overview:**  
Fetches the current list of missing episodes from Sonarr via its API.

- **Nodes Involved:**  
  - Check for Missing Episodes  
  - Edit Fields  
  - Split Out

- **Node Details:**

  - **Check for Missing Episodes**  
    - Type: HTTP Request  
    - Configuration: GET request to `/api/v3/wanted/missing` endpoint with query parameters for paging and filtering, including the API key.  
    - Inputs: info  
    - Outputs: Edit Fields  
    - Failure Modes: API authentication failure, network timeout, unexpected API response structure.

  - **Edit Fields**  
    - Type: Set  
    - Configuration: Wraps the returned JSON's `records` property into a new array field named `records` for further processing.  
    - Inputs: Check for Missing Episodes  
    - Outputs: Split Out  
    - Failure Modes: If `records` is missing or empty in the response, the next nodes receive an empty array.

  - **Split Out**  
    - Type: Split Out  
    - Configuration: Splits the `records` array into individual items for processing.  
    - Inputs: Edit Fields  
    - Outputs: Filter Series  
    - Failure Modes: Empty input array leads to no downstream processing.

---

#### 2.3 Data Preparation & Filtering

- **Overview:**  
Filters the missing episodes data to reduce duplicates by series and season, ensuring processing is done per unique season per series.

- **Nodes Involved:**  
  - Filter Series

- **Node Details:**

  - **Filter Series**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Uses a Set to track unique `seriesId`-`seasonNumber` combinations.  
      - Filters only the first occurrence per series-season.  
      - Adds a new field `myNewField` with value 1 for filtered items.  
      - Sorts filtered items by `seriesId` and then `seasonNumber`.  
    - Inputs: Split Out  
    - Outputs: Loop Over Items  
    - Failure Modes: Code errors if input format changes; empty input leads to empty output.

---

#### 2.4 Batch Processing & Episode Search

- **Overview:**  
Splits unique series-season items into batches and for each batch performs two parallel operations: adds matching episodes to download queue and fetches all releases for the specific season for validation.

- **Nodes Involved:**  
  - Loop Over Items  
  - Override and add to download queue  
  - Interactive search for all episodes in this season

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Processes items one by one (default batch size not specified, default is 1).  
    - Inputs: Filter Series  
    - Outputs: Two outputs connected to "Override and add to download queue" and "Interactive search for all episodes in this season".  
    - Failure Modes: Large datasets may cause long execution times.

  - **Override and add to download queue**  
    - Type: HTTP Request  
    - Configuration:  
      - POST to `/api/v3/release` endpoint to add a release to the download queue with override.  
      - Payload constructed dynamically from current batch item’s JSON properties, including indexerId, guid, seriesId, episodeIds, quality details, language, and override flag.  
      - Query parameter includes apikey.  
      - Retries up to 3 times on failure.  
      - On error, does not stop workflow (continueErrorOutput).  
    - Inputs: Loop Over Items (first output)  
    - Outputs: None (terminal)  
    - Failure Modes: API failures, invalid data in JSON, network issues, auth failures.

  - **Interactive search for all episodes in this season**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to `/api/v3/release` with query parameters `seriesId` and `seasonNumber` from the current batch item, plus apikey.  
      - Executes once per batch item.  
    - Inputs: Loop Over Items (second output)  
    - Outputs: Validate Quality and Language Match  
    - Failure Modes: API failures, missing parameters, response delays.

---

#### 2.5 Validation and Filtering

- **Overview:**  
Validates each release against user-defined quality and language filters to decide if it should be queued for download.

- **Nodes Involved:**  
  - Validate Quality and Language Match

- **Node Details:**

  - **Validate Quality and Language Match**  
    - Type: If  
    - Configuration:  
      - Checks if the release’s `quality.quality.name` equals the desired quality (e.g., "WEBDL-1080p")  
      - AND the first language in the `languages` array matches the desired language (e.g., "Portuguese (Brazil)").  
    - Inputs: Interactive search for all episodes in this season  
    - Outputs: True branch connected back to Loop Over Items (feeding Override and add to download queue) to queue the download; False branch ends.  
    - Failure Modes: Missing or differently structured `quality` or `languages` fields, case sensitivity causing mismatches.

---

### 3. Summary Table

| Node Name                               | Node Type              | Functional Role                                   | Input Node(s)                    | Output Node(s)                                | Sticky Note                       |
|----------------------------------------|------------------------|-------------------------------------------------|---------------------------------|----------------------------------------------|----------------------------------|
| Schedule Trigger                       | Schedule Trigger       | Periodic trigger every 12 hours                  | None                            | info                                         |                                  |
| info                                  | Set                    | Sets static Sonarr config and filters            | Schedule Trigger                | Check for Missing Episodes                    |                                  |
| Check for Missing Episodes             | HTTP Request           | Fetches missing episodes from Sonarr             | info                           | Edit Fields                                   |                                  |
| Edit Fields                          | Set                    | Wraps 'records' array for splitting               | Check for Missing Episodes      | Split Out                                     |                                  |
| Split Out                            | Split Out               | Splits records array into individual items        | Edit Fields                    | Filter Series                                 |                                  |
| Filter Series                       | Code                   | Filters unique series-season combos, sorts       | Split Out                      | Loop Over Items                               |                                  |
| Loop Over Items                     | Split In Batches        | Processes episodes in batches                      | Filter Series                  | Override and add to download queue, Interactive search for all episodes in this season |                                  |
| Override and add to download queue | HTTP Request            | Adds release to Sonarr download queue with override | Loop Over Items (1st output)   | None                                         |                                  |
| Interactive search for all episodes in this season | HTTP Request | Searches all releases for the season              | Loop Over Items (2nd output)   | Validate Quality and Language Match           |                                  |
| Validate Quality and Language Match | If                      | Validates quality and language against filters    | Interactive search for all episodes in this season | Loop Over Items (true branch)                   |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Set to run every 12 hours.

2. **Create Set node named "info":**  
   - Connect Schedule Trigger → info  
   - Set variables:  
     - `urlSonar` = `http://192.168.31.204:8989` (replace with your Sonarr URL)  
     - `apikey` = your Sonarr API key  
     - `quality` = e.g., `WEBDL-1080p`  
     - `languages` = e.g., `Portuguese (Brazil)`

3. **Create HTTP Request node "Check for Missing Episodes":**  
   - Connect info → Check for Missing Episodes  
   - Method: GET  
   - URL: `={{ $json.urlSonar }}/api/v3/wanted/missing`  
   - Query parameters:  
     - `page=1`  
     - `pageSize=100`  
     - `sortDirection=descending`  
     - `includeSeries=false`  
     - `includeImages=false`  
     - `monitored=true`  
     - `apikey={{ $json.apikey }}`  
   - Headers: `accept: application/json`

4. **Create Set node "Edit Fields":**  
   - Connect Check for Missing Episodes → Edit Fields  
   - Set field `records` to `={{ $json.records }}` (copies the records array)

5. **Create Split Out node "Split Out":**  
   - Connect Edit Fields → Split Out  
   - Field to split out: `records`

6. **Create Code node "Filter Series":**  
   - Connect Split Out → Filter Series  
   - Paste JavaScript code to filter unique series-season combinations and sort accordingly.

7. **Create Split In Batches node "Loop Over Items":**  
   - Connect Filter Series → Loop Over Items  
   - Use default batch size 1 (or set as needed)

8. **Create HTTP Request node "Override and add to download queue":**  
   - Connect Loop Over Items (first output) → Override and add to download queue  
   - Method: POST  
   - URL: `={{ $json.urlSonar }}/api/v3/release`  
   - Query parameters: `apikey={{ $json.apikey }}`  
   - Body (JSON): Construct dynamically with keys from the current item’s JSON (indexerId, guid, seriesId, episodeIds, quality, languages, etc.)  
   - Enable "Retry on Fail" with max 3 tries  
   - On error: continue (do not fail the workflow)

9. **Create HTTP Request node "Interactive search for all episodes in this season":**  
   - Connect Loop Over Items (second output) → Interactive search for all episodes in this season  
   - Method: GET  
   - URL: `={{ $json.urlSonar }}/api/v3/release`  
   - Query parameters:  
     - `seriesId={{ $json.seriesId }}`  
     - `seasonNumber={{ $json.seasonNumber }}`  
     - `apikey={{ $json.apikey }}`

10. **Create If node "Validate Quality and Language Match":**  
    - Connect Interactive search for all episodes in this season → Validate Quality and Language Match  
    - Condition:  
      - `quality.quality.name` equals `={{ $json.quality }}` (from info node)  
      - AND `languages[0].name` equals `={{ $json.languages }}` (from info node)  
    - True output: Connect back to Loop Over Items (to feed Override and add to download queue)  
    - False output: leave unconnected (ends branch)

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Hardcoded Sonarr URL and API key require secure handling and updates if infrastructure changes.            | Configuration security best practices            |
| The workflow uses Sonarr API v3 endpoints; ensure Sonarr server is compatible with this API version.        | https://sonarr.tv/docs/api/v3/                   |
| Retry logic is implemented on download queue override to handle transient network or API errors.           | Reliability enhancement                           |
| Language and quality filters are case-sensitive string comparisons; adjust if Sonarr uses different formatting. | Customization tip                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---