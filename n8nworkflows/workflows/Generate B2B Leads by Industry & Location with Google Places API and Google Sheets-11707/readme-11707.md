Generate B2B Leads by Industry & Location with Google Places API and Google Sheets

https://n8nworkflows.xyz/workflows/generate-b2b-leads-by-industry---location-with-google-places-api-and-google-sheets-11707


# Generate B2B Leads by Industry & Location with Google Places API and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of B2B leads by industry and location using the Google Places API and Google Sheets. It is designed for sales teams, marketing agencies, or business developers looking to build targeted outreach lists or enrich CRM data with accurate company contact details.

The logical flow is divided into the following key blocks:

- **1.1 Input Reception:** Captures user input (industry + location query) via a form trigger.
- **1.2 Initial Data Retrieval:** Sends the first Google Places Text Search request to retrieve leads and a token for additional pages.
- **1.3 Pagination Handling:** Checks for next pages, waits to respect API token activation timing, and fetches up to two more results pages.
- **1.4 Data Consolidation:** Merges all retrieved pages into a single dataset.
- **1.5 Detailed Data Enrichment:** Splits merged results and fetches detailed place information (phone, website, address) for each place.
- **1.6 Data Formatting and Storage:** Formats the enriched data and appends it to a specified Google Sheet for later use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures the user’s search query specifying industry and location via an interactive form.

**Nodes Involved:**  
- Submit Search Query

**Node Details:**  
- **Submit Search Query**  
  - Type: Form Trigger  
  - Role: Entry point for the workflow, collects user input for the search query.  
  - Configuration: Form titled "Enter your Query" with a required field labeled "Query" (e.g., "Accountants London").  
  - Input: User submits the form input.  
  - Output: JSON containing the user query accessible as `$json.Query`.  
  - Potential failures: Form submission errors, webhook conflicts, or invalid/missing input.  
  - Version: 2.3

---

#### 2.2 Initial Data Retrieval

**Overview:**  
Sends the first Google Places API Text Search request using the user query to fetch initial results and a next_page_token if more results exist.

**Nodes Involved:**  
- Text Search Page 1  
- Has Next Page? (Page 2)  
- Wait 5s (1)

**Node Details:**  
- **Text Search Page 1**  
  - Type: HTTP Request  
  - Role: Queries Google Places Text Search API with the user’s query string and API key.  
  - Configuration:  
    - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
    - Query parameters: `query={{ $json.Query }}`, `key=<YOUR KEY>`  
  - Input: Triggered by form submission.  
  - Output: JSON with results array and optionally `next_page_token`.  
  - Potential failures: Invalid API key, quota exceeded, malformed query, network issues.  
  - Version: 4.2

- **Has Next Page? (Page 2)**  
  - Type: If Condition  
  - Role: Checks if `next_page_token` exists in the response to determine if a second page is available.  
  - Configuration: Condition tests existence of `$json.next_page_token`.  
  - Input: Output from Text Search Page 1.  
  - Output: If true, proceeds to wait node; else, continues processing.  
  - Potential failures: Missing or malformed token, expression errors.  
  - Version: 2.2

- **Wait 5s (1)**  
  - Type: Wait  
  - Role: Pauses workflow for 5 seconds to allow Google API to activate `next_page_token` (Google API requirement).  
  - Configuration: Fixed 5-second delay.  
  - Input: Triggered if next page exists.  
  - Output: Proceeds to fetch second page.  
  - Potential failures: Delay too short causing token invalidity; network timeout rare but possible.  
  - Version: 1.1

---

#### 2.3 Pagination Handling

**Overview:**  
Fetches second and third pages of results if available, respecting API token activation delays.

**Nodes Involved:**  
- Text Search Page 2  
- Has Next Page? (Page 3)  
- Wait 5s (2)  
- Text Search Page 3

**Node Details:**  
- **Text Search Page 2**  
  - Type: HTTP Request  
  - Role: Retrieves second page of results using the `next_page_token` from Page 1.  
  - Configuration:  
    - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
    - Query parameters: `pagetoken={{ $json.next_page_token }}`, `key=<YOUR KEY>`  
  - Input: From Wait 5s (1).  
  - Output: JSON with results and optional next_page_token.  
  - Potential failures: Token not yet active (too short wait), invalid token, quota errors.  
  - Version: 4.2

- **Has Next Page? (Page 3)**  
  - Type: If Condition  
  - Role: Checks if second page response contains a `next_page_token` for a possible third page.  
  - Configuration: Similar to Page 2 checker, tests for existence of `$json.next_page_token`.  
  - Input: Output from Text Search Page 2.  
  - Output: If true, proceeds to Wait 5s (2); else continues.  
  - Potential failures: Missing token, expression errors.  
  - Version: 2.2

- **Wait 5s (2)**  
  - Type: Wait  
  - Role: 5-second delay before fetching third page to ensure token activation.  
  - Configuration: Fixed 5-second delay.  
  - Input: Triggered if third page is available.  
  - Output: Proceeds to Text Search Page 3.  
  - Potential failures: Same as previous wait node.  
  - Version: 1.1

- **Text Search Page 3**  
  - Type: HTTP Request  
  - Role: Retrieves third page of results using the `next_page_token` from Page 2.  
  - Configuration: Same as Text Search Page 2, but uses token from second page.  
  - Input: From Wait 5s (2).  
  - Output: JSON with results; no further paging supported here.  
  - Potential failures: Token timing issues, quota limits.  
  - Version: 4.2

---

#### 2.4 Data Consolidation

**Overview:**  
Merges all pages of retrieved results into one unified dataset for further processing.

**Nodes Involved:**  
- Merge All Pages

**Node Details:**  
- **Merge All Pages**  
  - Type: Merge  
  - Role: Combines results arrays from up to three page requests into a single output array.  
  - Configuration: Configured to accept 3 inputs (from Text Search Page 1, 2, and 3).  
  - Input: Outputs from the three text search nodes.  
  - Output: Single merged dataset containing all results.  
  - Potential failures: Missing inputs if fewer than 3 pages; merge conflicts if data formats differ.  
  - Version: 3.2

---

#### 2.5 Detailed Data Enrichment

**Overview:**  
Splits merged results into individual place entries and fetches detailed information for each place.

**Nodes Involved:**  
- Split out the Results  
- Get Place Details

**Node Details:**  
- **Split out the Results**  
  - Type: Split Out  
  - Role: Separates the merged results array into individual JSON objects, each representing one place.  
  - Configuration: Splits on the "results" field of the merged JSON.  
  - Input: From Merge All Pages.  
  - Output: Individual place JSON objects containing `place_id`.  
  - Potential failures: Missing or malformed results field.  
  - Version: 1

- **Get Place Details**  
  - Type: HTTP Request  
  - Role: Retrieves detailed information (name, phone, website, address) for each place using its `place_id`.  
  - Configuration:  
    - URL: `https://maps.googleapis.com/maps/api/place/details/json`  
    - Query parameters:  
      - `place_id={{ $json.place_id }}`  
      - `fields=name,formatted_phone_number,website,formatted_address`  
      - `key=<YOUR KEY>`  
  - Input: From Split out the Results, one place per execution.  
  - Output: JSON with detailed place info under `result`.  
  - Potential failures: Rate limiting by Google API, invalid place_id, missing fields in response.  
  - Version: 4.2

---

#### 2.6 Data Formatting and Storage

**Overview:**  
Formats the detailed place data to match Google Sheets columns and appends the data rows to a configured Google Sheet.

**Nodes Involved:**  
- Format the Output for Google Sheets  
- Append rows in the Google Sheet

**Node Details:**  
- **Format the Output for Google Sheets**  
  - Type: Code  
  - Role: Transforms raw place detail JSON into a simplified format compatible with Google Sheets columns.  
  - Configuration: JavaScript code extracts `name`, `formatted_phone_number`, `website`, `formatted_address` from each input, returns JSON with `companyName`, `number`, `address`, and `website` keys.  
  - Input: From Get Place Details (array of place details).  
  - Output: Reformatted JSON objects for spreadsheet insertion.  
  - Potential failures: Code errors if expected fields missing; null handling is present to avoid crashes.  
  - Version: 2

- **Append rows in the Google Sheet**  
  - Type: Google Sheets  
  - Role: Appends formatted lead data rows into a specified Google Sheets document and worksheet.  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet name configured to user’s target Google Sheet  
    - Columns defined: Company Name, Website, Phone Number, Email, Email Sent?, 1st Follow Up Sent?, 2nd Follow Up Sent?, Replied? (some columns may be empty initially)  
    - Credentials: Google Sheets OAuth2 with edit access.  
  - Input: From Format the Output for Google Sheets.  
  - Output: Confirmation of rows appended.  
  - Potential failures: OAuth permission errors, network issues, sheet not found, column mismatch.  
  - Version: 4.7

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                             | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                                             |
|---------------------------|--------------------|---------------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Submit Search Query        | Form Trigger       | User input collection of industry/location | -                             | Text Search Page 1              | ## Form Trigger                                                                                                                         |
| Text Search Page 1         | HTTP Request       | Initial Google Places Text Search call      | Submit Search Query            | Has Next Page? (Page 2), Merge All Pages | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Has Next Page? (Page 2)    | If Condition       | Checks for existence of next page token     | Text Search Page 1            | Wait 5s (1)                    | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Wait 5s (1)               | Wait                | Wait 5 seconds before requesting page 2    | Has Next Page? (Page 2)        | Text Search Page 2             | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Text Search Page 2         | HTTP Request       | Google Places API call for page 2           | Wait 5s (1)                   | Has Next Page? (Page 3), Merge All Pages | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Has Next Page? (Page 3)    | If Condition       | Checks for third page next_page_token       | Text Search Page 2            | Wait 5s (2)                   | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Wait 5s (2)               | Wait                | Wait 5 seconds before requesting page 3    | Has Next Page? (Page 3)        | Text Search Page 3             | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Text Search Page 3         | HTTP Request       | Google Places API call for page 3           | Wait 5s (2)                   | Merge All Pages                | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Merge All Pages            | Merge              | Combines results from all search pages      | Text Search Page 1, 2, 3      | Split out the Results          | ## Find and merge results                                                                                                               |
| Split out the Results      | Split Out          | Splits merged results into individual places| Merge All Pages               | Get Place Details              | ## Split out results and find places' details                                                                                           |
| Get Place Details          | HTTP Request       | Fetches detailed place info by place_id     | Split out the Results         | Format the Output for Google Sheets | ## Split out results and find places' details                                                                                           |
| Format the Output for Google Sheets | Code         | Formats data for Google Sheets insertion    | Get Place Details             | Append rows in the Google Sheet | ## Format output and upload to Google Sheet                                                                                            |
| Append rows in the Google Sheet | Google Sheets  | Appends formatted lead data to Google Sheet | Format the Output for Google Sheets | -                           | ## Format output and upload to Google Sheet                                                                                            |
| Sticky Note                | Sticky Note        | Workflow comment                             | -                             | -                             | ## Find and merge results                                                                                                               |
| Sticky Note1               | Sticky Note        | Workflow comment                             | -                             | -                             | ## Split out results and find places' details                                                                                           |
| Sticky Note2               | Sticky Note        | Workflow comment                             | -                             | -                             | ## Format output and upload to Google Sheet                                                                                            |
| Sticky Note3               | Sticky Note        | Workflow comment                             | -                             | -                             | ## Check if Google Maps next page exists (Wait nodes for API call quota)                                                                |
| Sticky Note4               | Sticky Note        | Workflow comment                             | -                             | -                             | ## Form Trigger                                                                                                                         |
| Sticky Note5               | Sticky Note        | Workflow overview, setup, use cases, tips   | -                             | -                             | ## How it works... (full multi-point explanation, setup steps, customization, use cases, troubleshooting tips - see node content above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: Form Trigger  
   - Name: Submit Search Query  
   - Configure a form with a required single field:  
     - Field Label: "Query"  
     - Placeholder: "Ex. Accountants London"  
   - This is the workflow entry point.

2. **Create HTTP Request node for first search page:**  
   - Type: HTTP Request  
   - Name: Text Search Page 1  
   - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
   - Method: GET  
   - Query Parameters:  
     - `query`: Expression `{{$json.Query}}` (user input)  
     - `key`: Your Google Places API key (replace `<YOUR KEY>`)  
   - Connect: Submit Search Query → Text Search Page 1

3. **Create If node to check for next page token from first page:**  
   - Type: If  
   - Name: Has Next Page? (Page 2)  
   - Condition: Check if `next_page_token` exists in JSON response (`{{$json.next_page_token}}`)  
   - Connect: Text Search Page 1 → Has Next Page? (Page 2)

4. **Create Wait node for 5 seconds before fetching page 2:**  
   - Type: Wait  
   - Name: Wait 5s (1)  
   - Duration: 5 seconds  
   - Connect: Has Next Page? (Page 2) [True] → Wait 5s (1)

5. **Create HTTP Request node for second search page:**  
   - Type: HTTP Request  
   - Name: Text Search Page 2  
   - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
   - Method: GET  
   - Query Parameters:  
     - `pagetoken`: Expression `{{$json.next_page_token}}` from previous page  
     - `key`: Your API key  
   - Connect: Wait 5s (1) → Text Search Page 2

6. **Create If node to check for next page token from second page:**  
   - Type: If  
   - Name: Has Next Page? (Page 3)  
   - Condition: Check if `next_page_token` exists in JSON response  
   - Connect: Text Search Page 2 → Has Next Page? (Page 3)

7. **Create Wait node for 5 seconds before fetching page 3:**  
   - Type: Wait  
   - Name: Wait 5s (2)  
   - Duration: 5 seconds  
   - Connect: Has Next Page? (Page 3) [True] → Wait 5s (2)

8. **Create HTTP Request node for third search page:**  
   - Type: HTTP Request  
   - Name: Text Search Page 3  
   - URL and parameters same as Text Search Page 2 but uses token from Page 2  
   - Connect: Wait 5s (2) → Text Search Page 3

9. **Create Merge node to combine all pages:**  
   - Type: Merge  
   - Name: Merge All Pages  
   - Set number of inputs to 3  
   - Connect:  
     - Text Search Page 1 → Merge All Pages (input 1)  
     - Text Search Page 2 → Merge All Pages (input 2)  
     - Text Search Page 3 → Merge All Pages (input 3)

10. **Create Split Out node to separate merged results:**  
    - Type: Split Out  
    - Name: Split out the Results  
    - Field to split: `results`  
    - Connect: Merge All Pages → Split out the Results

11. **Create HTTP Request node to get place details:**  
    - Type: HTTP Request  
    - Name: Get Place Details  
    - URL: `https://maps.googleapis.com/maps/api/place/details/json`  
    - Method: GET  
    - Query Parameters:  
      - `place_id`: Expression `{{$json.place_id}}`  
      - `fields`: `name,formatted_phone_number,website,formatted_address`  
      - `key`: Your API key  
    - Connect: Split out the Results → Get Place Details

12. **Create Code node to format output for Google Sheets:**  
    - Type: Code  
    - Name: Format the Output for Google Sheets  
    - Language: JavaScript  
    - Code:  
      ```js
      return $input.all().map(item => {
        const data = item.json.result || {};
        return {
          json: {
            companyName: data.name || null,
            number: data.formatted_phone_number || null,
            address: data.formatted_address || null,
            website: data.website || null
          }
        };
      });
      ```  
    - Connect: Get Place Details → Format the Output for Google Sheets

13. **Create Google Sheets node to append rows:**  
    - Type: Google Sheets  
    - Name: Append rows in the Google Sheet  
    - Operation: Append  
    - Document ID: Your Google Sheet document ID  
    - Sheet Name: Name or ID of target sheet (e.g., "Results")  
    - Columns: Define columns matching keys from code node (`Company Name`, `Phone Number`, `Website`, `Address`) plus optional columns for email/outreach status  
    - Credentials: Setup Google Sheets OAuth2 with edit permissions  
    - Connect: Format the Output for Google Sheets → Append rows in the Google Sheet

14. **Final checks:**  
    - Replace all `<YOUR KEY>` placeholders with your valid Google Places API key.  
    - Ensure Google Cloud project billing is enabled and Places API activated.  
    - Verify Google Sheets credentials and sheet access.  
    - Test with a small query and verify data flows correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| How it works: 1) Form trigger accepts query; 2) Text Search Page 1 fetches results and next page token; 3) Conditional + Wait nodes handle pagination; 4) Merge all pages; 5) Split results and fetch place details; 6) Format and append to Google Sheets.                                                                                                                                 | Detailed workflow explanation in Sticky Note5 node.                                               |
| Setup steps include enabling billing and Places API in Google Cloud, creating API key, setting up Google Sheets OAuth credentials, preparing sheet columns, and updating nodes with correct IDs and keys.                                                                                                                                                                              | See Sticky Note5 in workflow for comprehensive setup instructions.                                 |
| Customization ideas: Adjust Place Details fields to fetch more info, modify number of pages retrieved, change output format or add CRM/email integration post-sheet appending.                                                                                                                                                                                                       | Workflow is flexible for advanced use cases.                                                       |
| Troubleshooting tips: Confirm API key validity and restrictions; wait times for next_page_token activation; check quota and billing; handle missing data gracefully; consider deduplication to avoid duplicates in sheets; throttle requests to avoid rate limits; ensure OAuth permissions are correctly set.                                                                             | Guidance included in Sticky Note5.                                                                 |
| Use cases include B2B lead generation, outreach list building, CRM enrichment, and market mapping.                                                                                                                                                                                                                                                                                    | Business applications for this workflow.                                                           |
| Google Places API documentation: https://developers.google.com/maps/documentation/places/web-service/overview                                                                                                                                                                                                                                                                       | Official API docs for deeper reference.                                                            |
| Google Sheets API documentation: https://developers.google.com/sheets/api                                                                                                                                                                                                                                                                                                            | For configuring and troubleshooting Sheets node.                                                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.