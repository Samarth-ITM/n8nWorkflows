Role-Based LinkedIn Profile Discovery with Google Search for OSINT and Red Teams

https://n8nworkflows.xyz/workflows/role-based-linkedin-profile-discovery-with-google-search-for-osint-and-red-teams-6508


# Role-Based LinkedIn Profile Discovery with Google Search for OSINT and Red Teams

### 1. Workflow Overview

This workflow, titled **"Role-Based LinkedIn Profile Discovery with Google Search for OSINT and Red Teams"**, is designed to automate the process of gathering LinkedIn profiles related to specific roles or search queries using Google Search. It is targeted at OSINT (Open Source Intelligence) practitioners and Red Teams who need to efficiently build social graphs and identify relevant targets based on role-based search queries.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Query Setup**: Initiates the workflow manually and defines the search queries to be used for discovery.
- **1.2 Query Processing and External Search**: Splits the search queries into batches and performs Google Search API requests for each query.
- **1.3 Data Extraction and Storage**: Extracts LinkedIn profile data from the Google search results and appends the extracted profiles to a Google Sheets document for tracking and further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Query Setup

- **Overview**: This block triggers the workflow manually and sets the initial set of search queries that will be used to find LinkedIn profiles via Google Search.
- **Nodes Involved**:  
  - ⚡ Trigger SocialGraph  
  - 🧠 Set Search Queries

- **Node Details**:

  - **⚡ Trigger SocialGraph**  
    - *Type*: Manual Trigger  
    - *Role*: Starts the workflow on demand without any external input.  
    - *Configuration*: No parameters; it is triggered manually by the user.  
    - *Input/Output*: No input nodes; outputs to "🧠 Set Search Queries".  
    - *Edge Cases*: None specific, except user must trigger manually.  
    - *Version*: v1, standard manual trigger node.

  - **🧠 Set Search Queries**  
    - *Type*: Set Node  
    - *Role*: Defines and initializes the list of role-based search queries for Google Search.  
    - *Configuration*: Presumably contains an array of search queries (e.g., role titles, keywords related to LinkedIn profiles).  
    - *Input/Output*: Receives from "⚡ Trigger SocialGraph", outputs to "🔁 Split Queries".  
    - *Expressions*: May use static values or expressions to build queries.  
    - *Edge Cases*: Empty or malformed queries could cause downstream errors or empty search results.

#### 2.2 Query Processing and External Search

- **Overview**: This block takes the list of search queries, processes them in batches, and performs Google Search API requests to retrieve search results for each query.
- **Nodes Involved**:  
  - 🔁 Split Queries  
  - 🌐 Google Search API

- **Node Details**:

  - **🔁 Split Queries**  
    - *Type*: SplitInBatches  
    - *Role*: Processes the list of queries in manageable batches to comply with API rate limits or performance considerations.  
    - *Configuration*: Batch size parameter set (not visible from JSON, but typically between 1-10 queries per batch).  
    - *Input/Output*: Receives from "🧠 Set Search Queries", outputs to "🌐 Google Search API".  
    - *Edge Cases*: Improper batch size may cause slow execution or API throttling.

  - **🌐 Google Search API**  
    - *Type*: HTTP Request  
    - *Role*: Performs Google Search queries using an external API to fetch search results for each query batch.  
    - *Configuration*:  
      - HTTP method: GET or POST (likely GET)  
      - URL: Google Search API endpoint (custom or official)  
      - Authentication: May require API key or OAuth2 token (not explicitly shown)  
      - Query parameters: Incorporates current search query expression from batch node.  
    - *Input/Output*: Receives batched query from "🔁 Split Queries", outputs search results to "🧪 Extract Profiles".  
    - *Edge Cases*:  
      - API authentication failures  
      - Rate limiting or quota exhaustion  
      - Network timeouts or errors  
      - Malformed query parameters causing API errors

#### 2.3 Data Extraction and Storage

- **Overview**: This block extracts LinkedIn profile URLs or relevant data from the search results and appends the gathered information to a Google Sheet for record-keeping and further analysis.
- **Nodes Involved**:  
  - 🧪 Extract Profiles  
  - 📄 Append to RedOps_Targets

- **Node Details**:

  - **🧪 Extract Profiles**  
    - *Type*: Set Node  
    - *Role*: Parses the Google Search API response to extract LinkedIn profile URLs or other relevant fields.  
    - *Configuration*: Uses expressions or JSON parsing to filter and map the required data fields.  
    - *Input/Output*: Receives from "🌐 Google Search API", outputs to "📄 Append to RedOps_Targets".  
    - *Edge Cases*:  
      - Unexpected API response structure causing extraction failure  
      - Empty or no relevant profiles found  
      - Expression errors if fields do not exist

  - **📄 Append to RedOps_Targets**  
    - *Type*: Google Sheets Node  
    - *Role*: Appends extracted profiles as new rows in a specific Google Sheet document named "RedOps_Targets".  
    - *Configuration*:  
      - Spreadsheet ID and sheet name specified  
      - Append mode active  
      - Authentication via Google OAuth2 credentials required  
    - *Input/Output*: Receives from "🧪 Extract Profiles", outputs back to "🔁 Split Queries" for next batch processing iteration.  
    - *Edge Cases*:  
      - Google Sheets API quota or permission errors  
      - Invalid spreadsheet or sheet name  
      - Network interruptions during append operation

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                              | Input Node(s)           | Output Node(s)           | Sticky Note                  |
|-------------------------|-----------------------|----------------------------------------------|-------------------------|--------------------------|------------------------------|
| ⚡ Trigger SocialGraph   | Manual Trigger        | Starts the workflow manually                  | —                       | 🧠 Set Search Queries     |                              |
| 🧠 Set Search Queries    | Set                   | Defines the list of search queries            | ⚡ Trigger SocialGraph   | 🔁 Split Queries          |                              |
| 🔁 Split Queries         | SplitInBatches        | Splits queries into batches for processing    | 🧠 Set Search Queries    | 🌐 Google Search API      |                              |
| 🌐 Google Search API     | HTTP Request          | Performs Google Search API calls               | 🔁 Split Queries         | 🧪 Extract Profiles       |                              |
| 🧪 Extract Profiles      | Set                   | Extracts LinkedIn profiles from API results   | 🌐 Google Search API     | 📄 Append to RedOps_Targets |                              |
| 📄 Append to RedOps_Targets | Google Sheets       | Appends extracted profiles to Google Sheets   | 🧪 Extract Profiles      | 🔁 Split Queries          |                              |
| Sticky Note             | Sticky Note            | —                                              | —                       | —                        |                              |
| Sticky Note1            | Sticky Note            | —                                              | —                       | —                        |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `⚡ Trigger SocialGraph`  
   - Purpose: To manually start the workflow.

2. **Add a Set Node to Define Search Queries**  
   - Name: `🧠 Set Search Queries`  
   - Configure to set an array or list variable containing role-based search queries for LinkedIn profile discovery (e.g., ["Software Engineer LinkedIn", "Cybersecurity Analyst LinkedIn", ...]).  
   - Connect output of `⚡ Trigger SocialGraph` to this node.

3. **Add a SplitInBatches Node**  
   - Name: `🔁 Split Queries`  
   - Configure batch size (e.g., 1 or 5) to control the number of queries processed per batch.  
   - Connect output of `🧠 Set Search Queries` to this node.

4. **Add an HTTP Request Node for Google Search API**  
   - Name: `🌐 Google Search API`  
   - Configure:  
     - HTTP Method: GET  
     - URL: The Google Search API endpoint (or custom search API endpoint)  
     - Authentication: Configure API key or OAuth2 credentials as required by the API provider.  
     - Query Parameters: Use an expression to insert the current query from the batch (e.g., `{{$json["query"]}}` or equivalent).  
   - Connect output of `🔁 Split Queries` to this node.

5. **Add a Set Node to Extract Profiles**  
   - Name: `🧪 Extract Profiles`  
   - Configure expressions or JSON parsing to extract LinkedIn profile URLs or relevant data fields from the API response.  
   - Connect output of `🌐 Google Search API` to this node.

6. **Add a Google Sheets Node to Append Data**  
   - Name: `📄 Append to RedOps_Targets`  
   - Configure:  
     - Spreadsheet ID: Specify the target Google Sheet document.  
     - Sheet Name: Specify the worksheet/tab to append data to.  
     - Operation: Append rows.  
     - Authentication: Setup Google OAuth2 credentials with write permissions to the target sheet.  
   - Connect output of `🧪 Extract Profiles` to this node.  
   - Connect output of this node back to `🔁 Split Queries` to continue processing subsequent batches.

7. **Add Sticky Note Nodes (optional)**  
   - Add any notes or instructions relevant to the workflow for users.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                         |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow is part of a professional OSINT toolkit tagged as "Pro".                                    | Workflow metadata tag                  |
| Designed to integrate with Google Search API and Google Sheets for efficient data retrieval and storage. | Workflow integration details           |
| Manual trigger ensures controlled execution, suitable for iterative OSINT investigations.                 | Execution control note                  |
| Google API usage may require enabling billing and API quotas on Google Cloud Platform.                    | Google API usage considerations        |
| Google Sheets node requires OAuth2 credentials with permissions to the target spreadsheet.                | Credential setup instructions          |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.