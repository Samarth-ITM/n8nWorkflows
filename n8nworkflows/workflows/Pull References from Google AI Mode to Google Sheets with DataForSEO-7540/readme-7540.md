Pull References from Google AI Mode to Google Sheets with DataForSEO

https://n8nworkflows.xyz/workflows/pull-references-from-google-ai-mode-to-google-sheets-with-dataforseo-7540


# Pull References from Google AI Mode to Google Sheets with DataForSEO

### 1. Workflow Overview

This workflow automates the retrieval and recording of source references from Google's AI Mode search results using the DataForSEO API and stores them in a specified Google Sheet. It is designed for users who want to track and analyze the presence and details of web sources that appear in AI-generated search answers for particular keywords, locations, and languages.

**Target Use Cases:**
- Monitoring source references from AI Mode search results for SEO or content research.
- Automating data collection on source URLs, domains, titles, and descriptive text from Google AI Mode.
- Centralizing AI Mode SERP data in Google Sheets for further analysis or reporting.

**Logical Blocks:**

- **1.1 Scheduling Trigger:** Runs the workflow on a recurring 7-day interval.
- **1.2 Fetching AI Mode SERP Data:** Queries DataForSEO’s API to get AI Mode results based on predefined keyword, language, and location.
- **1.3 Data Splitting:** Processes the nested JSON response by splitting the results into manageable individual items and further splitting source references.
- **1.4 Storing Data:** Appends the extracted source reference details into a configured Google Sheet.

### 2. Block-by-Block Analysis

#### 1.1 Scheduling Trigger

- **Overview:** Initiates the workflow automatically every 7 days to ensure periodic data refresh without manual intervention.
- **Nodes Involved:**  
  - `Run evry 7 days`
  - `Sticky Note2`
  - `Sticky Note3` (contextual information)
- **Node Details:**

  - **Run evry 7 days**  
    - *Type:* Schedule Trigger  
    - *Configuration:* Interval set to every 7 days (daysInterval: 7)  
    - *Input:* None (trigger node)  
    - *Output:* Triggers the next node (`Get Google AI Mode SERP Data`)  
    - *Edge Cases:* If the n8n instance is offline during scheduled time, trigger will be delayed. Timezone considerations may affect exact run time.  
    - *Notes:* The schedule can be adjusted as needed for different frequencies.

  - **Sticky Note2**  
    - *Type:* Sticky Note  
    - *Content:* "The flow will run every 7 days. Adjust the schedule if needed."  
    - *Purpose:* Inform users about the periodicity.

  - **Sticky Note3**  
    - *Type:* Sticky Note  
    - *Content:* High-level explanation of the workflow purpose and steps.  
    - *Purpose:* Provides user context on the workflow’s functionality.

#### 1.2 Fetching AI Mode SERP Data

- **Overview:** Queries the DataForSEO API to retrieve AI Mode search results for a specified keyword, language, and location.
- **Nodes Involved:**  
  - `Get Google AI Mode SERP Data`  
  - `Sticky Note` (keyword, location, language instructions)
- **Node Details:**

  - **Get Google AI Mode SERP Data**  
    - *Type:* DataForSEO API Node  
    - *Configuration:*  
      - `resource`: serp  
      - `operation`: get-google-ai-mode-serp  
      - `keyword`: "why sky is blue" (default example; user should change)  
      - `language_name`: "english"  
      - `location_name`: "united states"  
      - Browser screen dimensions left empty (optional)  
    - *Credentials:* Uses configured DataForSEO API credentials  
    - *Input:* Trigger from schedule node  
    - *Output:* JSON containing AI Mode SERP results including nested `tasks[0].result[0].items` array  
    - *Expressions/Variables:* Keyword and location are hardcoded but meant to be parameterized by user  
    - *Edge Cases:*  
      - API rate limits or quota exceeded  
      - Invalid or missing credentials  
      - Network or timeout errors  
      - Empty or malformed response if keyword not found  
    - *Notes:* User must specify the keyword, language, and location relevant to their needs.

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Content:* "Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for."  
    - *Purpose:* Instruction for user input parameters.

#### 1.3 Data Splitting

- **Overview:** Processes the nested JSON response by splitting it first into individual item results and then further splitting the references within each item, preparing data for insertion into the sheet.
- **Nodes Involved:**  
  - `Split Out (items)`  
  - `Split Out (references)`
- **Node Details:**

  - **Split Out (items)**  
    - *Type:* Split Out Node  
    - *Configuration:* Splits array at path `tasks[0].result[0].items` from the API response  
    - *Input:* Output from `Get Google AI Mode SERP Data`  
    - *Output:* Each item from the AI Mode results as a separate output item  
    - *Edge Cases:* If the path does not exist or is empty, no items will be output.

  - **Split Out (references)**  
    - *Type:* Split Out Node  
    - *Configuration:* Splits array at path `references` inside each item  
    - *Input:* Output from `Split Out (items)`  
    - *Output:* Each source reference as a separate output item  
    - *Edge Cases:* Items without references will produce no output; empty references array results in no data passed downstream.

#### 1.4 Storing Data

- **Overview:** Appends each extracted reference’s details into a Google Sheet with predefined columns, enabling easy tracking and further analysis.
- **Nodes Involved:**  
  - `Record references to your Google Sheet.`  
  - `Sticky Note1`
- **Node Details:**

  - **Record references to your Google Sheet.**  
    - *Type:* Google Sheets Node  
    - *Operation:* Append rows  
    - *Configuration:*  
      - Document ID: `1XCjkjyVxrtpTUQenHeR3B07xfEZ489mhuVidjhGOO7I` (example)  
      - Sheet name: `Sheet1` (gid=0)  
      - Columns mapped: Source, Domain, URL, Title, Text — all dynamically populated from the JSON fields of the reference item (`$json.source`, `$json.domain`, etc.)  
      - Matching columns and type conversions disabled (append only)  
    - *Credentials:* Uses Google Sheets OAuth2 credentials  
    - *Input:* Output from `Split Out (references)`  
    - *Output:* None (terminal node)  
    - *Edge Cases:*  
      - Authentication/authorization errors if credentials expire or lack permissions  
      - API quota limits or rate limits on Google Sheets  
      - Data type mismatches if input data is malformed  
    - *Notes:* User must ensure the Google Sheet has the expected columns to match the data structure.

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Content:* "Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns."  
    - *Purpose:* User instruction to prepare the Google Sheet accordingly.

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                  | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                             |
|----------------------------------|---------------------------|--------------------------------|--------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------|
| Run evry 7 days                  | Schedule Trigger          | Initiates workflow every 7 days | None                           | Get Google AI Mode SERP Data      | The flow will run every 7 days. Adjust the schedule if needed.                                        |
| Get Google AI Mode SERP Data     | DataForSEO API            | Fetches AI Mode SERP data      | Run evry 7 days                | Split Out (items)                 | Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for.           |
| Split Out (items)                | Split Out                 | Splits AI Mode results into items | Get Google AI Mode SERP Data   | Split Out (references)            |                                                                                                       |
| Split Out (references)           | Split Out                 | Splits each item into source references | Split Out (items)              | Record references to your Google Sheet. |                                                                                                       |
| Record references to your Google Sheet. | Google Sheets           | Appends references to Google Sheet | Split Out (references)         | None                             | Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns. |
| Sticky Note                     | Sticky Note               | User instruction note           | None                           | None                            | Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for.           |
| Sticky Note1                    | Sticky Note               | User instruction note           | None                           | None                            | Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns. |
| Sticky Note2                    | Sticky Note               | User instruction note           | None                           | None                            | The flow will run every 7 days. Adjust the schedule if needed.                                        |
| Sticky Note3                    | Sticky Note               | Workflow explanation note       | None                           | None                            | This n8n template automatically collects all source references from Google’s AI Mode for any keyword you choose using DataForSEO and Google Sheets. What it does: * Uses DataForSEO API to fetch AI Mode results. * Extracts the source title, URL, and domain. * Stores the data in Google Sheets. You can use this data to track your website’s presence in AI Mode answers. |

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Node Type: Schedule Trigger  
   - Parameters: Set interval to every 7 days (daysInterval: 7)  
   - Connect output to next node.

2. **Create DataForSEO API Node**  
   - Node Type: DataForSEO  
   - Credentials: Configure DataForSEO API credentials with valid account  
   - Parameters:  
     - Resource: `serp`  
     - Operation: `get-google-ai-mode-serp`  
     - Keyword: Enter desired keyword (default example: "why sky is blue")  
     - Language Name: e.g., "english"  
     - Location Name: e.g., "united states"  
     - Leave browser screen parameters empty or set as needed  
   - Connect input to Schedule Trigger node output.

3. **Create Split Out Node for Items**  
   - Node Type: Split Out  
   - Parameters:  
     - Field to split out: `tasks[0].result[0].items`  
   - Connect input to DataForSEO node output.

4. **Create Split Out Node for References**  
   - Node Type: Split Out  
   - Parameters:  
     - Field to split out: `references`  
   - Connect input to previous Split Out (items) node output.

5. **Create Google Sheets Node**  
   - Node Type: Google Sheets  
   - Credentials: Configure with Google Sheets OAuth2 credentials, authorized to access the target spreadsheet  
   - Parameters:  
     - Operation: Append  
     - Document ID: ID of the Google Sheet to store data  
     - Sheet Name: Target sheet name or gid (e.g., "Sheet1" or "gid=0")  
     - Columns to map:  
       - Source: `={{ $json.source }}`  
       - Domain: `={{ $json.domain }}`  
       - URL: `={{ $json.url }}`  
       - Title: `={{ $json.title }}`  
       - Text: `={{ $json.text }}`  
     - Disable matching columns and type conversion (append only)  
   - Connect input to Split Out (references) node output.

6. **Add Sticky Notes as Needed**  
   - Add notes to instruct users on required inputs (Keyword, Location, Language) and Google Sheet setup.

7. **Save and Activate Workflow**  
   - Validate all connections and parameters.  
   - Activate the workflow to start running on schedule.

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses the DataForSEO API to fetch Google AI Mode SERP results and requires a valid DataForSEO account for API access.    | DataForSEO API documentation: https://docs.dataforseo.com/                                        |
| Google Sheets OAuth2 credentials must have write permissions for the target spreadsheet.                                              | Google Sheets API documentation: https://developers.google.com/sheets/api                         |
| The default keyword and location parameters should be customized by the user to match their tracking requirements.                   | N8N expression editing: https://docs.n8n.io/nodes/expressions/                                     |
| The workflow is designed to run every 7 days by default but can be adjusted to any schedule as needed in the Schedule Trigger node.  |                                                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.