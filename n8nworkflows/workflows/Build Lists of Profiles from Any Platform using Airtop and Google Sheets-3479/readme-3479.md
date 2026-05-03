Build Lists of Profiles from Any Platform using Airtop and Google Sheets

https://n8nworkflows.xyz/workflows/build-lists-of-profiles-from-any-platform-using-airtop-and-google-sheets-3479


# Build Lists of Profiles from Any Platform using Airtop and Google Sheets

### 1. Workflow Overview

This workflow automates the process of building targeted lists of profiles from any platform using Airtop's AI-powered data extraction and Google Sheets for storage. It is designed to reduce manual research time drastically by automating multi-source data collection, verification, deduplication, and consolidation into a structured spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger and parameter setup for defining the target audience and platform.
- **1.2 Initial Data Extraction:** Query Airtop to retrieve URLs of relevant web pages containing lists of profiles.
- **1.3 Profile Extraction:** Extract detailed profile information (name, handle, URL) from each retrieved URL.
- **1.4 Data Deduplication and Cleaning:** Consolidate and remove duplicate profiles based on URLs.
- **1.5 Data Storage:** Append the cleaned and deduplicated profile data into a Google Sheets spreadsheet for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and sets the search parameters defining the target audience ("who") and platform ("where") for list building.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Parameters (Set Node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters; simply triggers the workflow.  
    - Inputs: None  
    - Outputs: Connects to Parameters node  
    - Edge cases: None typical; user must manually trigger.

  - **Parameters**  
    - Type: Set  
    - Role: Defines the input parameters for the search query.  
    - Configuration:  
      - `who`: Target audience description (default: `Top "Build in Public" influencers`)  
      - `where`: Platform or domain (default: `X`)  
    - Inputs: From Manual Trigger  
    - Outputs: Connects to Get urls node  
    - Edge cases: User must provide meaningful parameters; empty or invalid values may lead to poor or no results.

#### 2.2 Initial Data Extraction

- **Overview:**  
  This block queries Airtop to perform a Google search based on the parameters and extracts up to 10 URLs of web pages likely containing lists of the target profiles.

- **Nodes Involved:**  
  - Get urls (Airtop Node)  
  - Format results (Code Node)

- **Node Details:**

  - **Get urls**  
    - Type: Airtop (Extraction - Query)  
    - Role: Sends a search query to Airtop to retrieve relevant web page URLs.  
    - Configuration:  
      - URL dynamically constructed as a Google search query combining `who` and `where` parameters.  
      - Prompt instructs Airtop to return up to 10 non-sponsored search results with title and URL.  
      - Output schema expects an array of objects with `title` and `url`.  
      - Session mode: new (fresh context for each query).  
      - Credentials: Airtop API Key required.  
    - Inputs: From Parameters node  
    - Outputs: Connects to Format results node  
    - Edge cases:  
      - API key invalid or expired → authentication failure.  
      - No results returned → empty output.  
      - Network timeout or rate limiting by Airtop or Google.  

  - **Format results**  
    - Type: Code (JavaScript)  
    - Role: Parses Airtop's JSON response and formats it into a list of URLs for the next extraction step.  
    - Configuration:  
      - Parses `data.modelResponse` JSON string to extract `results` array.  
      - Maps each result to an object with a single `url` property.  
    - Inputs: From Get urls node  
    - Outputs: Connects to Get people node  
    - Edge cases:  
      - Malformed JSON in response → parsing error.  
      - Empty results array → no output items.

#### 2.3 Profile Extraction

- **Overview:**  
  This block uses Airtop to extract detailed profile information (name, handle/ID, URL) from each URL obtained in the previous step.

- **Nodes Involved:**  
  - Get people (Airtop Node)

- **Node Details:**

  - **Get people**  
    - Type: Airtop (Extraction - Query)  
    - Role: Extracts up to 20 profile items from each URL, including name, identifier, and URL.  
    - Configuration:  
      - URL parameter is dynamically set from each input item's `url`.  
      - Prompt instructs Airtop to extract profile details based on the original `who` and `where` parameters.  
      - Output schema expects an array of `items` with `name`, `identifier`, and `url`.  
      - Session mode: new.  
      - Credentials: Airtop API Key required.  
    - Inputs: From Format results node (list of URLs)  
    - Outputs: Connects to Dedupe results node  
    - Edge cases:  
      - Invalid or unreachable URLs → extraction failure or empty results.  
      - API errors or rate limits.  
      - Partial data extraction if page structure is unexpected.

#### 2.4 Data Deduplication and Cleaning

- **Overview:**  
  This block consolidates all extracted profiles from multiple URLs, cleans URLs, filters out incomplete entries, and removes duplicates based on URL.

- **Nodes Involved:**  
  - Dedupe results (Code Node)

- **Node Details:**

  - **Dedupe results**  
    - Type: Code (JavaScript)  
    - Role: Aggregates all profile items, cleans URLs by removing query parameters, filters out entries without names, and deduplicates by URL.  
    - Configuration:  
      - Iterates over all input items from Get people node.  
      - Parses JSON response to extract `items`.  
      - Filters out items missing `name`.  
      - Cleans URLs by stripping query strings.  
      - Removes duplicates by URL, preserving first occurrence.  
    - Inputs: From Get people node (multiple items)  
    - Outputs: Connects to Add to spreadsheet node  
    - Edge cases:  
      - Malformed JSON → parsing errors.  
      - Empty input → no output.  
      - Duplicate URLs with different casing or trailing slashes may not be deduped perfectly.

#### 2.5 Data Storage

- **Overview:**  
  This block appends the cleaned and deduplicated profile data into a specified Google Sheets spreadsheet, including metadata such as the search parameters and timestamp.

- **Nodes Involved:**  
  - Add to spreadsheet (Google Sheets Node)

- **Node Details:**

  - **Add to spreadsheet**  
    - Type: Google Sheets (Append Operation)  
    - Role: Adds each profile as a new row in the target Google Sheets document.  
    - Configuration:  
      - Maps profile fields to columns: URL, Name, Who?, Where?, Added on (timestamp), ID or Handle.  
      - Spreadsheet ID and sheet name are preconfigured to a template spreadsheet.  
      - Uses Google OAuth2 credentials.  
      - Mapping mode: define below (explicit column mapping).  
    - Inputs: From Dedupe results node  
    - Outputs: None (end of workflow)  
    - Edge cases:  
      - Authentication failure with Google Sheets.  
      - Spreadsheet ID or sheet name invalid or inaccessible.  
      - API quota limits or rate limiting.  
      - Data type mismatches (unlikely due to string mapping).

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                         |
|----------------------------|-----------------------|----------------------------------------|-----------------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger        | Starts the workflow manually            | None                        | Parameters                |                                                                                                   |
| Parameters                 | Set                   | Defines search parameters (who, where) | When clicking ‘Test workflow’ | Get urls                  |                                                                                                   |
| Get urls                   | Airtop                | Queries Airtop for URLs of relevant pages | Parameters                  | Format results            | Requires Airtop API Key; constructs Google search query dynamically                                |
| Format results             | Code                  | Parses and formats URLs from Airtop response | Get urls                    | Get people                |                                                                                                   |
| Get people                 | Airtop                | Extracts profile details from each URL | Format results              | Dedupe results            | Requires Airtop API Key; extracts up to 20 profiles per URL                                       |
| Dedupe results             | Code                  | Cleans, filters, and deduplicates profiles | Get people                  | Add to spreadsheet        |                                                                                                   |
| Add to spreadsheet         | Google Sheets         | Appends profiles to Google Sheets       | Dedupe results              | None                      | Requires Google OAuth2 credentials; appends data to predefined spreadsheet                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters needed.

3. **Add a Set node:**  
   - Name: `Parameters`  
   - Connect Manual Trigger → Parameters  
   - Add two string fields:  
     - `who` with default value: `Top "Build in Public" influencers`  
     - `where` with default value: `X`  
   - These define the target audience and platform.

4. **Add an Airtop node:**  
   - Name: `Get urls`  
   - Connect Parameters → Get urls  
   - Set resource: `extraction`  
   - Operation: `query`  
   - Session mode: `new`  
   - URL parameter:  
     ```
     https://www.google.com/search?q={{ encodeURI($json.who + ' on ' + $json.where) }}
     ```  
   - Prompt:  
     ```
     Those are search results, return up to 10 non-sponsored results that lead to a web page with a list of {{$json.who}} on {{$json.where}}. For each return the title and URL.
     ```  
   - Output schema: Define JSON schema expecting an object with a `results` array containing `title` and `url` strings.  
   - Credentials: Configure Airtop API Key (create free at https://portal.airtop.ai/api-keys).

5. **Add a Code node:**  
   - Name: `Format results`  
   - Connect Get urls → Format results  
   - Code (JavaScript):  
     ```javascript
     const input = $input.first().json.data.modelResponse;
     const listOfLinks = JSON.parse(input).results;
     return listOfLinks.map(item => ({ json: { url: item.url } }));
     ```  
   - This extracts URLs for next step.

6. **Add another Airtop node:**  
   - Name: `Get people`  
   - Connect Format results → Get people  
   - Resource: `extraction`  
   - Operation: `query`  
   - Session mode: `new`  
   - URL parameter: `={{ $json.url }}` (dynamic from input)  
   - Prompt:  
     ```
     This is a list of {{ $('Parameters').item.json.who }} on {{ $('Parameters').item.json.where }}.
     Extract up to 20 items. For each person extract: 
     - name 
     - handle or ID 
     - URL
     ```  
   - Output schema: JSON schema expecting an object with an `items` array of objects with `name`, `identifier`, and `url`.  
   - Credentials: Airtop API Key.

7. **Add a Code node:**  
   - Name: `Dedupe results`  
   - Connect Get people → Dedupe results  
   - Code (JavaScript):  
     ```javascript
     const allResults = [];
     for (const inputItem of $input.all()) {
       const input = inputItem.json.data.modelResponse;
       const results = JSON.parse(input).items;
       const cleanedResults = results
         .filter(res => res.name)
         .map(res => ({ ...res, url: res.url.split('?')[0] }));
       allResults.push(...cleanedResults);
     }
     const uniqueList = allResults.filter((item, index, self) =>
       index === self.findIndex(t => t.url === item.url)
     );
     return uniqueList.map(item => ({ json: { ...item } }));
     ```  
   - This cleans and deduplicates profiles.

8. **Add a Google Sheets node:**  
   - Name: `Add to spreadsheet`  
   - Connect Dedupe results → Add to spreadsheet  
   - Operation: `append`  
   - Document ID: Use the ID of your copied Google Sheets template (e.g., `150eh4t5GyEBN_TcO5TDeNWpE2GzHR4hQWoNRbUpw7A0`)  
   - Sheet Name: `gid=0` (or your target sheet)  
   - Mapping mode: Define below with columns:  
     - URL → `{{$json.url}}`  
     - Name → `{{$json.name}}`  
     - Who? → `{{ $('Parameters').first().json.who }}`  
     - Where? → `{{ $('Parameters').first().json.where }}`  
     - Added on → `{{$now}}` (current timestamp)  
     - ID or Handle → `{{$json.identifier}}`  
   - Credentials: Configure Google OAuth2 credentials with access to the target spreadsheet.

9. **Save and activate the workflow.**

10. **Run the workflow manually to test.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow reduces research time by up to 90% and improves list accuracy using Airtop's AI-powered extraction. | Workflow description and use case overview.                                                        |
| Airtop API Key is required; create one for free at the Airtop Portal.                                         | https://portal.airtop.ai/api-keys                                                                   |
| Use the provided Google Sheets template as the target spreadsheet for storing results.                        | https://docs.google.com/spreadsheets/d/150eh4t5GyEBN_TcO5TDeNWpE2GzHR4hQWoNRbUpw7A0/edit?usp=sharing |
| Video demo available showing the list building process in action.                                             | Embedded GIF in original description (fileId:1097)                                                 |
| Best practices: use specific parameters, update regularly, combine multiple runs for comprehensive coverage.  | Workflow description section "Best Practices"                                                      |
| Potential next steps include automating outreach, lead scoring, and list maintenance.                         | Workflow description section "What's Next?"                                                        |

---

This documentation provides a complete, structured reference for understanding, reproducing, and extending the "Build Lists of Profiles from Any Platform using Airtop and Google Sheets" workflow in n8n.