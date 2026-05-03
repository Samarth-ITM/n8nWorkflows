Automated SEO Keyword & SERP Analysis with DataForSEO for High-Converting Content

https://n8nworkflows.xyz/workflows/automated-seo-keyword---serp-analysis-with-dataforseo-for-high-converting-content-4277


# Automated SEO Keyword & SERP Analysis with DataForSEO for High-Converting Content

### 1. Workflow Overview

This workflow automates comprehensive SEO keyword research and SERP (Search Engine Results Page) analysis using the DataForSEO API. Targeted at SEO specialists, content marketers, and digital strategists, it gathers keyword suggestions, related keywords, autocomplete phrases, subtopics, and analyzes SERPs and People Also Ask (PAA) data to generate actionable insights for creating high-converting content.

The workflow is logically divided into these key blocks:

- **1.1 Trigger and Setup**: Manual initiation and preparation of Google Drive folders and Google Sheets templates for data storage.
- **1.2 Keyword Data Retrieval**: Sequential HTTP requests to DataForSEO endpoints to fetch keyword suggestions, related keywords, keyword ideas, autocomplete suggestions, and subtopics.
- **1.3 Data Processing and Splitting**: Splitting aggregated API responses into individual items for granular processing.
- **1.4 Data Formatting**: Setting and normalizing fields for each keyword category and SERP/PAA data.
- **1.5 Data Storage**: Adding processed data into respective Google Sheets for organized storage and further analysis.
- **1.6 SERP and PAA Filtering and Storage**: Filtering SERP and PAA results, formatting their fields, and saving them to dedicated sheets.
- **1.7 Master Data Consolidation**: Consolidating all keyword variations into master sheets for unified overview.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Setup

- **Overview:** Starts the workflow manually and prepares the environment by creating necessary Google Drive folders and copying template files.  
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Google Sheets KW Research Template  
  - Google Drive Create KW Folder  
  - Google Drive Copy KW Template  
  - Set Main Fields  

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Inputs: None  
    - Outputs: Triggers Google Sheets template node.  
    - Failures: None typical; manual trigger depends on user action.  

  - **Google Sheets KW Research Template**  
    - Type: Google Sheets  
    - Role: Retrieves or initializes the keyword research template spreadsheet.  
    - Configuration: Uses credentials authorized for Google Sheets API; reads base template.  
    - Inputs: Trigger node output  
    - Outputs: Triggers Google Drive folder creation.  
    - Failures: API auth errors, sheet not found.  

  - **Google Drive Create KW Folder**  
    - Type: Google Drive  
    - Role: Creates a dedicated folder on Google Drive for keyword files.  
    - Configuration: Uses Drive API credentials; specifies folder name and parent folder if any.  
    - Inputs: Output of the Sheets node  
    - Outputs: Triggers folder copy operation.  
    - Failures: Permission errors, quota limits.  

  - **Google Drive Copy KW Template**  
    - Type: Google Drive  
    - Role: Copies the keyword template file into the newly created folder.  
    - Configuration: Drive API credentials; source file ID and destination folder ID parameters.  
    - Inputs: Output of folder creation node  
    - Outputs: Triggers Set Main Fields node.  
    - Failures: File not found, permission errors.  

  - **Set Main Fields**  
    - Type: Set  
    - Role: Initializes main parameters and variables used downstream, such as keywords or API parameters for requests.  
    - Configuration: Sets key-value pairs for API calls and data handling.  
    - Inputs: Output of copy operation  
    - Outputs: Triggers all HTTP request nodes for keyword and SERP data.  
    - Failures: Expression errors if variables are missing.  

#### 2.2 Keyword Data Retrieval

- **Overview:** Makes HTTP requests to DataForSEO API endpoints to fetch various keyword-related data: related keywords, keyword suggestions, keyword ideas, autocomplete phrases, and subtopics.  
- **Nodes Involved:**  
  - HTTP Related Keywords  
  - HTTP Keyword Suggestions  
  - HTTP Keyword Ideas  
  - HTTP Autocomplete  
  - HTTP Subtopics  
  - HTTP SERPs  

- **Node Details:**  

  - **HTTP Related Keywords**  
    - Type: HTTP Request  
    - Role: Fetch related keywords from DataForSEO API.  
    - Configuration: REST API call with proper authentication and query parameters for related keywords endpoint.  
    - Inputs: Output from Set Main Fields  
    - Outputs: Triggers "Split Out Related KWs" node.  
    - Failures: API auth errors, rate limits, malformed request.  

  - **HTTP Keyword Suggestions**  
    - Type: HTTP Request  
    - Role: Fetch keyword suggestions.  
    - Config: API call with appropriate parameters for suggestions endpoint.  
    - Inputs: Set Main Fields output  
    - Outputs: Triggers "Split Out KW Suggestions" node.  
    - Failures: Same as above.  

  - **HTTP Keyword Ideas**  
    - Type: HTTP Request  
    - Role: Fetch keyword ideas data.  
    - Inputs: Set Main Fields output  
    - Outputs: Triggers "Split Out KW Ideas" node.  
    - Failures: Same as above.  

  - **HTTP Autocomplete**  
    - Type: HTTP Request  
    - Role: Fetch autocomplete keyword phrases.  
    - Inputs: Set Main Fields output  
    - Outputs: Triggers "Split Out Autocomplete" node.  
    - Failures: Same as above.  

  - **HTTP Subtopics**  
    - Type: HTTP Request  
    - Role: Fetch keyword subtopics.  
    - Inputs: Set Main Fields output  
    - Outputs: Triggers "Split Out Subtopics" node.  
    - Failures: Same as above.  

  - **HTTP SERPs**  
    - Type: HTTP Request  
    - Role: Fetch SERP and PAA (People Also Ask) data.  
    - Inputs: Set Main Fields output  
    - Outputs: Triggers "Split Out SERPs and PAA" node.  
    - Failures: Same as above.  

#### 2.3 Data Processing and Splitting

- **Overview:** Splits the array responses from the API calls into individual keyword or SERP items for detailed processing.  
- **Nodes Involved:**  
  - Split Out Related KWs  
  - Split Out KW Suggestions  
  - Split Out KW Ideas  
  - Split Out Autocomplete  
  - Split Out Subtopics  
  - Split Out SERPs and PAA  
  - Split Out PAA  

- **Node Details:**  

  - All Split Out nodes are of type `splitOut`.  
  - Role: Takes an array of items (keywords, subtopics, SERPs, etc.) and outputs each item as a separate execution item.  
  - Inputs: Corresponding HTTP request node outputs.  
  - Outputs: Connect to respective Set Fields nodes for field setting.  
  - Failures: Empty arrays lead to no downstream data; ensure API returns data.  

#### 2.4 Data Formatting

- **Overview:** Sets and normalizes fields for each individual keyword or SERP/PAA item for consistent data structure.  
- **Nodes Involved:**  
  - Set Related KW Fields  
  - Set KW Suggestion Fields  
  - Set KW Ideas Fields  
  - Set Autocomplete Fields  
  - Set Subtopics Fields  
  - Set SERP Fields  
  - Set PAA Fields  

- **Node Details:**  

  - All are `set` nodes.  
  - Role: Map API response fields to a normalized schema suitable for storage in Google Sheets.  
  - Configuration: Uses expressions to extract relevant properties (e.g., keyword text, search volume, difficulty).  
  - Inputs: Corresponding Split Out nodes.  
  - Outputs: Connect to respective Google Sheets nodes for insertion.  
  - Failures: Expression evaluation errors if expected fields are missing; handle nulls gracefully.  

#### 2.5 Data Storage

- **Overview:** Inserts the processed keyword and subtopic data into dedicated Google Sheets spreadsheets for organized storage and further analysis.  
- **Nodes Involved:**  
  - Add Related KWs to Related KWs Sheet  
  - Add Related KWs to Master all KW Variations Sheet  
  - Add KWs to Keyword Suggestions Sheet  
  - Add KWs to Master All KW Sheet  
  - Add KWs to Keyword Ideas Sheet  
  - Add KWs to Master All KW Sheet1  
  - Add Autocomplete to Autocomplete Sheet  
  - Add Subtopics to Subtopics Sheet  
  - Add Subtopics to Master KW sheet  

- **Node Details:**  

  - All are `googleSheets` nodes.  
  - Role: Append rows to target sheets representing different keyword categories or consolidated master sheets.  
  - Configuration: Uses Google Sheets API credentials with write access, sheet and range defined.  
  - Inputs: Output from corresponding Set Fields nodes.  
  - Outputs: Some trigger further consolidation steps.  
  - Failures: API auth errors, write permission issues, quota limits.  

#### 2.6 SERP and PAA Filtering and Storage

- **Overview:** Filters SERP and PAA data to separate relevant entries, formats their fields, and stores the results into dedicated Google Sheets.  
- **Nodes Involved:**  
  - Split Out SERPs and PAA  
  - Filter SERPs  
  - Filter PAA  
  - Set SERP Fields  
  - Set PAA Fields  
  - Add SERPs to SERP Sheet  
  - Add PAA to PAA Sheet  
  - Add PAA to Master KW Variations Sheet  

- **Node Details:**  

  - **Split Out SERPs and PAA**  
    - Splits combined API response into SERP and PAA items.  
  - **Filter SERPs / Filter PAA**  
    - Filters data based on criteria to separate SERP results from PAA questions.  
  - **Set SERP Fields / Set PAA Fields**  
    - Normalizes fields for storage.  
  - **Add SERPs to SERP Sheet / Add PAA to PAA Sheet / Add PAA to Master KW Variations Sheet**  
    - Stores filtered and formatted data into Google Sheets.  
  - Failures: Filtering logic may exclude data unexpectedly; verify filtering criteria; API limits and auth errors possible.  

#### 2.7 Master Data Consolidation

- **Overview:** Consolidates keyword variations, related keywords, and subtopics into master sheets for holistic overview and analysis.  
- **Nodes Involved:**  
  - Add Related KWs to Master all KW Variations Sheet  
  - Add KWs to Master All KW Sheet  
  - Add KWs to Master All KW Sheet1  
  - Add Subtopics to Master KW sheet  
  - Add PAA to Master KW Variations Sheet  

- **Node Details:**  

  - These are Google Sheets append operations consolidating data from various categories into master sheets.  
  - Role: Provide unified data repository for analysis and reporting.  
  - Inputs: From respective data category storage nodes.  
  - Outputs: End of data pipeline.  
  - Failures: Same as other Google Sheets nodes.  

---

### 3. Summary Table

| Node Name                           | Node Type          | Functional Role                        | Input Node(s)                        | Output Node(s)                           | Sticky Note                         |
|-----------------------------------|--------------------|-------------------------------------|------------------------------------|----------------------------------------|-----------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger     | Workflow start                      | None                               | Google Sheets KW Research Template      |                                   |
| Google Sheets KW Research Template | Google Sheets      | Load KW research template           | When clicking ‘Test workflow’      | Google Drive Create KW Folder            |                                   |
| Google Drive Create KW Folder      | Google Drive       | Create folder for KW files          | Google Sheets KW Research Template | Google Drive Copy KW Template            |                                   |
| Google Drive Copy KW Template      | Google Drive       | Copy KW template to new folder      | Google Drive Create KW Folder      | Set Main Fields                         |                                   |
| Set Main Fields                   | Set                | Initialize main API and workflow params | Google Drive Copy KW Template      | HTTP Related Keywords, HTTP Keyword Suggestions, HTTP Keyword Ideas, HTTP Autocomplete, HTTP Subtopics, HTTP SERPs |                                   |
| HTTP Related Keywords            | HTTP Request       | Fetch related keywords              | Set Main Fields                   | Split Out Related KWs                    |                                   |
| Split Out Related KWs            | Split Out          | Split related keywords array       | HTTP Related Keywords             | Set Related KW Fields                    |                                   |
| Set Related KW Fields            | Set                | Format related keyword fields      | Split Out Related KWs             | Add Related KWs to Related KWs Sheet    |                                   |
| Add Related KWs to Related KWs Sheet | Google Sheets  | Store related keywords              | Set Related KW Fields             | Add Related KWs to Master all KW Variations Sheet |                         |
| Add Related KWs to Master all KW Variations Sheet | Google Sheets | Consolidate related KWs to master sheet | Add Related KWs to Related KWs Sheet |                                   |                                   |
| HTTP Keyword Suggestions         | HTTP Request       | Fetch keyword suggestions          | Set Main Fields                   | Split Out KW Suggestions                 |                                   |
| Split Out KW Suggestions         | Split Out          | Split keyword suggestions array    | HTTP Keyword Suggestions          | Set KW Suggestion Fields                 |                                   |
| Set KW Suggestion Fields         | Set                | Format keyword suggestion fields   | Split Out KW Suggestions          | Add KWs to Keyword Suggestions Sheet    |                                   |
| Add KWs to Keyword Suggestions Sheet | Google Sheets   | Store keyword suggestions          | Set KW Suggestion Fields          | Add KWs to Master All KW Sheet           |                                   |
| Add KWs to Master All KW Sheet   | Google Sheets      | Consolidate keyword suggestions    | Add KWs to Keyword Suggestions Sheet |                                   |                                   |
| HTTP Keyword Ideas              | HTTP Request       | Fetch keyword ideas                | Set Main Fields                   | Split Out KW Ideas                       |                                   |
| Split Out KW Ideas              | Split Out          | Split keyword ideas array          | HTTP Keyword Ideas                | Set KW Ideas Fields                      |                                   |
| Set KW Ideas Fields             | Set                | Format keyword ideas fields        | Split Out KW Ideas                | Add KWs to Keyword Ideas Sheet           |                                   |
| Add KWs to Keyword Ideas Sheet  | Google Sheets      | Store keyword ideas                | Set KW Ideas Fields               | Add KWs to Master All KW Sheet1          |                                   |
| Add KWs to Master All KW Sheet1  | Google Sheets      | Consolidate keyword ideas          | Add KWs to Keyword Ideas Sheet    |                                        |                                   |
| HTTP Autocomplete              | HTTP Request       | Fetch autocomplete keywords       | Set Main Fields                  | Split Out Autocomplete                   |                                   |
| Split Out Autocomplete          | Split Out          | Split autocomplete array           | HTTP Autocomplete                | Set Autocomplete Fields                  |                                   |
| Set Autocomplete Fields          | Set                | Format autocomplete keyword fields | Split Out Autocomplete            | Add Autocomplete to Autocomplete Sheet  |                                   |
| Add Autocomplete to Autocomplete Sheet | Google Sheets | Store autocomplete keywords        | Set Autocomplete Fields          | Add Subtopics to Master KWs Sheet        |                                   |
| HTTP Subtopics                | HTTP Request       | Fetch keyword subtopics            | Set Main Fields                  | Split Out Subtopics                      |                                   |
| Split Out Subtopics            | Split Out          | Split subtopics array              | HTTP Subtopics                  | Set Subtopics Fields                     |                                   |
| Set Subtopics Fields            | Set                | Format subtopics fields            | Split Out Subtopics              | Add Subtopics to Subtopics Sheet          |                                   |
| Add Subtopics to Subtopics Sheet | Google Sheets      | Store subtopics                   | Set Subtopics Fields             | Add Subtopics to Master KW sheet          |                                   |
| Add Subtopics to Master KW sheet | Google Sheets      | Consolidate subtopics              | Add Subtopics to Subtopics Sheet |                                        |                                   |
| HTTP SERPs                   | HTTP Request       | Fetch SERPs and PAA data          | Set Main Fields                  | Split Out SERPs and PAA                  |                                   |
| Split Out SERPs and PAA         | Split Out          | Split SERPs and PAA array          | HTTP SERPs                     | Filter SERPs, Filter PAA                 |                                   |
| Filter SERPs                   | Filter             | Filter SERP results                | Split Out SERPs and PAA          | Set SERP Fields                         |                                   |
| Set SERP Fields                | Set                | Format SERP fields                 | Filter SERPs                   | Add SERPs to SERP Sheet                 |                                   |
| Add SERPs to SERP Sheet          | Google Sheets      | Store SERP data                   | Set SERP Fields                |                                        |                                   |
| Filter PAA                    | Filter             | Filter PAA results                | Split Out SERPs and PAA          | Split Out PAA                          |                                   |
| Split Out PAA                 | Split Out          | Split PAA array                   | Filter PAA                    | Set PAA Fields                        |                                   |
| Set PAA Fields                | Set                | Format PAA fields                 | Split Out PAA                 | Add PAA to PAA Sheet                  |                                   |
| Add PAA to PAA Sheet           | Google Sheets      | Store PAA data                   | Set PAA Fields                | Add PAA to Master KW Variations Sheet   |                                   |
| Add PAA to Master KW Variations Sheet | Google Sheets | Consolidate PAA data             | Add PAA to PAA Sheet           |                                        |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Sheets Node ("Google Sheets KW Research Template")**  
   - Connect from Manual Trigger.  
   - Configure to read the base keyword research template spreadsheet (read operation).  
   - Credentials: Google Sheets with read access.

3. **Add Google Drive Node ("Google Drive Create KW Folder")**  
   - Connect from Google Sheets node.  
   - Configure to create a new folder for keyword research files.  
   - Specify folder name dynamically (e.g., with date or project name).  
   - Credentials: Google Drive with create folder permission.

4. **Add Google Drive Node ("Google Drive Copy KW Template")**  
   - Connect from folder creation node.  
   - Configure to copy the base keyword template file into the newly created folder.  
   - Specify source file ID and destination folder ID.  
   - Credentials: Google Drive with file copy permission.

5. **Add Set Node ("Set Main Fields")**  
   - Connect from copy operation node.  
   - Set variables for API calls such as the main keyword, API credentials reference, region, language, and other parameters required by DataForSEO endpoints.

6. **Add HTTP Request Nodes for Each Data Type:**  
   - Connect all from "Set Main Fields" node (parallel execution).  
   - Configure each with DataForSEO API endpoint URLs and parameters for:  
     - Related Keywords  
     - Keyword Suggestions  
     - Keyword Ideas  
     - Autocomplete Keywords  
     - Subtopics  
     - SERPs and PAA data  
   - Use credential of type HTTP or OAuth with API key/token.  
   - Set request method to GET or POST as per API docs.  
   - Handle authentication headers and query/body parameters properly.

7. **Add Split Out Nodes for Each HTTP Request Response:**  
   - Connect each HTTP node to corresponding Split Out node to iterate over array results.

8. **Add Set Nodes to Format Each Split Item:**  
   - Connect from each Split Out node.  
   - Map API response fields to standardized internal fields (e.g., keyword text, search volume, CPC, difficulty).  
   - Handle missing or null values with defaulting expressions.

9. **Add Google Sheets Nodes to Store Each Data Category:**  
   - Connect from each Set node.  
   - Configure to append rows in specific sheets: Related Keywords, Keyword Suggestions, Keyword Ideas, Autocomplete, Subtopics, SERP, PAA.  
   - Use appropriate sheet names and column mappings.  
   - Credentials: Google Sheets with write access.

10. **Add Filter Nodes for SERPs and PAA:**  
    - Connect from "Split Out SERPs and PAA" node.  
    - Configure filter conditions to separate SERP entries and PAA questions.

11. **Add Additional Split and Set Nodes for PAA Data:**  
    - Split filtered PAA data further if needed.  
    - Set normalized fields for PAA entries.

12. **Add Google Sheets Nodes to Store Filtered SERPs and PAA:**  
    - Append SERPs and PAA to their respective sheets.  
    - Also append PAA data into master keyword variations sheet.

13. **Add Google Sheets Nodes for Master Sheet Consolidation:**  
    - Connect from all category-specific storage nodes to append data into master sheets for unified overview.

14. **Test Workflow:**  
    - Run manual trigger.  
    - Monitor API responses and Google Sheets to verify correct data flow and storage.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                    |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow uses DataForSEO API to automate comprehensive keyword and SERP analysis workflows.     | https://dataforseo.com/api/                                       |
| Ensure all Google API credentials have required scopes (Sheets and Drive) authorized beforehand.| Google Cloud Console API & Credentials setup                      |
| API rate limits and quotas may apply; handle possible HTTP request failures gracefully.          | API documentation and n8n error handling best practices          |
| This workflow is designed for manual trigger but can be adapted for scheduled triggers.          | n8n scheduling options documentation                              |

---

This document provides a full technical reference for the "Automated SEO Keyword & SERP Analysis with DataForSEO for High-Converting Content" workflow, enabling reproduction, customization, and troubleshooting.