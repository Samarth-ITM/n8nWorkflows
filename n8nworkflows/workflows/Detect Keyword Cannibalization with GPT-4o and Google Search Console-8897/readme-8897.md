Detect Keyword Cannibalization with GPT-4o and Google Search Console

https://n8nworkflows.xyz/workflows/detect-keyword-cannibalization-with-gpt-4o-and-google-search-console-8897


# Detect Keyword Cannibalization with GPT-4o and Google Search Console

### 1. Workflow Overview

This n8n workflow is designed to detect keyword cannibalization risks for multiple client websites by analyzing Google Search Console (GSC) data combined with target keywords listed in a Google Sheet. It leverages AI (OpenAI GPT-4o) to categorize the cannibalization risk based on how many pages from the same domain compete for the same keyword.

The workflow is structured into the following logical blocks:

**1.1 Trigger & Input Reception**  
- Monitors a Google Sheet for keyword changes to initiate processing in near real-time.  
- Retrieves client website URLs and target keywords from separate sheets.

**1.2 Client Routing & GSC Data Fetching**  
- Routes processing depending on the client’s website URL to fetch GSC data for up to 4 clients in parallel via API calls for the last 30 days.

**1.3 Data Transformation & Integration**  
- Processes raw GSC data separately for each client to group keyword data by URLs and their performance metrics.  
- Merges all grouped GSC data with the target keywords from the sheet into a unified dataset.

**1.4 Keyword Matching & Filtering**  
- Matches target keywords from the sheet against actual GSC data to identify which keywords rank and which do not.  
- Filters out keywords not found in GSC to focus AI analysis on relevant data only.

**1.5 AI-powered Cannibalization Risk Analysis**  
- Uses GPT-4o via a LangChain agent to evaluate keyword cannibalization risk levels based on ranking page counts and performance distribution per domain.

**1.6 Results Parsing & Reporting**  
- Parses the AI’s natural language output into structured JSON.  
- Writes the structured risk analysis results back to a Google Sheet for reporting and client review.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

**Overview:**  
Monitors keyword changes in Google Sheets and fetches client URLs and target keywords to start the workflow.

**Nodes Involved:**  
- Monitor Keywords Sheet for Changes  
- Fetch Client Website URLs  
- Fetch Target Keywords from Sheet

**Node Details:**  

- **Monitor Keywords Sheet for Changes**  
  - Type: Google Sheets Trigger  
  - Role: Triggers workflow when keywords sheet is modified (polls every minute).  
  - Config: Monitors specific sheet and document ID for changes.  
  - Inputs: External (sheet changes)  
  - Outputs: Client URLs fetch, Target keywords fetch  
  - Edge cases: Missing sheet access, API quota limits.

- **Fetch Client Website URLs**  
  - Type: Google Sheets  
  - Role: Retrieves client website URLs to determine routing.  
  - Config: Reads from a specific sheet dedicated to client URLs.  
  - Inputs: Trigger from previous node  
  - Outputs: List of client URLs for routing  
  - Edge cases: Empty sheet, invalid/malformed URLs.

- **Fetch Target Keywords from Sheet**  
  - Type: Google Sheets  
  - Role: Reads target keywords for cannibalization analysis.  
  - Config: Reads from a dedicated keywords sheet.  
  - Inputs: Trigger from monitor node  
  - Outputs: Keywords list for merging and analysis  
  - Edge cases: Empty keyword list.

---

#### 2.2 Client Routing & GSC Data Fetching

**Overview:**  
Routes the workflow based on client website URLs to fetch GSC data for each client independently.

**Nodes Involved:**  
- Route to Client 1  
- Route to Client 2  
- Route to Client 3  
- Route to Client 4  
- Fetch GSC Data (Client 1)  
- Fetch GSC Data (Client 2)  
- Fetch GSC Data (Client 3)  
- Fetch GSC Data (Client 4)

**Node Details:**  

- **Route to Client [1-4]**  
  - Type: If (conditional routing)  
  - Role: Checks if the client URL matches a predefined domain pattern to route processing.  
  - Config: String contains conditional matching on the client website URL field.  
  - Inputs: Client URLs list  
  - Outputs: Corresponding Fetch GSC Data node for matched client  
  - Edge cases: URL not matching any condition, case sensitivity in match.

- **Fetch GSC Data (Client [1-4])**  
  - Type: HTTP Request  
  - Role: Calls Google Search Console API for last 30 days’ search analytics data (dimensions: query & page).  
  - Config: POST request with JSON body specifying date range and dimensions; uses Google OAuth2 credentials.  
  - Inputs: Routed client URL from routing nodes  
  - Outputs: Raw GSC response with keyword-page performance data  
  - Edge cases: API rate limits, authentication errors, empty data from GSC, network timeouts.

---

#### 2.3 Data Transformation & Integration

**Overview:**  
Converts raw GSC API response into structured data grouped by keyword for each client and merges all client data with target keywords.

**Nodes Involved:**  
- Group GSC Data by Keyword (Client 1)  
- Group GSC Data by Keyword (Client 2)  
- Group GSC Data by Keyword (Client 3)  
- Group GSC Data by Keyword (Client 4)  
- Merge All Client GSC Data

**Node Details:**  

- **Group GSC Data by Keyword (Client [1-4])**  
  - Type: Code  
  - Role: Transforms flat GSC rows into grouped objects keyed by keyword, each containing an array of URLs with position, clicks, impressions, and CTR.  
  - Config: JavaScript iterates over rows, groups data, returns array of keyword objects.  
  - Inputs: Raw GSC data from HTTP Request nodes  
  - Outputs: Structured data grouped by keyword  
  - Edge cases: Missing or malformed GSC rows, empty arrays, data inconsistency.

- **Merge All Client GSC Data**  
  - Type: Merge  
  - Role: Combines GSC data from all 4 clients plus target keywords into a single dataset for matching.  
  - Config: Merges 5 inputs (4 client groups + target keywords) into one output.  
  - Inputs: Grouped GSC data + target keywords  
  - Outputs: Unified dataset for keyword matching  
  - Edge cases: Mismatched input counts, missing data from any client.

---

#### 2.4 Keyword Matching & Filtering

**Overview:**  
Matches target keywords from sheet with GSC data to identify ranking keywords and filters out keywords not found in GSC.

**Nodes Involved:**  
- Match Keywords from Sheet with GSC Data  
- Filter Keywords Found in GSC

**Node Details:**  

- **Match Keywords from Sheet with GSC Data**  
  - Type: Code  
  - Role: Cross-references target keywords with GSC keywords; flags keywords as 'found_in_gsc' or 'not_found_in_gsc'.  
  - Config: JavaScript normalizes keywords for matching, outputs matched and unmatched keywords with URLs and status.  
  - Inputs: Merged dataset of keywords and GSC data  
  - Outputs: Keyword objects with status flags  
  - Edge cases: Case mismatches, missing URLs array, duplicate keywords.

- **Filter Keywords Found in GSC**  
  - Type: If  
  - Role: Filters out keywords with status 'not_found_in_gsc' to prevent unnecessary AI processing.  
  - Config: Condition excludes 'not_found_in_gsc' keywords.  
  - Inputs: Output from matching node  
  - Outputs: Only keywords found in GSC for AI analysis  
  - Edge cases: Keywords with unexpected status values.

---

#### 2.5 AI-powered Cannibalization Risk Analysis

**Overview:**  
Uses GPT-4o via LangChain agent to analyze keyword cannibalization risk based on domain and page count ranking the keyword.

**Nodes Involved:**  
- Analyze Keyword Cannibalization Risk  
- OpenAI GPT-4o Model  
- Parse AI Analysis to Structured JSON

**Node Details:**  

- **Analyze Keyword Cannibalization Risk**  
  - Type: LangChain Agent  
  - Role: Sends input of keyword and URLs to AI to analyze domain-level cannibalization risk using defined logic and prompt.  
  - Config: Custom prompt instructing the AI on risk criteria and output formatting.  
  - Inputs: Filtered keywords with URLs and metrics  
  - Outputs: AI-generated natural language analysis  
  - Edge cases: AI model errors, prompt parsing errors, incomplete input data.

- **OpenAI GPT-4o Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides GPT-4o language model for the agent to perform natural language processing.  
  - Config: Model set to "gpt-4o", default options.  
  - Inputs: Prompt from Analyze Keyword Cannibalization Risk node  
  - Outputs: AI textual response  
  - Edge cases: API rate limits, authentication failures, model downtime.

- **Parse AI Analysis to Structured JSON**  
  - Type: LangChain Output Parser Structured  
  - Role: Converts AI text output into structured JSON with fields: Keyword, Domain, URLs, Risk Level, Reasoning, Observation, Summary, Remediation steps.  
  - Config: JSON schema example defined for output structure.  
  - Inputs: AI natural language output  
  - Outputs: Structured JSON for reporting  
  - Edge cases: Parsing failures if AI output is malformed.

---

#### 2.6 Results Parsing & Reporting

**Overview:**  
Saves the structured cannibalization analysis results back to Google Sheets for client visibility and tracking.

**Nodes Involved:**  
- Save Cannibalization Analysis Results

**Node Details:**  

- **Save Cannibalization Analysis Results**  
  - Type: Google Sheets  
  - Role: Appends or updates rows in Google Sheets with detailed cannibalization results including risk level, reasoning, summary, and remediation steps.  
  - Config: Maps structured JSON fields to sheet columns; matches on "Targetted_Keywords" for update or append.  
  - Inputs: Structured JSON from AI parser  
  - Outputs: Google Sheets updated rows  
  - Edge cases: Sheet access errors, write conflicts, data type mismatches.

---

### 3. Summary Table

| Node Name                          | Node Type                            | Functional Role                                    | Input Node(s)                                  | Output Node(s)                                | Sticky Note                                                                                                     |
|-----------------------------------|------------------------------------|---------------------------------------------------|-----------------------------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Monitor Keywords Sheet for Changes| Google Sheets Trigger               | Triggers workflow on keyword sheet changes        | External trigger                              | Fetch Client Website URLs, Fetch Target Keywords from Sheet | Monitors the Keywords Google Sheet for any changes and triggers the workflow when modifications are detected. Polls every minute. |
| Fetch Client Website URLs          | Google Sheets                      | Retrieves client website URLs                      | Monitor Keywords Sheet for Changes             | Route to Client 1, 2, 3, 4                     | Retrieves the list of client website URLs from the Google Sheet for routing.                                    |
| Fetch Target Keywords from Sheet   | Google Sheets                      | Reads target keywords to analyze                   | Monitor Keywords Sheet for Changes             | Merge All Client GSC Data                       | Reads the target keywords from the Google Sheet that need to be analyzed for cannibalization.                   |
| Route to Client 1                  | If                                | Routes processing to Client 1 based on URL         | Fetch Client Website URLs                       | Fetch GSC Data (Client 1)                       | Routes workflow execution to Client 1's GSC data fetching if URL matches.                                        |
| Route to Client 2                  | If                                | Routes processing to Client 2 based on URL         | Fetch Client Website URLs                       | Fetch GSC Data (Client 2)                       | Routes workflow execution to Client 2's GSC data fetching if URL matches.                                        |
| Route to Client 3                  | If                                | Routes processing to Client 3 based on URL         | Fetch Client Website URLs                       | Fetch GSC Data (Client 3)                       | Routes workflow execution to Client 3's GSC data fetching if URL matches.                                        |
| Route to Client 4                  | If                                | Routes processing to Client 4 based on URL         | Fetch Client Website URLs                       | Fetch GSC Data (Client 4)                       | Routes workflow execution to Client 4's GSC data fetching if URL matches.                                        |
| Fetch GSC Data (Client 1)          | HTTP Request                      | Fetches GSC search analytics data for Client 1    | Route to Client 1                              | Group GSC Data by Keyword (Client 1)            | Makes API call to Google Search Console for Client 1 (last 30 days).                                            |
| Fetch GSC Data (Client 2)          | HTTP Request                      | Fetches GSC search analytics data for Client 2    | Route to Client 2                              | Group GSC Data by Keyword (Client 2)            | Makes API call to Google Search Console for Client 2 (last 30 days).                                            |
| Fetch GSC Data (Client 3)          | HTTP Request                      | Fetches GSC search analytics data for Client 3    | Route to Client 3                              | Group GSC Data by Keyword (Client 3)            | Makes API call to Google Search Console for Client 3 (last 30 days).                                            |
| Fetch GSC Data (Client 4)          | HTTP Request                      | Fetches GSC search analytics data for Client 4    | Route to Client 4                              | Group GSC Data by Keyword (Client 4)            | Makes API call to Google Search Console for Client 4 (last 30 days).                                            |
| Group GSC Data by Keyword (Client 1)| Code                            | Groups GSC data by keyword for Client 1           | Fetch GSC Data (Client 1)                      | Merge All Client GSC Data                       | Groups GSC data by keyword for Client 1.                                                                        |
| Group GSC Data by Keyword (Client 2)| Code                            | Groups GSC data by keyword for Client 2           | Fetch GSC Data (Client 2)                      | Merge All Client GSC Data                       | Groups GSC data by keyword for Client 2.                                                                        |
| Group GSC Data by Keyword (Client 3)| Code                            | Groups GSC data by keyword for Client 3           | Fetch GSC Data (Client 3)                      | Merge All Client GSC Data                       | Groups GSC data by keyword for Client 3.                                                                        |
| Group GSC Data by Keyword (Client 4)| Code                            | Groups GSC data by keyword for Client 4           | Fetch GSC Data (Client 4)                      | Merge All Client GSC Data                       | Groups GSC data by keyword for Client 4.                                                                        |
| Merge All Client GSC Data           | Merge                            | Combines all client GSC data and target keywords  | Group GSC Data by Keyword (Client 1-4), Fetch Target Keywords from Sheet | Match Keywords from Sheet with GSC Data         | Combines GSC data from all clients plus target keywords into unified dataset.                                   |
| Match Keywords from Sheet with GSC Data | Code                        | Matches sheet keywords with GSC data, flags status| Merge All Client GSC Data                       | Filter Keywords Found in GSC                     | Cross-references target keywords with GSC data, adding status flags.                                           |
| Filter Keywords Found in GSC        | If                               | Filters out keywords not found in GSC              | Match Keywords from Sheet with GSC Data        | Analyze Keyword Cannibalization Risk, Save Cannibalization Analysis Results | Filters out keywords not found in GSC to prevent AI analysis of irrelevant keywords.                             |
| Analyze Keyword Cannibalization Risk| LangChain Agent                  | Uses AI to analyze cannibalization risk            | Filter Keywords Found in GSC                     | Parse AI Analysis to Structured JSON            | Uses AI to analyze keyword cannibalization risk based on domain page counts and performance.                    |
| OpenAI GPT-4o Model                 | LangChain LM Chat OpenAI         | Provides GPT-4o model for AI analysis              | Analyze Keyword Cannibalization Risk            | Analyze Keyword Cannibalization Risk (input)    | Provides the AI language model (GPT-4o) for cannibalization analysis.                                          |
| Parse AI Analysis to Structured JSON| LangChain Output Parser Structured| Parses AI response to structured JSON             | Analyze Keyword Cannibalization Risk            | Save Cannibalization Analysis Results            | Converts AI textual output into structured JSON for reporting.                                                 |
| Save Cannibalization Analysis Results| Google Sheets                   | Writes cannibalization results back to sheet       | Parse AI Analysis to Structured JSON            | None                                            | Writes final analysis results back to Google Sheets with detailed risk info and remediation suggestions.        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Google Sheets Trigger** node named "Monitor Keywords Sheet for Changes".  
   - Configure to monitor the sheet containing target keywords for changes. Set polling interval to every minute.  
   - Use the document ID and sheet name of the keywords sheet.

2. **Fetch Client Website URLs:**  
   - Add a **Google Sheets** node named "Fetch Client Website URLs".  
   - Connect it to the trigger node.  
   - Configure to read the sheet containing client website URLs. Use appropriate document ID and sheet name.

3. **Fetch Target Keywords from Sheet:**  
   - Add a **Google Sheets** node named "Fetch Target Keywords from Sheet".  
   - Connect it to the trigger node (parallel to step 2).  
   - Configure to read the target keywords sheet.

4. **Create Routing Nodes for Each Client:**  
   - Add four **If** nodes named "Route to Client 1", "Route to Client 2", "Route to Client 3", and "Route to Client 4".  
   - Connect the "Fetch Client Website URLs" node to all four routing nodes.  
   - Configure each node with a string "contains" condition matching the client website domain or URL pattern.

5. **Fetch GSC Data for Each Client:**  
   - For each client, add an **HTTP Request** node named "Fetch GSC Data (Client X)".  
   - Connect each routing node to respective HTTP Request node.  
   - Configure the HTTP Request as POST to:  
     `https://www.googleapis.com/webmasters/v3/sites/{{ encodeURIComponent($json['Client Website']) }}/searchAnalytics/query` (or `sc-domain:` prefix for domain properties).  
   - Set body JSON with startDate (30 days ago) and endDate (today), dimensions: ["query", "page"].  
   - Use Google OAuth2 credentials for authentication.  
   - Set “Execute Once” to true to avoid redundant calls.

6. **Group GSC Data by Keyword for Each Client:**  
   - Add four **Code** nodes named "Group GSC Data by Keyword (Client X)".  
   - Connect each "Fetch GSC Data (Client X)" node to its corresponding Code node.  
   - Use JavaScript code to iterate over GSC rows, group by keyword, and output array of `{ keyword, urls }` objects. Each URL object includes url, position, clicks, impressions, and ctr.

7. **Merge All Client GSC Data and Target Keywords:**  
   - Add a **Merge** node named "Merge All Client GSC Data".  
   - Connect all four grouping nodes and the "Fetch Target Keywords from Sheet" node as inputs (total 5 inputs).  
   - Configure the merge to combine all inputs into one stream.

8. **Match Keywords from Sheet with GSC Data:**  
   - Add a **Code** node named "Match Keywords from Sheet with GSC Data".  
   - Connect the merge node to this code node.  
   - Implement logic to:  
     - Extract keywords from sheet and GSC data, normalize casing and trim.  
     - Flag keywords found in GSC with status "found_in_gsc" and those missing as "not_found_in_gsc".  
     - Attach URLs and performance data for found keywords.

9. **Filter Keywords Found in GSC:**  
   - Add an **If** node named "Filter Keywords Found in GSC".  
   - Connect from matching node.  
   - Configure condition to pass only keywords where status is not "not_found_in_gsc".

10. **Add AI Analysis Nodes:**  
    - Add a **LangChain Agent** node named "Analyze Keyword Cannibalization Risk".  
    - Connect the true output of the filter node to this AI node.  
    - Configure prompt to instruct AI to group pages by domain and categorize cannibalization risk levels (High, Moderate, Low, No) based on page counts and clicks/impressions distribution.

11. **Configure OpenAI GPT-4o Model:**  
    - Add a **LangChain LM Chat OpenAI** node named "OpenAI GPT-4o Model".  
    - Connect this node as the language model provider for the AI agent node.  
    - Set model to "gpt-4o".  
    - Provide valid OpenAI API credentials.

12. **Parse AI Output to Structured JSON:**  
    - Add a **LangChain Output Parser Structured** node named "Parse AI Analysis to Structured JSON".  
    - Connect from the AI agent node's output.  
    - Provide a JSON schema example for expected fields: Keyword, Domain, URLs for Keyword, Risk Level, Reasoning, Observation, Summary, Remediation steps.

13. **Save Cannibalization Analysis Results:**  
    - Add a **Google Sheets** node named "Save Cannibalization Analysis Results".  
    - Connect from the parsed JSON node.  
    - Configure to append or update rows in a dedicated output sheet.  
    - Map fields from parsed JSON to columns: Targetted_Keywords, Status, Domain, Date, Target page, Data (detailed URL metrics), Risk Level, Reasoning, Observation, Summary, remediation steps.  
    - Use the same Google Sheets document with appropriate sheet for results.  
    - Set matching column to "Targetted_Keywords" for update.

14. **Connect Filter False Output:**  
    - Connect the false output of "Filter Keywords Found in GSC" directly to "Save Cannibalization Analysis Results" node to save entries with no data as well.

15. **Test and Validate:**  
    - Test with live Google Sheet data and GSC API credentials.  
    - Validate AI responses and sheet updates.  
    - Handle any API errors or data mismatches gracefully.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow automates keyword cannibalization detection across multiple clients, integrating Google Search Console data with AI analysis for actionable insights. | Workflow purpose and design |
| See Google Search Console API documentation for query parameters and authentication: https://developers.google.com/webmaster-tools/search-console-api-original/v3/searchanalytics/query | GSC API reference |
| OpenAI GPT-4o is used as the language model for nuanced AI-driven keyword risk analysis. | OpenAI model choice |
| The workflow polls the keywords Google Sheet every minute to ensure near real-time processing of updates. | Trigger design |
| The JavaScript code nodes group flat GSC data by keyword to facilitate domain-level analysis. | Data transformation logic |
| Multiple client websites are handled in parallel with routing nodes based on URL matching. | Multi-client architecture |
| AI prompt logic includes domain extraction, page count thresholds, and click/impression dominance to categorize risk levels. | AI analysis criteria |
| Final results written back to Google Sheets include comprehensive fields for client reporting and remediation guidance. | Reporting and output |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.