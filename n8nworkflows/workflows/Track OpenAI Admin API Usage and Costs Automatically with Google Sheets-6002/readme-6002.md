Track OpenAI Admin API Usage and Costs Automatically with Google Sheets

https://n8nworkflows.xyz/workflows/track-openai-admin-api-usage-and-costs-automatically-with-google-sheets-6002


# Track OpenAI Admin API Usage and Costs Automatically with Google Sheets

### 1. Workflow Overview

This n8n workflow is designed to **automatically track OpenAI Admin API usage and associated costs**, consolidating this data into Google Sheets for easy monitoring and analysis. It targets organizations or developers who manage multiple OpenAI API keys and projects and need an automated solution to gather, process, and log usage and cost details regularly.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Data Retrieval**: Periodic scheduling triggers two parallel HTTP requests to OpenAI Admin API endpoints to fetch cost and token usage data.
- **1.2 API Key and Project Data Enrichment**: Extracts and processes API key and project metadata, removing duplicates and merging details to enrich usage and cost data with meaningful identifiers.
- **1.3 Data Structuring and Merging**: Combines token usage, costs, and project details into structured objects for downstream processing.
- **1.4 Data Splitting and Formatting**: Splits the merged data into discrete usage and cost subsets and formats them for Google Sheets.
- **1.5 Data Appending to Google Sheets**: Appends the structured usage and cost data into designated Google Sheets documents.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Retrieval

**Overview:**  
This block initiates the workflow on a schedule and concurrently fetches the latest OpenAI API usage and cost data via HTTP requests.

**Nodes Involved:**  
- Schedule Trigger  
- OpenAI Admin - Get cost  
- OpenAI Admin - Get token usage

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow based on a configured time interval (default scheduling parameters assumed).  
  - Configuration: No parameters specified; likely a periodic trigger (e.g., daily/hourly).  
  - Connections: Outputs to both "OpenAI Admin - Get cost" and "OpenAI Admin - get token usage".  
  - Edge Cases: No trigger means no data retrieval; scheduling misconfiguration could halt data collection.

- **OpenAI Admin - Get cost**  
  - Type: HTTP Request  
  - Role: Calls OpenAI Admin API endpoint to retrieve cost data.  
  - Configuration: Uses retry logic with up to 3 attempts and 5-second wait between tries to handle transient errors.  
  - Credentials: Requires OpenAI Admin API credentials.  
  - Connections: Outputs to "Merge token and usage".  
  - Edge Cases: HTTP errors, authentication failures, API rate limits, malformed responses.

- **OpenAI Admin - get token usage**  
  - Type: HTTP Request  
  - Role: Calls OpenAI Admin API endpoint to retrieve token usage data.  
  - Configuration: Similar retry and wait between tries as for cost node.  
  - Connections: Outputs to "Set api_key and project ids" and "Merge Usage data".  
  - Edge Cases: Same as cost node.

---

#### 2.2 API Key and Project Data Enrichment

**Overview:**  
This block processes API keys and project IDs fetched from usage data, removing duplicates, and enriching the data with names and details from OpenAI Admin API.

**Nodes Involved:**  
- Set api_key and project ids  
- Split Out api_key and project  
- OpenAI Admin - Get API Key details  
- Set api_key id and name  
- Remove Duplicates

**Node Details:**  

- **Set api_key and project ids**  
  - Type: Set  
  - Role: Initializes and sets API key and project ID variables extracted from usage data.  
  - Configuration: Executes once per workflow run.  
  - Connections: Outputs to "Split Out api_key and project".  
  - Edge Cases: Missing or malformed keys/IDs could disrupt downstream calls.

- **Split Out api_key and project**  
  - Type: Split Out  
  - Role: Splits the combined API key and project data into individual items for sequential processing.  
  - Connections: Outputs to "OpenAI Admin - Get API Key details".  
  - Edge Cases: Empty arrays lead to no downstream processing.

- **OpenAI Admin - Get API Key details**  
  - Type: HTTP Request  
  - Role: Retrieves detailed info about each API key from OpenAI Admin API.  
  - Configuration: Retry enabled with 3 attempts and 5-second wait.  
  - Connections: Outputs to "Set api_key id and name".  
  - Edge Cases: API rate limits, invalid keys, network errors.

- **Set api_key id and name**  
  - Type: Set  
  - Role: Sets the API key name and ID after fetching details.  
  - Connections: Outputs to "Remove Duplicates".  
  - Edge Cases: Missing names or IDs could cause data mismatches.

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Ensures only unique API key entries continue downstream.  
  - Connections: Outputs to "Merge Usage data".  
  - Edge Cases: Over-aggressive deduplication might drop needed records.

---

#### 2.3 Data Structuring and Merging

**Overview:**  
Combines token usage data, cost data, and project details into unified structured objects, adding project names to each usage and cost record.

**Nodes Involved:**  
- Merge Usage data  
- Add api_key name to token usage  
- Merge token and usage  
- Get project_ids  
- OpenAI Admin - Get Project details  
- Merge token, usage, projects  
- Structure usage, cost, projects  
- Add Project name in cost and usage objects

**Node Details:**  

- **Merge Usage data**  
  - Type: Merge  
  - Role: Merges the output of token usage with API key names.  
  - Connections: Outputs to "Add api_key name to token usage".  
  - Edge Cases: Data mismatch causing merge failures.

- **Add api_key name to token usage**  
  - Type: Code  
  - Role: Adds API key names to token usage objects programmatically.  
  - Connections: Outputs to "Merge token and usage".  
  - Edge Cases: Code errors, missing fields.

- **Merge token and usage**  
  - Type: Merge  
  - Role: Merges token usage data with cost data.  
  - Connections: Outputs to "Get project_ids".  
  - Edge Cases: Merge conflicts, empty inputs.

- **Get project_ids**  
  - Type: Code  
  - Role: Extracts project IDs from merged data to query for project details.  
  - Connections: Outputs to "OpenAI Admin - Get Project details".  
  - Edge Cases: Malformed data, missing project IDs.

- **OpenAI Admin - Get Project details**  
  - Type: HTTP Request  
  - Role: Fetches detailed project info from OpenAI Admin API.  
  - Connections: Outputs to "Merge token, usage, projects".  
  - Edge Cases: API failures, auth errors.

- **Merge token, usage, projects**  
  - Type: Merge  
  - Role: Combines usage, cost, and project details into one dataset.  
  - Connections: Outputs to "Structure usage, cost, projects".  
  - Edge Cases: Data alignment issues.

- **Structure usage, cost, projects**  
  - Type: Set  
  - Role: Structurally organizes combined data for further processing.  
  - Connections: Outputs to "Add Project name in cost and usage objects".  
  - Edge Cases: Data format inconsistencies.

- **Add Project name in cost and usage objects**  
  - Type: Code  
  - Role: Adds project names to both cost and usage objects for clarity.  
  - Connections: Outputs to "Split Out Usage" and "Split Out Cost".  
  - Edge Cases: Missing project names, code errors.

---

#### 2.4 Data Splitting and Formatting

**Overview:**  
Separates the enriched combined data into distinct usage and cost datasets, formats them appropriately, and prepares them for appending to Google Sheets.

**Nodes Involved:**  
- Split Out Usage  
- Split Out Cost  
- Split Out Usage Results  
- Split Out Cost results  
- Structure Usage data  
- Structure Cost data  
- Set Usage data for Gsheets  
- Set Cost data for Gsheets

**Node Details:**  

- **Split Out Usage**  
  - Type: Split Out  
  - Role: Separates usage-specific data items.  
  - Connections: Outputs to "Structure Usage data".  
  - Edge Cases: Empty usage data arrays.

- **Split Out Cost**  
  - Type: Split Out  
  - Role: Separates cost-specific data items.  
  - Connections: Outputs to "Structure Cost data".  
  - Edge Cases: Empty cost data arrays.

- **Structure Usage data**  
  - Type: Set  
  - Role: Formats usage data objects into a structure compatible with Google Sheets append operation.  
  - Connections: Outputs to "Split Out Usage Results".  
  - Edge Cases: Incorrect field mapping.

- **Structure Cost data**  
  - Type: Set  
  - Role: Formats cost data objects similarly for Google Sheets.  
  - Connections: Outputs to "Split Out Cost results".  
  - Edge Cases: Same as usage data.

- **Split Out Usage Results**  
  - Type: Split Out  
  - Role: Further splits structured usage data for batch appending.  
  - Connections: Outputs to "Set Usage data for Gsheets".  
  - Edge Cases: Empty input.

- **Split Out Cost results**  
  - Type: Split Out  
  - Role: Further splits structured cost data for batch appending.  
  - Connections: Outputs to "Set Cost data for Gsheets".  
  - Edge Cases: Empty input.

- **Set Usage data for Gsheets**  
  - Type: Set  
  - Role: Sets the final payload for appending usage data to Google Sheets.  
  - Connections: Outputs to "Append Usage to GSheets".  
  - Edge Cases: Data format or schema errors.

- **Set Cost data for Gsheets**  
  - Type: Set  
  - Role: Sets the final payload for appending cost data to Google Sheets.  
  - Connections: Outputs to "Append Cost to GSheets".  
  - Edge Cases: Data format or schema errors.

---

#### 2.5 Data Appending to Google Sheets

**Overview:**  
Appends the prepared usage and cost data entries into designated Google Sheets for persistent storage and analysis.

**Nodes Involved:**  
- Append Usage to GSheets  
- Append Cost to GSheets

**Node Details:**  

- **Append Usage to GSheets**  
  - Type: Google Sheets  
  - Role: Appends usage data rows to a target Google Sheets document.  
  - Configuration: Requires OAuth2 Google Sheets credentials with edit rights.  
  - Connections: Terminal node in workflow branch.  
  - Edge Cases: Authentication failures, API rate limits, sheet access errors.

- **Append Cost to GSheets**  
  - Type: Google Sheets  
  - Role: Appends cost data rows similarly.  
  - Configuration: Same as usage append node.  
  - Edge Cases: Same as usage append node.

---

### 3. Summary Table

| Node Name                          | Node Type            | Functional Role                          | Input Node(s)                            | Output Node(s)                           | Sticky Note                     |
|-----------------------------------|----------------------|----------------------------------------|-----------------------------------------|-----------------------------------------|--------------------------------|
| Schedule Trigger                  | Schedule Trigger     | Initiates workflow on schedule         |                                         | OpenAI Admin - Get cost, OpenAI Admin - get token usage |                                |
| OpenAI Admin - Get cost           | HTTP Request         | Fetches cost data from OpenAI Admin API| Schedule Trigger                        | Merge token and usage                    |                                |
| OpenAI Admin - get token usage    | HTTP Request         | Fetches token usage data from OpenAI Admin API | Schedule Trigger                   | Set api_key and project ids, Merge Usage data |                                |
| Set api_key and project ids       | Set                  | Initializes API key and project ID variables | OpenAI Admin - get token usage       | Split Out api_key and project            |                                |
| Split Out api_key and project     | Split Out            | Splits combined API key/project data   | Set api_key and project ids             | OpenAI Admin - Get API Key details       |                                |
| OpenAI Admin - Get API Key details| HTTP Request         | Retrieves API key details               | Split Out api_key and project           | Set api_key id and name                  |                                |
| Set api_key id and name           | Set                  | Stores API key id and name              | OpenAI Admin - Get API Key details      | Remove Duplicates                        |                                |
| Remove Duplicates                 | Remove Duplicates    | Removes duplicate API key entries       | Set api_key id and name                 | Merge Usage data                        |                                |
| Merge Usage data                  | Merge                | Merges token usage with API key names   | Remove Duplicates, OpenAI Admin - get token usage | Add api_key name to token usage     |                                |
| Add api_key name to token usage  | Code                 | Adds API key names programmatically     | Merge Usage data                        | Merge token and usage                   |                                |
| Merge token and usage            | Merge                | Merges token usage and cost data         | OpenAI Admin - Get cost, Add api_key name to token usage | Get project_ids                       |                                |
| Get project_ids                  | Code                 | Extracts project IDs for querying        | Merge token and usage                   | OpenAI Admin - Get Project details      |                                |
| OpenAI Admin - Get Project details| HTTP Request         | Retrieves project details                | Get project_ids                        | Merge token, usage, projects            |                                |
| Merge token, usage, projects     | Merge                | Combines usage, cost, and project details| OpenAI Admin - Get Project details, Merge token and usage | Structure usage, cost, projects      |                                |
| Structure usage, cost, projects  | Set                  | Organizes combined data for processing  | Merge token, usage, projects            | Add Project name in cost and usage objects |                                |
| Add Project name in cost and usage objects | Code        | Adds project names to usage and cost objects | Structure usage, cost, projects      | Split Out Usage, Split Out Cost          |                                |
| Split Out Usage                 | Split Out            | Splits usage data from combined objects | Add Project name in cost and usage objects | Structure Usage data                  |                                |
| Split Out Cost                  | Split Out            | Splits cost data from combined objects  | Add Project name in cost and usage objects | Structure Cost data                   |                                |
| Structure Usage data           | Set                  | Formats usage data for Google Sheets     | Split Out Usage                         | Split Out Usage Results                 |                                |
| Structure Cost data            | Set                  | Formats cost data for Google Sheets      | Split Out Cost                         | Split Out Cost results                  |                                |
| Split Out Usage Results        | Split Out            | Further splits usage data for batch append | Structure Usage data                  | Set Usage data for Gsheets             |                                |
| Split Out Cost results         | Split Out            | Further splits cost data for batch append  | Structure Cost data                   | Set Cost data for Gsheets               |                                |
| Set Usage data for Gsheets     | Set                  | Prepares payload for appending usage data | Split Out Usage Results               | Append Usage to GSheets                 |                                |
| Set Cost data for Gsheets      | Set                  | Prepares payload for appending cost data  | Split Out Cost results                | Append Cost to GSheets                  |                                |
| Append Usage to GSheets        | Google Sheets        | Appends usage data to Google Sheets      | Set Usage data for Gsheets             |                                         | Requires Google Sheets OAuth2 credentials |
| Append Cost to GSheets         | Google Sheets        | Appends cost data to Google Sheets       | Set Cost data for Gsheets              |                                         | Requires Google Sheets OAuth2 credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - No special parameters needed; set the interval to desired frequency (e.g., daily or hourly).

2. **Add two HTTP Request nodes:**  
   - **OpenAI Admin - Get cost**  
     - Method: GET  
     - URL: OpenAI Admin API endpoint for cost data  
     - Credentials: Set OpenAI Admin API credentials  
     - Retry: Enabled (3 tries, 5s wait)  
     - Connect input from Schedule Trigger.  
   - **OpenAI Admin - get token usage**  
     - Method: GET  
     - URL: OpenAI Admin API endpoint for token usage data  
     - Credentials: Same as above  
     - Retry: Enabled (3 tries, 5s wait)  
     - Connect input from Schedule Trigger.

3. **Add a Set node: "Set api_key and project ids"**  
   - Configure to extract and set API key and project IDs from token usage data.  
   - Execute once per run.  
   - Connect output from "OpenAI Admin - get token usage".

4. **Add a Split Out node: "Split Out api_key and project"**  
   - Configure to split combined api_key/project data into individual items.  
   - Connect output from "Set api_key and project ids".

5. **Add an HTTP Request node: "OpenAI Admin - Get API Key details"**  
   - Method: GET  
   - URL: OpenAI Admin API endpoint for API key details, parameterized per item.  
   - Credentials: OpenAI Admin API credentials  
   - Retry enabled (3 tries, 5s wait).  
   - Connect output from "Split Out api_key and project".

6. **Add a Set node: "Set api_key id and name"**  
   - Set API key ID and name from the response.  
   - Connect output from "OpenAI Admin - Get API Key details".

7. **Add a Remove Duplicates node: "Remove Duplicates"**  
   - Remove duplicate API key entries.  
   - Connect output from "Set api_key id and name".

8. **Add a Merge node: "Merge Usage data"**  
   - Merge outputs from "Remove Duplicates" and "OpenAI Admin - get token usage".  
   - Connect accordingly.

9. **Add a Code node: "Add api_key name to token usage"**  
   - Programmatically add API key names to token usage data.  
   - Connect output from "Merge Usage data".

10. **Add a Merge node: "Merge token and usage"**  
    - Merge outputs from "OpenAI Admin - Get cost" and "Add api_key name to token usage".  
    - Connect accordingly.

11. **Add a Code node: "Get project_ids"**  
    - Extract project IDs from merged data to query projects.  
    - Connect output from "Merge token and usage".

12. **Add an HTTP Request node: "OpenAI Admin - Get Project details"**  
    - Fetch detailed project info from OpenAI Admin API.  
    - Credentials: OpenAI Admin API credentials  
    - Connect output from "Get project_ids".

13. **Add a Merge node: "Merge token, usage, projects"**  
    - Merge project details with existing merged data.  
    - Connect outputs from "OpenAI Admin - Get Project details" and "Merge token and usage".

14. **Add a Set node: "Structure usage, cost, projects"**  
    - Organize combined data into structured format.  
    - Execute once.  
    - Connect output from "Merge token, usage, projects".

15. **Add a Code node: "Add Project name in cost and usage objects"**  
    - Add project names to usage and cost objects programmatically.  
    - Connect output from "Structure usage, cost, projects".

16. **Add two Split Out nodes: "Split Out Usage" and "Split Out Cost"**  
    - Split enriched data into usage and cost subsets.  
    - Connect outputs from "Add Project name in cost and usage objects".

17. **Add two Set nodes: "Structure Usage data" and "Structure Cost data"**  
    - Format data objects for Google Sheets.  
    - Connect output from "Split Out Usage" and "Split Out Cost" respectively.

18. **Add two Split Out nodes: "Split Out Usage Results" and "Split Out Cost results"**  
    - Further split structured data for batch processing.  
    - Connect outputs from "Structure Usage data" and "Structure Cost data".

19. **Add two Set nodes: "Set Usage data for Gsheets" and "Set Cost data for Gsheets"**  
    - Prepare final payloads for Google Sheets append operations.  
    - Connect outputs from "Split Out Usage Results" and "Split Out Cost results".

20. **Add two Google Sheets nodes: "Append Usage to GSheets" and "Append Cost to GSheets"**  
    - Append data to respective Google Sheets documents.  
    - Configure with Google Sheets OAuth2 credentials and specify target spreadsheet and sheet name for usage and cost data.  
    - Connect outputs from "Set Usage data for Gsheets" and "Set Cost data for Gsheets".

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                        |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Requires valid OpenAI Admin API credentials with permissions to access usage, cost, API key, and project details APIs.  | OpenAI API Documentation                              |
| Google Sheets nodes require OAuth2 credentials with edit rights to target spreadsheets.                                 | n8n Google Sheets Integration Documentation           |
| Retry logic with multiple attempts and delays is implemented for HTTP requests to handle transient network or API errors.| Best practice for stable API integrations              |
| Data deduplication is critical to avoid redundant API key processing and incorrect data aggregation.                    | Workflow data integrity                                 |
| Structured data formatting before Google Sheets append ensures compatibility with sheet columns and prevents errors.    | Google Sheets API constraints                           |

---

**Disclaimer:** This document is generated solely from the analyzed n8n workflow and contains no illegal, offensive, or protected content. All data processed is legal and public.