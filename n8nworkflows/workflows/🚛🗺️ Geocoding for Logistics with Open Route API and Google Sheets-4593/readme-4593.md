🚛🗺️ Geocoding for Logistics with Open Route API and Google Sheets

https://n8nworkflows.xyz/workflows/------geocoding-for-logistics-with-open-route-api-and-google-sheets-4593


# 🚛🗺️ Geocoding for Logistics with Open Route API and Google Sheets

---

### 1. Workflow Overview

This workflow automates geocoding logistics addresses by integrating Google Sheets with the Open Route Service API. It is designed to take a list of addresses (including country codes) from a Google Sheet, query the Open Route Service API to obtain geographical coordinates and related location data, and then update the same Google Sheet with the enriched geodata.

**Target Use Cases:**  
- Logistics companies needing to geocode multiple delivery or pickup addresses efficiently.  
- Data enrichment workflows where location details such as borough, neighborhood, and local administrative area are required.  
- Automated batch processing of address data stored in Google Sheets.

**Logical Blocks:**

- **1.1 Input Reception:** Triggering the workflow manually and loading addresses from Google Sheets.  
- **1.2 Batch Processing Loop:** Iterating over each address in batches to avoid API rate limits or timeouts.  
- **1.3 Open Route API Query:** Sending requests to the Open Route Service geocoding endpoint for each address.  
- **1.4 Data Extraction:** Parsing the API response to extract coordinates and location details.  
- **1.5 Data Persistence:** Writing the enriched data back to the Google Sheet row by row, with a timed delay to handle rate limits.  
- **1.6 Workflow Control:** Looping back after a wait to process the next batch until all addresses are processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The workflow begins manually and pulls all address records from a Google Sheet. It expects columns including country code and address, with placeholders for geocoding outputs.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Collect Addresses (Google Sheets Read)  
  - Sticky Note2 (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Configuration: No parameters, simple manual start.  
    - Inputs: None  
    - Outputs: Connects to "Collect Addresses" node.  
    - Edge Cases: None significant; user must trigger.  

  - **Collect Addresses**  
    - Type: Google Sheets (Read)  
    - Role: Reads the sheet containing the addresses and country codes.  
    - Configuration:  
      - Uses Google Sheets API credentials (OAuth2 or API key).  
      - Selects the specific Google Sheet and worksheet by ID or URL.  
      - Retrieves all rows for processing.  
    - Key Expressions: None directly in parameters; data returned drives subsequent nodes.  
    - Inputs: Trigger node.  
    - Outputs: Data array of addresses with expected fields: country, address, longitude, latitude, borough, neighbourhood, localadmin.  
    - Failure Types: Authentication errors, missing file/sheet, API quota exceeded.  

  - **Sticky Note2**  
    - Type: Sticky Note (Documentation)  
    - Role: Provides detailed setup instructions for the Google Sheet node and required columns.  
    - Notes: Emphasizes leaving geocoding output fields empty for update later.  

#### 2.2 Batch Processing Loop

- **Overview:**  
  To handle potentially large datasets and API limits, the workflow splits the address data into batches processed one at a time.

- **Nodes Involved:**  
  - Loop Over Addresses (Split In Batches)  
  - Sticky Note (Instructional for API and sheet update)

- **Node Details:**

  - **Loop Over Addresses**  
    - Type: SplitInBatches  
    - Role: Divides the list of addresses into smaller portions for sequential processing.  
    - Configuration: Default batch size (implied).  
    - Inputs: Output from "Collect Addresses".  
    - Outputs: Two outputs: main batch output to "Query Open Route API", second output unused (empty).  
    - Edge Cases: Batch size too large causing timeouts; empty input data handling.  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Guides setup for Open Route API credentials and Google Sheets update node configuration. Includes link to Open Route API docs.  

#### 2.3 Open Route API Query

- **Overview:**  
  This block queries the Open Route Service geocoding API for each address in the current batch, sending country and address data as parameters.

- **Nodes Involved:**  
  - Query Open Route API (HTTP Request)

- **Node Details:**

  - **Query Open Route API**  
    - Type: HTTP Request  
    - Role: Makes a GET request to Open Route Service geocoding endpoint.  
    - Configuration:  
      - URL: `https://api.openrouteservice.org/geocode/search`  
      - Query Parameters:  
        - `api_key`: Set in credentials (not shown directly, but required).  
        - `text`: Address from current item (`={{ $json.address }}`).  
        - `boundary.country`: Country code from current item (`={{ $json.country }}`).  
        - `sources`: Fixed as "openstreetmap".  
        - `size`: 1 (only top result).  
      - Headers: Accepts multiple response formats, JSON content type.  
    - Inputs: Batch item from "Loop Over Addresses".  
    - Outputs: API response JSON object with geocoding features.  
    - Failure Types: API key invalid or missing, rate limiting, network errors, malformed requests, empty or ambiguous addresses.  

#### 2.4 Data Extraction

- **Overview:**  
  Extracts relevant geolocation fields from the API JSON response such as longitude, latitude, borough, neighborhood, and local administrative area.

- **Nodes Involved:**  
  - Extract Results (Set node)

- **Node Details:**

  - **Extract Results**  
    - Type: Set  
    - Role: Maps deeply nested JSON properties from the API response into flat key-value pairs for later sheet update.  
    - Configuration:  
      - Uses expressions to extract:  
        - longitude: `{{$json.features[0].geometry.coordinates[0]}}`  
        - latitude: `{{$json.features[0].geometry.coordinates[1]}}`  
        - borough: `{{$json.features[0].properties.borough}}`  
        - neighbourhood: `{{$json.features[0].properties.neighbourhood}}`  
        - localadmin: `{{$json.features[0].properties.localadmin}}`  
      - Assumes the first feature is the best match.  
    - Inputs: API response.  
    - Outputs: Enriched JSON ready for sheet update.  
    - Edge Cases: No features returned, missing properties, malformed response causing expression failures.  

#### 2.5 Data Persistence and Rate Limiting

- **Overview:**  
  Updates the Google Sheet row corresponding to the current address with the extracted geolocation data, then waits 5 seconds before processing the next batch to respect API rate limits.

- **Nodes Involved:**  
  - Save Results (Google Sheets Update)  
  - 5 sec (Wait)  

- **Node Details:**

  - **Save Results**  
    - Type: Google Sheets (Update)  
    - Role: Updates existing rows with geocoding results.  
    - Configuration:  
      - Matches rows by unique `id` field from the original row.  
      - Updates columns: longitude, latitude, borough, neighbourhood, localadmin.  
      - Uses defined mapping mode with explicit columns.  
      - Sheet and document ID same as "Collect Addresses".  
    - Inputs: Processed geocoding data from "Extract Results".  
    - Outputs: Connects to "5 sec" wait node.  
    - Failure Types: Authentication, sheet locked or missing rows, concurrency conflicts.  

  - **5 sec**  
    - Type: Wait  
    - Role: Pauses workflow for 5 seconds to mitigate API rate limit issues.  
    - Configuration: Fixed 5-second wait.  
    - Inputs: From "Save Results".  
    - Outputs: Loops back to "Loop Over Addresses" to process next batch.  

#### 2.6 Workflow Control (Looping)

- **Overview:**  
  After waiting 5 seconds, the workflow loops back to process the next batch of addresses until all are processed.

- **Nodes Involved:**  
  - Loop Over Addresses (main output 0 for next batch)

- **Node Details:**

  - This looping logic is implicit in the connections: after waiting, the workflow resumes the next batch from the split node.  
  - Ensures controlled sequential batch processing avoiding API overload.  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                               |
|-------------------------|--------------------|---------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts the workflow manually           | -                        | Collect Addresses       |                                                                                                                                           |
| Collect Addresses       | Google Sheets      | Reads addresses and country codes      | When clicking ‘Test workflow’ | Loop Over Addresses     | Sticky Note2: Instructions to setup Google Sheets node with fields for country, address, and empty geocoding columns                       |
| Loop Over Addresses     | SplitInBatches     | Splits address list into batches       | Collect Addresses         | Query Open Route API (main output 1), empty output 0 | Sticky Note: Setup instructions for Open Route API key and Google Sheets update node                                                        |
| Query Open Route API    | HTTP Request       | Queries Open Route Service geocode API | Loop Over Addresses (batch item) | Extract Results         |                                                                                                                                           |
| Extract Results         | Set                | Extracts coordinates and location info | Query Open Route API      | Save Results            |                                                                                                                                           |
| Save Results            | Google Sheets      | Updates Google Sheet with geocoded data| Extract Results           | 5 sec                   | Sticky Note: See Google Sheets setup instructions in previous sticky note                                                                 |
| 5 sec                   | Wait               | Waits 5 seconds before next batch      | Save Results              | Loop Over Addresses     |                                                                                                                                           |
| Sticky Note2            | Sticky Note        | Instruction for initial Google Sheets setup | -                        | -                       | Instructions for Google Sheets node setup, mapping fields, and leaving geocoding fields empty                                               |
| Sticky Note             | Sticky Note        | Setup instructions for Open Route API and sheet update | -                        | -                       | Instructions for API key setup and Google Sheet update configuration                                                                       |
| Sticky Note4            | Sticky Note        | Links to video tutorial and thumbnail  | -                        | -                       | [Check the Tutorial](https://www.youtube.com/watch?v=IlblIlKcL0k) ![Thumbnail](https://www.samirsaci.com/content/images/2025/06/image-10.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’". No parameters needed.  

2. **Create Google Sheets Node to Read Addresses:**  
   - Add "Google Sheets" node named "Collect Addresses".  
   - Configure credentials with your Google API OAuth2.  
   - Select your target spreadsheet by URL, ID, or list selection.  
   - Choose the worksheet containing the address data.  
   - Ensure columns include: `id`, `country`, `address`, `longitude`, `latitude`, `borough`, `neighbourhood`, `localadmin`.  
   - This node reads all rows to process.  

3. **Create Split In Batches Node:**  
   - Add "SplitInBatches" node named "Loop Over Addresses".  
   - Connect "Collect Addresses" output to this node.  
   - Default batch size is usually 1 or configurable (adjust according to API limits).  

4. **Create HTTP Request Node for Open Route API:**  
   - Add "HTTP Request" node named "Query Open Route API".  
   - Set method to GET.  
   - URL: `https://api.openrouteservice.org/geocode/search`.  
   - Add Query Parameters:  
     - `api_key`: Use an expression referencing stored credentials or set manually.  
     - `text`: Expression `={{ $json.address }}` to use address from current batch item.  
     - `boundary.country`: Expression `={{ $json.country }}`.  
     - `sources`: fixed to "openstreetmap".  
     - `size`: fixed to 1.  
   - Add headers:  
     - `Content-Type`: `application/json; charset=utf-8`  
     - `Accept`: `application/json, application/geo+json, application/gpx+xml, img/png; charset=utf-8`  
   - Connect "Loop Over Addresses" second output (batch items) to this node.  

5. **Create Set Node to Extract Results:**  
   - Add "Set" node named "Extract Results".  
   - Add fields with expressions extracting from API response:  
     - `longitude`: `{{$json.features[0].geometry.coordinates[0]}}`  
     - `latitude`: `{{$json.features[0].geometry.coordinates[1]}}`  
     - `borough`: `{{$json.features[0].properties.borough}}`  
     - `neighbourhood`: `{{$json.features[0].properties.neighbourhood}}`  
     - `localadmin`: `{{$json.features[0].properties.localadmin}}`  
   - Connect output of "Query Open Route API" to this node.  

6. **Create Google Sheets Node to Update Rows:**  
   - Add "Google Sheets" node named "Save Results".  
   - Use the same credentials and spreadsheet as "Collect Addresses".  
   - Select the same worksheet.  
   - Set operation to "Update".  
   - Map columns to update:  
     - Use `id` for row matching (`={{ $('Loop Over Addresses').item.json.id }}`).  
     - Update `longitude`, `latitude`, `borough`, `neighbourhood`, `localadmin` from values set in "Extract Results".  
   - Connect "Extract Results" output to this node.  

7. **Create Wait Node to Handle Rate Limiting:**  
   - Add "Wait" node named "5 sec".  
   - Set wait time to 5 seconds.  
   - Connect "Save Results" output to this node.  

8. **Loop Back to Batch Node:**  
   - Connect "5 sec" node output back to "Loop Over Addresses" main input to continue processing the next batch.  

9. **Add Sticky Notes (Optional):**  
   - Add sticky notes to document setup instructions for Google Sheets node and Open Route API key, referencing official documentation and tutorials.  

**Credentials Setup:**  
- Google Sheets API credentials (OAuth2 recommended).  
- Open Route Service API key (free registration at https://openrouteservice.org/dev/#/api-docs). Store securely in n8n credentials.  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Detailed instructions for Google Sheets node setup and field mapping must be followed carefully.| See Sticky Note2 in workflow. [Google Sheets Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Open Route API registration and key setup required before using the HTTP Request node.           | [Open Route API Documentation](https://openrouteservice.org/dev/#/api-docs)                         |
| Workflow designed to handle API rate limits by batching and a 5-second delay between calls.      | Helps avoid request throttling and timeouts.                                                      |
| Tutorial video with detailed explanation available.                                              | [YouTube Tutorial](https://www.youtube.com/watch?v=IlblIlKcL0k)                                   |
| Thumbnail image for tutorial included in Sticky Note4.                                           | ![Thumbnail](https://www.samirsaci.com/content/images/2025/06/image-10.png)                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---