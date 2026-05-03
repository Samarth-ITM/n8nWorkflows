Regional Prospecting for registered Companies in Germany

https://n8nworkflows.xyz/workflows/regional-prospecting-for-registered-companies-in-germany-11750


# Regional Prospecting for registered Companies in Germany

### 1. Workflow Overview

This workflow, titled **"Regional Prospecting for registered Companies in Germany"**, is designed to find, normalize, score, and qualify registered companies in Germany based on regional and industry-specific search criteria. It leverages the Implisense Search API (via RapidAPI) to query the official commercial register data, processes the results to assign relevance scores, and categorizes leads into quality tiers for further use.

The workflow is organized into the following logical blocks:

- **1.1 Initialization:** Setup of API authorization and input parameters.
- **1.2 Input Validation:** Ensures search parameters meet required constraints.
- **1.3 External API Search:** Queries the Implisense Search API for companies.
- **1.4 Search Result Validation:** Checks if search returned any results.
- **1.5 Normalization & Scoring:** Processes raw company data, normalizes it, and calculates relevance scores.
- **1.6 Sorting & Classification:** Sorts companies by relevance and classifies them into high or medium quality leads.
- **1.7 Payload Preparation:** Structures lead data for output.
- **1.8 Merging & Reporting:** Combines results and generates summary information for the run.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization

- **Overview:** Sets up the API key credential and prepares the input parameters with default values, including a unique workflow run ID and timestamp.
- **Nodes Involved:** `Manual Trigger`, `Authorization`, `Prepare Search Input`

**Node Details:**

- **Manual Trigger**
  - Type: Trigger node to manually start the workflow.
  - Configuration: Default; no parameters.
  - Input: None (start node).
  - Output: Triggers `Authorization`.
  - Failures: None expected.

- **Authorization**
  - Type: Set node.
  - Configuration: Stores RapidAPI `x-rapidapi-key` credential key (placeholder value "XXXX").
  - Input: From `Manual Trigger`.
  - Output: Triggers `Prepare Search Input`.
  - Notes: API key must be obtained from RapidAPI Implisense API page.
  - Failure modes: Missing or invalid API key causes downstream HTTP request failures.

- **Prepare Search Input**
  - Type: Set node.
  - Configuration: Assigns workflow-run metadata and default search parameters if not provided:
    - `workflow_run_id`: Unique execution ID.
    - `started_at`: ISO timestamp.
    - `query`: Default `"hubspot AND chat"`.
    - `regionCode`: Default `"de-10"`.
    - `industryCode`: Default `"J62"`.
    - `pageSize`: Default `100`.
  - Input: From `Authorization`.
  - Output: Triggers `Validate Input`.
  - Edge cases: Default values ensure workflow runs even if external input is missing.

---

#### 1.2 Input Validation

- **Overview:** Validates that required inputs (`query`, `regionCode`, and `pageSize`) are present and within allowed constraints.
- **Nodes Involved:** `Validate Input`

**Node Details:**

- **Validate Input**
  - Type: Code node (JavaScript).
  - Configuration: Checks:
    - `query` and `regionCode` are not empty strings.
    - `pageSize` is between 1 and 1000.
  - Throws errors on invalid input to stop workflow execution.
  - Logs validation success and input parameters.
  - Input: From `Prepare Search Input`.
  - Output: Triggers `Implisense Search`.
  - Failure modes: Throws error to stop workflow if validation fails.

---

#### 1.3 External API Search

- **Overview:** Sends a POST request to Implisense Search API with search parameters to retrieve companies.
- **Nodes Involved:** `Implisense Search`

**Node Details:**

- **Implisense Search**
  - Type: HTTP Request node.
  - Configuration:
    - URL: `https://german-company-data.p.rapidapi.com/search`
    - Method: POST
    - Body: JSON with `query`, `from: 0`, `size: pageSize`, `explain: false`.
    - Headers: Includes `x-rapidapi-key` from `Authorization` node, and standard Accept header.
    - Query parameters redundantly also specify pagination for robustness.
  - Input: From `Validate Input`.
  - Output: Triggers `Search Success?`
  - Failure modes:
    - HTTP errors (e.g., 401 Unauthorized if API key invalid).
    - Timeout or network errors.
    - Unexpected response format.

---

#### 1.4 Search Result Validation

- **Overview:** Determines if the API returned any company results to continue processing or branch to empty handling.
- **Nodes Involved:** `Search Success?`

**Node Details:**

- **Search Success?**
  - Type: If node.
  - Configuration: Checks if `companies` array exists and is non-empty in the response JSON.
  - Input: From `Implisense Search`.
  - Output:
    - True branch: triggers `Normalize & Score Results`.
    - False branch: empty (no further action).
  - Failure modes:
    - Expression failure if response lacks expected structure.

---

#### 1.5 Normalization & Scoring

- **Overview:** Extracts companies from response, normalizes data, and computes a relevance score based on activity, presence of website, and address completeness.
- **Nodes Involved:** `Normalize & Score Results`

**Node Details:**

- **Normalize & Score Results**
  - Type: Code node.
  - Configuration:
    - Extracts company list robustly from response variations.
    - Calculates `relevanceScore` starting from API score.
    - Adds 10 points if company is active.
    - Adds 5 points if website URL exists.
    - Adds 3 points if full address (street, zip, city) is present.
    - Includes metadata: target region, industry code, workflow run ID, processing timestamp.
    - Logs summary info.
    - Returns array of normalized company JSON objects.
  - Input: From `Search Success?` (true branch).
  - Output: Triggers `Sort by Relevance`.
  - Edge cases:
    - Empty results handled by returning a warning JSON.
    - Missing fields default to empty or false.
  - Failure modes:
    - Expression errors if response format changes.

---

#### 1.6 Sorting & Classification

- **Overview:** Sorts companies by descending relevance score and classifies leads into high or medium quality based on a threshold.
- **Nodes Involved:** `Sort by Relevance`, `High Quality Leads?`, `Prepare High Quality Payload`, `Prepare Medium Quality Payload`

**Node Details:**

- **Sort by Relevance**
  - Type: Code node.
  - Configuration:
    - Sorts all incoming items by `relevanceScore` descending.
    - Logs top company name and score.
  - Input: From `Normalize & Score Results`.
  - Output: Triggers `High Quality Leads?`.

- **High Quality Leads?**
  - Type: If node.
  - Configuration:
    - Condition: `relevanceScore >= 15` for high quality.
  - Input: From `Sort by Relevance`.
  - Output:
    - True branch: triggers `Prepare High Quality Payload`.
    - False branch: triggers `Prepare Medium Quality Payload`.
  - Failure modes:
    - Expression errors if relevanceScore missing.

- **Prepare High Quality Payload**
  - Type: Set node.
  - Configuration:
    - Sets lead properties including company name, website, address, region, industry, Implisense ID, relevance score.
    - Sets `leadQuality` = `"high"`.
    - Marks `isActive` and source system.
    - Adds enrichment timestamp.
  - Input: From `High Quality Leads?` true branch.
  - Output: Triggers `Merge & Log Results`.

- **Prepare Medium Quality Payload**
  - Type: Set node.
  - Configuration:
    - Same as high quality but sets `leadQuality` = `"medium"`.
  - Input: From `High Quality Leads?` false branch.
  - Output: Triggers `Merge & Log Results`.
  - Note: `alwaysOutputData` is true to ensure output even if empty.

---

#### 1.7 Merging & Reporting

- **Overview:** Combines all qualified leads, logs summary counts, and generates a detailed summary report of the workflow run.
- **Nodes Involved:** `Merge & Log Results`, `Generate Summary Report`

**Node Details:**

- **Merge & Log Results**
  - Type: Code node.
  - Configuration:
    - Receives all leads (high and medium quality).
    - Logs total qualified leads and counts per quality level.
    - Returns all leads unchanged.
  - Input: From both `Prepare High Quality Payload` and `Prepare Medium Quality Payload`.
  - Output: Triggers `Generate Summary Report`.

- **Generate Summary Report**
  - Type: Set node.
  - Configuration:
    - Constructs a summary object containing:
      - Workflow run ID, start and completion timestamps.
      - Input parameters.
      - Result counts: total found, high quality leads, total qualified.
    - Prepares an array of all lead JSON objects under `leads`.
  - Input: From `Merge & Log Results`.
  - Output: Workflow end.
  - Failure modes:
    - Expression errors if prior nodes output missing or malformed.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                           | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                                                       |
|---------------------------|---------------------|-----------------------------------------|--------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger            | manualTrigger       | Workflow start trigger                   | None                     | Authorization               | ## Init                                                                                                                          |
| Authorization             | set                 | Stores RapidAPI API key                  | Manual Trigger           | Prepare Search Input        | Get API key here: https://rapidapi.com/Implisense/api/german-company-data/playground                                            |
| Prepare Search Input      | set                 | Prepares search parameters and metadata | Authorization            | Validate Input              | ## Init                                                                                                                          |
| Validate Input            | code                | Validates input parameters               | Prepare Search Input     | Implisense Search           | ## Search                                                                                                                        |
| Implisense Search         | httpRequest         | Queries Implisense Search API            | Validate Input           | Search Success?             | ## Search                                                                                                                        |
| Search Success?           | if                  | Checks if search returned companies      | Implisense Search        | Normalize & Score Results   | ## Vetting                                                                                                                       |
| Normalize & Score Results | code                | Normalizes and scores company results    | Search Success?          | Sort by Relevance           | ## Vetting                                                                                                                       |
| Sort by Relevance         | code                | Sorts companies by relevance score       | Normalize & Score Results| High Quality Leads?         | ## Vetting                                                                                                                       |
| High Quality Leads?       | if                  | Classifies leads by relevance threshold  | Sort by Relevance        | Prepare High Quality Payload (true branch), Prepare Medium Quality Payload (false branch) | ## Vetting                                                                                                                       |
| Prepare High Quality Payload | set               | Prepares payload for high quality leads  | High Quality Leads?      | Merge & Log Results         | ## Output                                                                                                                        |
| Prepare Medium Quality Payload | set             | Prepares payload for medium quality leads| High Quality Leads?      | Merge & Log Results         | ## Output                                                                                                                        |
| Merge & Log Results       | code                | Merges leads and logs summary            | Prepare High Quality Payload, Prepare Medium Quality Payload | Generate Summary Report | ## Output                                                                                                                        |
| Generate Summary Report   | set                 | Creates summary report and aggregates leads | Merge & Log Results      | None (workflow end)         | ## Output                                                                                                                        |
| 🏗️ Architecture Notes     | stickyNote          | Documentation & instructions              | None                     | None                       | # Regional Prospecting for registered Companies in Germany ... See detailed note content in workflow                          |
| Sticky Note1              | stickyNote          | Visual block label "Init"                  | None                     | None                       | ## Init                                                                                                                          |
| Sticky Note2              | stickyNote          | Visual block label "Search"                | None                     | None                       | ## Search                                                                                                                        |
| Sticky Note3              | stickyNote          | Visual block label "Vetting"               | None                     | None                       | ## Vetting                                                                                                                       |
| Sticky Note               | stickyNote          | Visual block label "Output"                 | None                     | None                       | ## Output                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Manual Trigger node**
   - Name: `Manual Trigger`
   - Default configuration; triggers workflow manually.

3. **Add Set node for Authorization**
   - Name: `Authorization`
   - Add a string field `x-rapidapi-key` with placeholder value `"XXXX"`.
   - This stores the RapidAPI key.
   - Connect `Manual Trigger` output to `Authorization` input.
   - Note: Replace `"XXXX"` with your actual RapidAPI key.

4. **Add Set node for Prepare Search Input**
   - Name: `Prepare Search Input`
   - Add assignments:
     - `workflow_run_id`: Expression `{{$execution.id}}`
     - `started_at`: Expression `{{$now.toISO()}}`
     - `query`: Expression `{{$json.query || 'hubspot AND chat'}}`
     - `regionCode`: Expression `{{$json.regionCode || 'de-10'}}`
     - `industryCode`: Expression `{{$json.industryCode || 'J62'}}`
     - `pageSize`: Expression `{{$json.pageSize || 100}}`
   - Connect `Authorization` output to this node's input.

5. **Add Code node for Validate Input**
   - Name: `Validate Input`
   - Paste the following JavaScript code:

     ```js
     const input = $input.first().json;

     if (!input.query || input.query.trim() === '') {
       throw new Error('Search query is required');
     }

     if (!input.regionCode || input.regionCode.trim() === '') {
       throw new Error('Region code is required');
     }

     if (input.pageSize < 1 || input.pageSize > 1000) {
       throw new Error('Page size must be between 1 and 1000');
     }

     console.log('✅ Input validation passed');
     console.log('Query: ' + input.query);
     console.log('Region: ' + input.regionCode);
     console.log('Industry: ' + (input.industryCode || 'None'));
     console.log('Page size: ' + input.pageSize);

     return [{ json: input }];
     ```
   - Connect `Prepare Search Input` output to this node's input.

6. **Add HTTP Request node for Implisense Search**
   - Name: `Implisense Search`
   - HTTP Method: POST
   - URL: `https://german-company-data.p.rapidapi.com/search`
   - Body Content Type: JSON
   - Body Parameters (JSON):
     ```json
     {
       "query": {{$json.query}},
       "from": 0,
       "size": {{$json.pageSize}},
       "explain": false
     }
     ```
   - Headers:
     - `Accept`: `application/json`
     - `x-rapidapi-host`: `german-company-data.p.rapidapi.com`
     - `x-rapidapi-key`: Expression: `{{$node["Authorization"].json["x-rapidapi-key"]}}`
   - Query Parameters (optional for redundancy):
     - `explain`: `true`
     - `from`: `0`
     - `size`: `3`
   - Connect `Validate Input` output to this node's input.

7. **Add If node for Search Success?**
   - Name: `Search Success?`
   - Condition:
     - Check if `{{$json.companies}}` exists (array and non-empty).
   - Connect `Implisense Search` output to this node's input.

8. **Add Code node for Normalize & Score Results**
   - Name: `Normalize & Score Results`
   - Paste the JavaScript code which processes response, normalizes companies, calculates relevance scores, and returns an array of normalized company JSON objects.
   - Connect `Search Success?` True output to this node's input.

9. **Add Code node for Sort by Relevance**
   - Name: `Sort by Relevance`
   - Paste JavaScript code to sort companies descending by `relevanceScore`.
   - Connect `Normalize & Score Results` output to this node's input.

10. **Add If node for High Quality Leads?**
    - Name: `High Quality Leads?`
    - Condition: `relevanceScore >= 15`
    - Connect `Sort by Relevance` output to this node's input.

11. **Add Set node for Prepare High Quality Payload**
    - Name: `Prepare High Quality Payload`
    - Assign fields:
      - `companyName`: `{{$json.name}}`
      - `website`: `{{$json.url}}`
      - `fullAddress`: Expression concatenating street, zip, city if available.
      - `zipRegion`: `{{$json.targetRegion}}`
      - `industryCode`: `{{$json.industryCode}}`
      - `implisenseId`: `{{$json.implisenseId}}`
      - `relevanceScore`: `{{$json.relevanceScore}}`
      - `leadQuality`: `"high"`
      - `isActive`: `{{$json.active}}`
      - `sourceSystem`: `"implisense_geotargeting"`
      - `enrichedAt`: `{{$now.toISO()}}`
    - Connect True output of `High Quality Leads?` to this node.

12. **Add Set node for Prepare Medium Quality Payload**
    - Name: `Prepare Medium Quality Payload`
    - Same assignments as above but set `leadQuality` to `"medium"`.
    - Set `alwaysOutputData` to true.
    - Connect False output of `High Quality Leads?` to this node.

13. **Add Code node for Merge & Log Results**
    - Name: `Merge & Log Results`
    - JavaScript code to merge leads from both quality branches, log counts, and return all leads.
    - Connect outputs of both `Prepare High Quality Payload` and `Prepare Medium Quality Payload` to this node.

14. **Add Set node for Generate Summary Report**
    - Name: `Generate Summary Report`
    - Assignments:
      - `summary`: Object with workflow run metadata, input parameters, and results counts.
      - `leads`: Array of all leads from `Merge & Log Results`.
    - Connect `Merge & Log Results` output to this node.

15. **Add Sticky Notes** (optional for documentation)
    - Add sticky notes to label major logical blocks: Init, Search, Vetting, Output.
    - Add architecture note with workflow description and instructions.

16. **Configure Credentials**
    - Ensure the RapidAPI API key is replaced in the `Authorization` node.
    - Test with manual trigger and input parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Get API key here: https://rapidapi.com/Implisense/api/german-company-data/playground                                                                            | RapidAPI Implisense API key setup                                                               |
| This workflow uses Implisense Search API which covers ~2.5 million officially registered companies in Germany                                                    | Implisense Search API documentation                                                             |
| Quality scoring boosts active companies with websites and full addresses, threshold of 15 for high quality leads                                                | Quality scoring methodology described in code                                                   |
| Suggested post-processing: connect "Merge & Log Results" node to CRM, database, or HTTP request for further lead management                                    | Workflow output extension ideas                                                                 |
| Default search parameters: query `"hubspot AND chat"`, region `"de-10"`, industry `"J62"`, pageSize `100`                                                       | Default values in `Prepare Search Input` node                                                   |
| Workflow provides detailed console logging for debugging and operational monitoring                                                                             | Logs in code nodes                                                                              |
| Beware of API rate limits and ensure valid API key to avoid HTTP 401 errors                                                                                      | Error handling notes                                                                            |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow, strictly complying with content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.