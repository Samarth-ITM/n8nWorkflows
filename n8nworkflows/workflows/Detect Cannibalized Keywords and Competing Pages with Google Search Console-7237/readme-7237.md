Detect Cannibalized Keywords and Competing Pages with Google Search Console

https://n8nworkflows.xyz/workflows/detect-cannibalized-keywords-and-competing-pages-with-google-search-console-7237


# Detect Cannibalized Keywords and Competing Pages with Google Search Console

---

### 1. Workflow Overview

This workflow is designed to identify keyword cannibalization issues using data from Google Search Console (GSC). Keyword cannibalization occurs when multiple pages from the same website compete for the same query, potentially diluting SEO effectiveness. The workflow targets SEO professionals and website owners looking to detect such competing pages and queries that generate clicks on more than one page.

The workflow is logically divided into:

- **1.1 Input Reception**: Manual trigger to start the process.
- **1.2 Data Fetching from Google Search Console**: Retrieve search analytics data for the last 12 months with dimensions `query` and `page`.
- **1.3 Data Summarization and Aggregation**: Summarize results by query to group pages and clicks.
- **1.4 Filtering Cannibalized Queries**: Filter queries where more than one page has clicks, indicating true cannibalization.

Additional documentation and usage instructions are provided within a sticky note node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Initiates the workflow manually.

- **Nodes Involved:**  
  - Manual Start

- **Node Details:**  
  - **Manual Start**  
    - Type: Start node (trigger)  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None (entry point)  
    - Outputs: Triggers the subsequent node, "Query Search Analytics".  
    - Edge cases: None (manual trigger)  
    - Version-specific notes: Standard across n8n versions.

#### 2.2 Data Fetching from Google Search Console

- **Overview:**  
  Retrieves search analytics data from Google Search Console for the configured website property, spanning the last 12 months, with specific dimensions to detect queries and their corresponding pages.

- **Nodes Involved:**  
  - Query Search Analytics

- **Node Details:**  
  - **Query Search Analytics**  
    - Type: Google Search Console node  
    - Role: Fetch search analytics data with dimensions `query` and `page`  
    - Configuration:  
      - `siteUrl`: `"sc-domain:example.com"` (placeholder, must be replaced with actual property)  
      - `rowLimit`: 10,000 rows to ensure enough data coverage  
      - `operation`: `getPageInsights` - fetch detailed page-level insights  
      - `dimensions`: `["query","page"]` - to get query and page pairs  
      - `dateRangeMode`: `last12mo` - last 12 months data  
    - Key expressions/variables: None (data retrieval node)  
    - Input: Triggered by Manual Start  
    - Output: Emits raw data rows containing fields `query`, `page`, `clicks`, etc.  
    - Credentials: Requires Google Search Console OAuth2 credential  
    - Edge cases:  
      - Authentication failure if credential invalid or expired  
      - API quota limits or timeouts from GSC  
      - Incorrect `siteUrl` format leading to no data or errors  
      - Large data volume might cause slow execution

#### 2.3 Data Summarization and Aggregation

- **Overview:**  
  Aggregates the fetched data to group all pages and clicks under each query for easier analysis of competing pages.

- **Nodes Involved:**  
  - Summarize by Query

- **Node Details:**  
  - **Summarize by Query**  
    - Type: Summarize node  
    - Role: Group data by `query` and append associated pages and clicks into arrays  
    - Configuration:  
      - `fieldsToSplitBy`: `"query"` - group records by query string  
      - `fieldsToSummarize`:  
        - Append `page` values into `appended_page[]` array  
        - Append `clicks` values into `appended_clicks[]` array  
        - Keep `query` field as is  
    - Key expressions: None, direct aggregation operation  
    - Input: Output from "Query Search Analytics" node  
    - Output: Records with fields:  
      - `query` (string)  
      - `appended_page[]` (array of pages associated to the query)  
      - `appended_clicks[]` (array of clicks corresponding to each page)  
      - `count_query` (number of pages per query, auto-generated count from grouping)  
    - Edge cases:  
      - If data is empty, output will be empty  
      - Arrays might be empty or have inconsistent lengths if GSC data is incomplete  
      - Potential performance issues with very large datasets  
    - Version notes: Uses n8n Summarize node version 1.1

#### 2.4 Filtering Cannibalized Queries

- **Overview:**  
  Filters aggregated queries to keep only those where multiple pages have clicks, indicating true keyword cannibalization.

- **Nodes Involved:**  
  - Filter Cannibal Queries

- **Node Details:**  
  - **Filter Cannibal Queries**  
    - Type: Filter node  
    - Role: Apply conditions to identify cannibalized queries  
    - Configuration:  
      - Conditions (AND):  
        - `count_query` > 1 (query associated with more than one page)  
        - `appended_clicks[1]` > 0 (the second page in the list has clicks greater than zero)  
      - Case sensitive: true  
      - Type validation strict  
    - Key expressions:  
      - `={{ $json.count_query }}` and `={{ $json.appended_clicks[1] }}`  
      - Index `[1]` for second page clicks (zero-based indexing)  
    - Input: Output from "Summarize by Query" node  
    - Output: Only records matching the filter (true cannibalization)  
    - Edge cases:  
      - Queries with only one page or second page with zero clicks are filtered out  
      - If `appended_clicks` array has fewer than two elements, indexing `[1]` may cause errors or undefined behavior — expression must be carefully handled  
      - If clicks data is missing or not numeric, condition might fail or skip valid records  
    - Version notes: Uses Filter node version 2.2

#### 2.5 Workflow Description (Documentation)

- **Overview:**  
  Provides detailed documentation on the workflow's purpose, usage instructions, configuration tips, and output format.

- **Nodes Involved:**  
  - Workflow Description (Sticky Note)

- **Node Details:**  
  - **Workflow Description**  
    - Type: Sticky Note  
    - Role: Human-readable documentation inside the workflow canvas  
    - Content includes:  
      - Purpose: Detect keyword cannibalization with GSC data  
      - How it works: Stepwise logic explanation  
      - Pre-run requirements (OAuth2 credential, domain replacement)  
      - Tips for better accuracy and usage modes (manual or scheduled)  
      - Output fields description  
      - Tags for categorization (SEO, GSC, Cannibalization)  
    - Position: Informative, no input/output connections  
    - Edge cases: None

---

### 3. Summary Table

| Node Name             | Node Type                               | Functional Role                       | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                                                                                                                                                                   |
|-----------------------|---------------------------------------|-------------------------------------|---------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Description   | Sticky Note                           | Documentation of workflow purpose   | None                | None                     | ## 🔎 Find Query Cannibalization (Google Search Console) Purpose: Detect queries where multiple pages compete and receive clicks. Instructions for setup and usage included. Replace example domain with your own and connect GSC OAuth2.       |
| Manual Start          | Start                                | Manual trigger to start workflow    | None                | Query Search Analytics    |                                                                                                                                                                                                                                               |
| Query Search Analytics | Google Search Console                | Fetch GSC data with query and page  | Manual Start        | Summarize by Query       | Before running, connect Google Search Console OAuth2 credential. Replace `sc-domain:example.com` with your actual property (e.g., your domain).                                                                                                 |
| Summarize by Query    | Summarize                            | Aggregate pages and clicks per query| Query Search Analytics | Filter Cannibal Queries |                                                                                                                                                                                                                                               |
| Filter Cannibal Queries| Filter                              | Filter queries with >1 page and second page clicks > 0 | Summarize by Query   | None                     | Filter conditions ensure only queries with true cannibalization are output (more than one page with clicks). Beware of array indexing for clicks on second page (appended_clicks[1]).                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note node**  
   - Name: `Workflow Description`  
   - Content: Include detailed description of the workflow purpose, usage instructions, configuration reminders, and output fields as provided in the original sticky note.  
   - Position it separately for clarity; no connections.

2. **Create a Start node**  
   - Name: `Manual Start`  
   - Default manual trigger node with no parameters.

3. **Create a Google Search Console node**  
   - Name: `Query Search Analytics`  
   - Credentials: Set up and select your Google Search Console OAuth2 credentials.  
   - Parameters:  
     - `siteUrl`: Set to your property, e.g., `sc-domain:your-domain.com` (replace the placeholder).  
     - `operation`: Select `getPageInsights`.  
     - `dimensions`: Select `query` and `page`.  
     - `dateRangeMode`: Set to `last12mo` (last 12 months).  
     - `rowLimit`: Set to `10000`.  
   - Connect input from `Manual Start`.

4. **Create a Summarize node**  
   - Name: `Summarize by Query`  
   - Parameters:  
     - `fieldsToSplitBy`: `query`  
     - `fieldsToSummarize`:  
       - Append `page` values into an array.  
       - Append `clicks` values into an array.  
       - Keep `query` as is.  
   - Connect input from `Query Search Analytics`.

5. **Create a Filter node**  
   - Name: `Filter Cannibal Queries`  
   - Parameters:  
     - Set to version 2.2 or latest.  
     - Conditions (AND):  
       - Numeric `count_query` > 1  
       - Numeric `appended_clicks[1]` > 0 (second element in clicks array)  
     - Case sensitive: true  
     - Strict type validation enabled.  
   - Connect input from `Summarize by Query`.

6. **Connect nodes in sequence:**  
   `Manual Start` → `Query Search Analytics` → `Summarize by Query` → `Filter Cannibal Queries`

7. **Testing and Validation:**  
   - Replace domain placeholder in `Query Search Analytics`.  
   - Ensure Google Search Console OAuth2 credentials are valid and have access to the property.  
   - Execute manual start; verify data flows as expected.  
   - Check filtered output for queries with multiple pages receiving clicks.

8. **Optional Enhancements:**  
   - Adjust summarization aggregation from `append` to `sum` for clicks if you want aggregated click counts per page.  
   - Add scheduling trigger (e.g., weekly) for automation.  
   - Add error handling nodes for API call failures or credential issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Replace `sc-domain:example.com` with your actual Google Search Console property identifier before running the workflow.                                           | Critical setup step in "Query Search Analytics" node configuration.                                              |
| For more robust results, consider summing clicks per page and filtering queries with at least two pages having clicks greater than zero.                         | Workflow Description sticky note, under Tips section.                                                            |
| You can run this workflow manually via the start node or automate it using a scheduled (e.g., weekly) trigger for continuous monitoring.                         | Workflow Description sticky note, under How it works section.                                                    |
| Google Search Console API quotas and authentication tokens can impact the workflow execution; ensure OAuth2 tokens are valid and refreshed as needed.           | Best practice for Google Search Console integrations.                                                            |
| Keyword cannibalization can affect SEO rankings by splitting page authority and confusing search engines; this workflow aids in early detection and remediation. | SEO best practice rationale.                                                                                      |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---