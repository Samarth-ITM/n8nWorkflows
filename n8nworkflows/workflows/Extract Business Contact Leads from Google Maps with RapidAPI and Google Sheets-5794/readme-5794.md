Extract Business Contact Leads from Google Maps with RapidAPI and Google Sheets

https://n8nworkflows.xyz/workflows/extract-business-contact-leads-from-google-maps-with-rapidapi-and-google-sheets-5794


# Extract Business Contact Leads from Google Maps with RapidAPI and Google Sheets

# Extract Business Contact Leads from Google Maps with RapidAPI and Google Sheets

---

### 1. Workflow Overview

This workflow automates the extraction of business contact leads from Google Maps using the RapidAPI Local Business Data service, then stores the results in Google Sheets. It is designed for marketers, sales teams, and business developers seeking structured, enriched contact data for lead generation.

The workflow consists of the following logical blocks:

- **1.1 Input Reception & Filtering:** Loading search criteria from Google Sheets, loading existing leads, and filtering out already processed search queries to avoid duplicate API calls.
- **1.2 Parameter Configuration & API Request:** Preparing search parameters dynamically and querying the Google Maps data via RapidAPI.
- **1.3 Data Parsing & Structuring:** Extracting detailed business information and contact details from the API response into a normalized format.
- **1.4 Data Storage & Flow Control:** Saving new leads into Google Sheets, respecting duplicates, and implementing rate limiting to conform with API usage policies.
- **1.5 Workflow Initiation & User Guidance:** Entry point manual trigger and embedded sticky notes providing operational instructions and context.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Filtering

**Overview:**  
This block loads user-defined search criteria and existing leads from Google Sheets, then filters out searches that have already been processed to prevent redundant API calls and duplicate data.

**Nodes Involved:**  
- Start Lead Generation (Manual Trigger)  
- Load Search Criteria (Google Sheets)  
- Load Existing Leads (Google Sheets)  
- Filter New Searches (Merge)  
- Process Each Location (Split In Batches)  

---

**Node Details:**

- **Start Lead Generation**  
  - Type: Manual Trigger  
  - Role: Initiates workflow execution manually by user.  
  - Configuration: No parameters; user clicks to start.  
  - Connections: Outputs to Load Search Criteria and Load Existing Leads in parallel.  
  - Edge cases: No input validation; user must ensure search criteria configured.  

- **Load Search Criteria**  
  - Type: Google Sheets (Read)  
  - Role: Reads search queries from the 'keyword_searches' sheet filtered by 'X' in the 'select' column.  
  - Configuration: Filters rows where `select == "X"`. Reads columns including query, lat, lon, country_iso_code.  
  - Connections: Outputs to Filter New Searches (input1).  
  - Credentials: Requires Google Sheets OAuth2 credentials linked to the user’s Google Sheet.  
  - Edge cases: Empty or misformatted sheet may cause no queries to be processed.  

- **Load Existing Leads**  
  - Type: Google Sheets (Read)  
  - Role: Reads existing leads from 'stores_data' sheet for duplicate detection.  
  - Connections: Outputs to Filter New Searches (input2).  
  - Credentials: Requires Google Sheets OAuth2 credentials.  
  - Edge cases: Missing or inaccessible sheet may disable duplicate detection, causing duplicates or errors.  

- **Filter New Searches**  
  - Type: Merge  
  - Role: Compares loaded search queries with existing leads by matching on `query` and `lat`. Keeps only new, unprocessed queries to reduce API calls.  
  - Configuration:  
    - Mode: Combine  
    - Join Mode: Keep Non-Matches (passes queries that don’t match existing leads)  
    - Fields to Match: `query`, `lat`  
  - Inputs: Search Criteria (input1), Existing Leads (input2).  
  - Outputs: New searches to Process Each Location.  
  - Edge cases: Matching depends on exact string/number match; slight variations might cause misses.  

- **Process Each Location**  
  - Type: Split In Batches  
  - Role: Iterates over filtered search queries one by one to avoid API overload.  
  - Configuration: Default batch size (1).  
  - Inputs: New search queries from Filter New Searches.  
  - Outputs: Batch to Configure Search Parameters.  
  - Edge cases: Large number of queries may slow workflow; batch size adjustable as needed.  

---

#### 2.2 Parameter Configuration & API Request

**Overview:**  
This block prepares the API request parameters based on each search query and calls the RapidAPI Local Business Data service to retrieve business leads from Google Maps.

**Nodes Involved:**  
- Configure Search Parameters (Set)  
- Google Maps API Request (HTTP Request)  

---

**Node Details:**

- **Configure Search Parameters**  
  - Type: Set  
  - Role: Constructs the search parameters required by the API from the current search query batch.  
  - Configuration: Sets fields `search`, `latitude`, `longitude`, `region`, `language`, `zoom`, and `limit` extracted from batch data or fixed defaults.  
    - `limit`: Max 100 results per query  
    - `zoom`: 13 (city-wide scope)  
    - `language`: EN (English)  
  - Inputs: Batch data from Process Each Location.  
  - Outputs: Configured parameters to Google Maps API Request.  
  - Edge cases: Parameter values must be valid; missing or malformed input may cause API errors.  

- **Google Maps API Request**  
  - Type: HTTP Request  
  - Role: Calls the RapidAPI Local Business Data endpoint to retrieve detailed business information for given parameters.  
  - Configuration:  
    - URL: `https://local-business-data.p.rapidapi.com/search-in-area`  
    - Method: POST with JSON body containing search parameters including `extract_emails_and_contacts` set to true.  
    - Authentication: Uses HTTP header authentication via stored RapidAPI credentials.  
    - On error: Continues workflow (does not stop on errors).  
  - Inputs: Configured parameters from Configure Search Parameters.  
  - Outputs: API response JSON to Parse Business Data.  
  - Edge cases:  
    - API quota exceeded or invalid credentials cause errors.  
    - Network timeouts or malformed request body failures possible.  
    - Partial responses may require error handling downstream.  

---

#### 2.3 Data Parsing & Structuring

**Overview:**  
Processes the raw API response to extract and normalize business and contact details into a structured JSON format for downstream usage.

**Nodes Involved:**  
- Parse Business Data (Code)  

---

**Node Details:**

- **Parse Business Data**  
  - Type: Code (JavaScript)  
  - Role: Parses API JSON response, extracts relevant fields like business name, address, contact emails, social media profiles, phone numbers, ratings, location data, etc.  
  - Configuration:  
    - Validates presence of `data` array in response.  
    - Extracts multiple fields with safe fallback to null if missing.  
    - Formats emails and phone numbers as comma-separated strings.  
    - Handles nested fields for social media and contact info.  
  - Inputs: API JSON response from Google Maps API Request.  
  - Outputs: Array of structured business lead objects to Save New Leads.  
  - Edge cases:  
    - Missing or malformed response data triggers error output.  
    - Unexpected schema changes in API response could cause parsing failures.  
  - Version: Requires support for JavaScript node with async capabilities (n8n v0.141+ recommended).  

---

#### 2.4 Data Storage & Flow Control

**Overview:**  
Saves new leads into Google Sheets while ensuring no duplicates are inserted. Implements a delay between API calls to respect rate limits.

**Nodes Involved:**  
- Save New Leads (Google Sheets)  
- Rate Limit Delay (Wait)  

---

**Node Details:**

- **Save New Leads**  
  - Type: Google Sheets (Append)  
  - Role: Appends newly parsed leads into the 'stores_data' sheet.  
  - Configuration:  
    - Maps multiple lead fields to corresponding columns in the sheet, e.g., name, email, website, address, ratings, social media.  
    - Uses defined schema with fields allowed for duplicate matching (business_id, query, name, phone_number, email, website, full_address, rating, linkedin, instagram).  
    - Duplicate filtering expected to be handled prior to this node (business-level duplicate prevention).  
  - Inputs: Structured lead data from Parse Business Data.  
  - Outputs: Triggers Rate Limit Delay.  
  - Credentials: Requires Google Sheets OAuth2 credentials.  
  - Edge cases:  
    - Sheet access issues or quota limits may cause failures.  
    - Incorrect mapping may cause data corruption.  

- **Rate Limit Delay**  
  - Type: Wait  
  - Role: Pauses the workflow for 10 seconds between API calls to avoid exceeding rate limits.  
  - Configuration: Fixed 10-second wait.  
  - Inputs: From Save New Leads.  
  - Outputs: Loops back to Process Each Location for next batch.  
  - Edge cases: Delay length may need adjustment depending on API plan or errors.  

---

#### 2.5 Workflow Initiation & User Guidance

**Overview:**  
Provides manual start trigger and embedded operational instructions via sticky notes to assist users in setup and execution.

**Nodes Involved:**  
- Start Lead Generation (Manual Trigger)  
- Sticky Note (multiple)  

---

**Node Details:**

- **Sticky Notes (multiple)**  
  - Type: Sticky Note  
  - Role: Present detailed instructions, data structure examples, duplicate prevention strategy, API configuration notes, extracted data description, and execution flow overview.  
  - Content Highlights:  
    - How to prepare Google Sheets tabs and credentials.  
    - Usage tips for search queries.  
    - Explanation of 2-level duplicate prevention (search and business level).  
    - API configuration defaults and rate limiting rationale.  
    - Summarized execution steps for clarity.  
  - Position: Strategically placed around workflow nodes for easy reference.  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                   | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                             |
|-------------------------|---------------------|-------------------------------------------------|----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Start Lead Generation    | Manual Trigger      | Initiates the workflow manually                   | -                          | Load Search Criteria, Load Existing Leads | 🚀 Click here to start extracting Google Maps business leads. Ensure your search criteria are configured in the Google Sheets 'keyword_searches' tab first. |
| Load Search Criteria     | Google Sheets       | Reads user search queries marked for processing  | Start Lead Generation       | Filter New Searches (input1) | 📋 Reads search criteria from the 'keyword_searches' sheet. Only processes rows marked with 'X' in the 'select' column. |
| Load Existing Leads      | Google Sheets       | Reads existing leads for duplicate detection     | Start Lead Generation       | Filter New Searches (input2) | 📊 Reads existing leads from the database to enable duplicate detection and prevent re-processing the same businesses.  |
| Filter New Searches      | Merge               | Filters out already processed searches            | Load Search Criteria, Load Existing Leads | Process Each Location      | 🔍 Smart duplicate prevention: Only processes search criteria that haven't been executed before, saving API calls and avoiding duplicate data. |
| Process Each Location    | Split In Batches    | Iterates over filtered queries to control flow   | Filter New Searches         | Configure Search Parameters | 🔄 Loops through each selected search criteria to avoid overwhelming the API and ensure reliable processing of multiple locations. |
| Configure Search Parameters | Set                | Prepares API request parameters                   | Process Each Location       | Google Maps API Request    | ⚙️ Prepares API search parameters from Google Sheets data. Modify the limit (max 100) and other parameters as needed for your use case. |
| Google Maps API Request  | HTTP Request        | Calls RapidAPI to get business data               | Configure Search Parameters | Parse Business Data        | 🗺️ Calls RapidAPI's Local Business Data service to extract comprehensive business information from Google Maps, including emails and social media contacts. |
| Parse Business Data      | Code (JavaScript)   | Parses and structures API response data           | Google Maps API Request     | Save New Leads             | 🔄 Processes API response and extracts all business information including contact details, social media profiles, and location data into a structured format. |
| Save New Leads           | Google Sheets       | Appends new business leads to storage sheet       | Parse Business Data         | Rate Limit Delay           | 💾 Saves only new business leads to the 'stores_data' sheet. Duplicates are filtered out by the previous merge step.     |
| Rate Limit Delay        | Wait                | Waits 10 seconds to respect API rate limits       | Save New Leads              | Process Each Location      | ⏱️ 10-second delay between API calls to respect rate limits and prevent being blocked. Adjust timing based on your API plan. |
| Sticky Note              | Sticky Note         | Provides workflow overview and instructions       | -                          | -                         | # Google Maps Lead Generation Workflow ... (full detailed content in notes section)                                    |
| Sticky Note1             | Sticky Note         | Describes input data structure                      | -                          | -                         | ## 📊 **INPUT DATA STRUCTURE** ... (example table and usage tips)                                                        |
| Sticky Note2             | Sticky Note         | Explains duplicate prevention strategy             | -                          | -                         | ## 🔄 **DUPLICATE PREVENTION** ... (two-level prevention explanation)                                                  |
| Sticky Note3             | Sticky Note         | Details API configuration parameters                | -                          | -                         | ## ⚙️ **API CONFIGURATION** ... (limits, zoom, language, rate limiting)                                                |
| Sticky Note4             | Sticky Note         | Summarizes extracted data fields                    | -                          | -                         | ## 📱 **EXTRACTED DATA** ... (business info, contacts, location data)                                                   |
| Sticky Note5             | Sticky Note         | Summarizes execution flow steps                      | -                          | -                         | ## 📈 **EXECUTION FLOW** ... (step-by-step workflow logic)                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `Start Lead Generation`  
   - Purpose: Manually start the workflow. No parameters needed.

2. **Create Google Sheets Node to Load Search Criteria:**  
   - Name: `Load Search Criteria`  
   - Operation: Read rows from the Google Sheet tab `keyword_searches`  
   - Filter: Only rows where column `select` equals `"X"`  
   - Required Columns: `select`, `query`, `lat`, `lon`, `country_iso_code`  
   - Credentials: Configure Google Sheets OAuth2 with access to your spreadsheet.

3. **Create Google Sheets Node to Load Existing Leads:**  
   - Name: `Load Existing Leads`  
   - Operation: Read rows from the Google Sheet tab `stores_data`  
   - Credentials: Same Google Sheets OAuth2 credentials.

4. **Create Merge Node to Filter New Searches:**  
   - Name: `Filter New Searches`  
   - Mode: Combine  
   - Join Mode: Keep Non-Matches (keep rows from input1 not found in input2)  
   - Fields to Match: `query`, `lat`  
   - Inputs:  
     - Input 1: Output from `Load Search Criteria`  
     - Input 2: Output from `Load Existing Leads`

5. **Create Split In Batches Node:**  
   - Name: `Process Each Location`  
   - Batch Size: 1 (default)  
   - Input: Output from `Filter New Searches`

6. **Create Set Node to Configure API Search Parameters:**  
   - Name: `Configure Search Parameters`  
   - Assign these fields from split batch item:  
     - `search` = `{{ $json.query }}`  
     - `latitude` = `{{ $json.lat }}`  
     - `longitude` = `{{ $json.lon }}`  
     - `region` = `{{ $json.country_iso_code }}`  
     - `language` = `"EN"` (default, adjust as needed)  
     - `zoom` = `"13"` (city-wide setting)  
     - `limit` = `"100"` (max results per API call)  
   - Input: Output from `Process Each Location`

7. **Create HTTP Request Node to Call Google Maps API:**  
   - Name: `Google Maps API Request`  
   - URL: `https://local-business-data.p.rapidapi.com/search-in-area`  
   - Method: POST  
   - Body Format: JSON  
   - Body Content:  
     ```json
     {
       "query": "{{ $json.search }}",
       "lat": "{{ $json.latitude }}",
       "lng": "{{ $json.longitude }}",
       "zoom": "{{ $json.zoom }}",
       "limit": "{{ $json.limit }}",
       "language": "{{ $json.language }}",
       "region": "{{ $json.region }}",
       "extract_emails_and_contacts": "true"
     }
     ```  
   - Authentication: Use HTTP Header Auth with RapidAPI credentials  
   - On Error: Continue workflow (do not stop)  
   - Input: Output from `Configure Search Parameters`

8. **Create Code Node to Parse API Response:**  
   - Name: `Parse Business Data`  
   - Language: JavaScript  
   - Paste the provided JavaScript code that:  
     - Validates and extracts the `data` array from the API response  
     - Extracts multiple business and contact fields safely  
     - Returns an array of normalized JSON objects for each business  
   - Input: Output from `Google Maps API Request`

9. **Create Google Sheets Node to Save New Leads:**  
   - Name: `Save New Leads`  
   - Operation: Append rows to the `stores_data` tab  
   - Map fields to sheet columns, including business name, email, phone, website, addresses, ratings, social profiles, etc.  
   - Credentials: Google Sheets OAuth2 with write access to your spreadsheet  
   - Input: Output from `Parse Business Data`

10. **Create Wait Node for Rate Limiting:**  
    - Name: `Rate Limit Delay`  
    - Duration: 10 seconds (adjustable)  
    - Input: Output from `Save New Leads`

11. **Connect Nodes in Sequence:**  
    - `Start Lead Generation` → `Load Search Criteria` and `Load Existing Leads` (parallel)  
    - `Load Search Criteria` + `Load Existing Leads` → `Filter New Searches`  
    - `Filter New Searches` → `Process Each Location`  
    - `Process Each Location` → `Configure Search Parameters`  
    - `Configure Search Parameters` → `Google Maps API Request`  
    - `Google Maps API Request` → `Parse Business Data`  
    - `Parse Business Data` → `Save New Leads`  
    - `Save New Leads` → `Rate Limit Delay`  
    - `Rate Limit Delay` → loops back to `Process Each Location` for next batch  

12. **Add Sticky Notes in Workflow Editor:**  
    - Add detailed instructions from sticky notes content to assist users with setup and operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow extracts Google Maps business leads including emails and social media contacts via RapidAPI Local Business Data API.                         | Overview and API usage                                                                                 |
| Ensure your Google Sheet has two tabs: `keyword_searches` (for input queries) and `stores_data` (for output leads).                                    | Sheet setup requirement                                                                               |
| Include columns in `keyword_searches`: `select` (mark with X to process), `query`, `lat`, `lon`, `country_iso_code`.                                   | Input data structure                                                                                   |
| Duplicate prevention uses two levels: filtering on search queries and filtering on business IDs in stored leads to avoid redundant processing.          | Data quality and cost-saving strategy                                                                 |
| API rate limit respected by 10-second wait between calls; adjust according to your RapidAPI plan.                                                      | Rate limiting best practice                                                                             |
| JavaScript code node requires n8n version supporting code execution with async and ES6 syntax (v0.141+ recommended).                                   | Version compatibility                                                                                   |
| Configure RapidAPI HTTP Header Auth credentials with your RapidAPI key for `Google Maps API Request` node.                                              | Credential setup                                                                                       |
| Google Sheets nodes require OAuth2 credentials with access to your Google Sheets document.                                                             | Credential setup                                                                                       |
| Sticky notes in the workflow provide comprehensive operational guidance and can be referenced at any time during usage.                              | User assistance                                                                                       |

---

**Disclaimer:** This documentation is generated from an n8n workflow for lawful and public data processing purposes only. All data handled complies with applicable content policies and legal standards.