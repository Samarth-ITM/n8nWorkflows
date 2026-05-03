Convert Addresses to LatLong with Google Sheets and Google Maps API

https://n8nworkflows.xyz/workflows/convert-addresses-to-latlong-with-google-sheets-and-google-maps-api-2739


# Convert Addresses to LatLong with Google Sheets and Google Maps API

### 1. Workflow Overview

This workflow automates the conversion of physical addresses into geographic coordinates (latitude and longitude) by integrating Google Sheets with the Google Maps API. It is designed for users who maintain address lists in Google Sheets and require accurate geolocation data for applications such as delivery routing, event planning, or market analysis.

The workflow is logically divided into four main blocks:

- **1.1 Trigger**: Initiates the workflow manually.
- **1.2 Retrieve Address Data**: Reads address entries from a specified Google Sheet.
- **1.3 Geolocation via Google Maps API**: Sends each address to the Google Maps API to obtain latitude and longitude.
- **1.4 Update Google Sheet**: Writes the retrieved LatLong data back into the original Google Sheet, aligned with each address.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger

- **Overview:**  
  This block starts the workflow manually, allowing the user to control when the address conversion process runs.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Test workflow’  
  - **Type:** Manual Trigger  
  - **Role:** Initiates the workflow execution on user command.  
  - **Configuration:** Default manual trigger with no parameters.  
  - **Expressions/Variables:** None.  
  - **Input:** None (start node).  
  - **Output:** Connects to "Extract The Places from Google Sheet".  
  - **Version Requirements:** Compatible with n8n version 1+.  
  - **Potential Failures:** None expected unless n8n is offline or user cancels execution.  
  - **Sub-workflow:** None.

---

#### 1.2 Retrieve Address Data

- **Overview:**  
  This block fetches the list of addresses from a specified Google Sheet, preparing them for geolocation processing.

- **Nodes Involved:**  
  - Extract The Places from Google Sheet

- **Node Details:**  
  - **Node Name:** Extract The Places from Google Sheet  
  - **Type:** Google Sheets (Read Operation)  
  - **Role:** Reads rows from a specific sheet containing addresses.  
  - **Configuration:**  
    - Document ID: `15Fz57qiARIJ6R5OzBAVgiAHnU8sOSX8eYFEP6thOrMM` (Google Sheet ID)  
    - Sheet Name: Sheet with ID `1738976351` (likely "Sheet2")  
    - Operation: Read all rows (default options)  
  - **Expressions/Variables:** None explicitly; outputs each row with its data and row number.  
  - **Input:** From manual trigger node.  
  - **Output:** Passes each row (address data) to the Google Maps API node.  
  - **Version Requirements:** Requires Google Sheets OAuth2 credentials configured.  
  - **Potential Failures:**  
    - Authentication errors if OAuth2 token is invalid or expired.  
    - Sheet or document ID incorrect or inaccessible.  
    - Empty or malformed address data.  
  - **Sub-workflow:** None.

---

#### 1.3 Geolocation via Google Maps API

- **Overview:**  
  This block sends each address to the Google Maps Places Text Search API to retrieve geographic coordinates.

- **Nodes Involved:**  
  - Using Google Map API to Return Lat Long Back

- **Node Details:**  
  - **Node Name:** Using Google Map API to Return Lat Long Back  
  - **Type:** HTTP Request  
  - **Role:** Queries Google Maps Places API with the address to get location data.  
  - **Configuration:**  
    - HTTP Method: GET (default)  
    - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
    - Query Parameters:  
      - `query`: Address string dynamically set as `={{ $json.Address }}`  
      - `key`: API key hardcoded as `AIzaSyCwQkEAOhqxxyXygUri-6xhzFSQuidA5TM` (should be replaced with user’s own key)  
  - **Expressions/Variables:** Uses the current item’s `Address` field for the query parameter.  
  - **Input:** Receives each address row from Google Sheets node.  
  - **Output:** JSON response from Google Maps API containing location data.  
  - **Version Requirements:** HTTP Request node version 4.2 or higher recommended.  
  - **Potential Failures:**  
    - API key invalid, missing, or quota exceeded.  
    - Address not found or ambiguous results.  
    - Network timeouts or API rate limiting.  
    - JSON parsing errors if API response changes.  
  - **Sub-workflow:** None.

---

#### 1.4 Update Google Sheet

- **Overview:**  
  This block updates the original Google Sheet by writing the latitude and longitude coordinates back into the corresponding rows.

- **Nodes Involved:**  
  - Update Lat-Long in Each Places

- **Node Details:**  
  - **Node Name:** Update Lat-Long in Each Places  
  - **Type:** Google Sheets (Update Operation)  
  - **Role:** Updates the `Latlong` column for each address row with the retrieved coordinates.  
  - **Configuration:**  
    - Document ID: Same as read node (`15Fz57qiARIJ6R5OzBAVgiAHnU8sOSX8eYFEP6thOrMM`)  
    - Sheet Name: Same sheet (`1738976351`)  
    - Operation: Update row  
    - Matching Column: `row_number` (to ensure the correct row is updated)  
    - Columns to update:  
      - `Latlong`: Set to a string combining latitude and longitude extracted from API response:  
        `={{ $json.results[0].geometry.location.lat }},{{ $json.results[0].geometry.location.lng }}`  
      - `row_number`: Passed through from the original Google Sheets data to identify the row.  
  - **Expressions/Variables:** Uses expressions to extract lat/lng from API JSON and to match row numbers.  
  - **Input:** Receives API response with location data and original row number.  
  - **Output:** None (end of workflow).  
  - **Version Requirements:** Google Sheets node version 4.5 or higher recommended.  
  - **Potential Failures:**  
    - Authentication errors with Google Sheets OAuth2.  
    - Row number mismatch or missing causing update failures.  
    - API response missing expected fields (e.g., no results).  
    - Google Sheets API quota limits.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                      | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                      |
|----------------------------------|---------------------|------------------------------------|-------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger      | Starts the workflow manually        | None                          | Extract The Places from Google Sheet |                                                                                                 |
| Extract The Places from Google Sheet | Google Sheets       | Reads addresses from Google Sheet   | When clicking ‘Test workflow’ | Using Google Map API to Return Lat Long Back |                                                                                                 |
| Using Google Map API to Return Lat Long Back | HTTP Request       | Queries Google Maps API for LatLong | Extract The Places from Google Sheet | Update Lat-Long in Each Places      |                                                                                                 |
| Update Lat-Long in Each Places    | Google Sheets       | Updates Google Sheet with LatLong   | Using Google Map API to Return Lat Long Back | None                            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - No parameters needed.

2. **Add Google Sheets Node to Read Addresses**  
   - Add a **Google Sheets** node named "Extract The Places from Google Sheet".  
   - Set operation to **Read** (default).  
   - Configure credentials with your Google Sheets OAuth2 account.  
   - Set **Document ID** to your Google Sheet ID (e.g., `15Fz57qiARIJ6R5OzBAVgiAHnU8sOSX8eYFEP6thOrMM`).  
   - Set **Sheet Name** to the appropriate sheet (e.g., Sheet2 or by sheet ID `1738976351`).  
   - Leave other options as default to read all rows.  
   - Connect the output of the Manual Trigger node to this node.

3. **Add HTTP Request Node for Google Maps API**  
   - Add an **HTTP Request** node named "Using Google Map API to Return Lat Long Back".  
   - Set HTTP Method to **GET**.  
   - Set URL to `https://maps.googleapis.com/maps/api/place/textsearch/json`.  
   - Under **Query Parameters**, add:  
     - `query` with value expression: `={{ $json.Address }}`  
     - `key` with your Google Maps API key (replace the example key).  
   - Connect the output of the Google Sheets read node to this node.

4. **Add Google Sheets Node to Update LatLong**  
   - Add a **Google Sheets** node named "Update Lat-Long in Each Places".  
   - Set operation to **Update**.  
   - Configure credentials with the same Google Sheets OAuth2 account.  
   - Set **Document ID** and **Sheet Name** to match the read node.  
   - Under **Columns**, define:  
     - `Latlong` with expression: `={{ $json.results[0].geometry.location.lat }},{{ $json.results[0].geometry.location.lng }}`  
     - `row_number` with expression: `={{ $('Extract The Places from Google Sheet').item.json.row_number }}`  
   - Set **Matching Columns** to `row_number` to update the correct row.  
   - Connect the output of the HTTP Request node to this node.

5. **Verify Credentials and Permissions**  
   - Ensure Google Sheets OAuth2 credentials have read/write access to the target sheet.  
   - Ensure Google Maps API key has Places API enabled and is unrestricted or properly restricted for your use case.

6. **Test the Workflow**  
   - Save and activate the workflow.  
   - Click "Execute Workflow" manually to start processing.  
   - Verify that the LatLong column in your Google Sheet updates with coordinates.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Google Sheets Template for addresses and LatLong columns                                        | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1VvFne32djbKeo-g5pbpOaFKxU-ps0LyVVKsBbIUjRxk/edit?usp=sharing) |
| Instructions to enable Google Maps API and generate API key                                    | Refer to Google Cloud Console documentation for enabling Places API and creating API keys               |
| Ensure API key security by restricting usage to authorized IPs or referrers                     | Google Cloud Platform best practices                                                                    |
| Workflow designed for batch processing; consider API quota limits and rate limiting             | Google Maps API quotas and billing details                                                              |

---

This documentation provides a comprehensive understanding of the workflow’s structure, node configurations, and operational logic, enabling users and automation agents to reproduce, modify, or troubleshoot the process effectively.