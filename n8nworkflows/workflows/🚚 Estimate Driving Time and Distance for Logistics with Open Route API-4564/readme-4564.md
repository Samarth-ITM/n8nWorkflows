🚚 Estimate Driving Time and Distance for Logistics with Open Route API

https://n8nworkflows.xyz/workflows/---estimate-driving-time-and-distance-for-logistics-with-open-route-api-4564


# 🚚 Estimate Driving Time and Distance for Logistics with Open Route API

### 1. Workflow Overview

This workflow estimates driving time and distance for logistics routes using the Open Route Service API, specifically optimized for truck (heavy goods vehicle) travel. It targets logistics planners who want to automate the retrieval of driving distances and durations for multiple routes stored in a Google Sheet, updating the sheet with the API results.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Manual trigger and loading route data from Google Sheets.
- **1.2 Iterative Processing**: Looping through each route to query the Open Route API.
- **1.3 API Request & Data Extraction**: Sending requests to Open Route API and extracting distance, duration, and step count.
- **1.4 Results Saving & Throttling**: Updating Google Sheets with the results and waiting briefly before processing the next item to avoid rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block starts the workflow manually and loads the routes data (departure and destination coordinates) from a Google Sheet.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Collect Routes (Google Sheets)

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point to start the workflow manually for testing.  
     - Configuration: Default manual trigger (no parameters).  
     - Input: None (manual start).  
     - Output: Sends trigger to "Collect Routes".  
     - Edge Cases: None specific; user must start workflow manually.

  2. **Collect Routes**  
     - Type: Google Sheets  
     - Role: Reads the list of routes (with departure and destination coordinates) from a specified Google Sheet.  
     - Configuration:  
       - Google Sheets OAuth2 credentials required.  
       - Document ID: Set to a specific Google Sheets file containing logistics routes.  
       - Sheet name: “Distance” (gid=0).  
       - Reads columns including city_departure, longitude_departure, latitude_departure, city_destination, longitude_destination, latitude_destination, distance, duration, n_steps.  
     - Input: Trigger from manual node.  
     - Output: Provides a list of route items for looping.  
     - Edge Cases:  
       - Google Sheets API auth errors.  
       - Empty or malformed rows.  
       - Missing coordinate data could cause downstream errors.

#### 1.2 Iterative Processing

- **Overview:**  
  Processes each route individually by looping through them in batches.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches)

- **Node Details:**

  1. **Loop Over Items**  
     - Type: Split In Batches  
     - Role: Iterates over each route item one by one (or batch-wise) to control execution flow.  
     - Configuration: Default batch size (unspecified, defaults to 1).  
     - Input: List of route items from "Collect Routes".  
     - Output: Sends each individual route item to API request node.  
     - Edge Cases:  
       - Large number of routes could cause performance delays.  
       - Improper batching could lead to API rate limit issues.

#### 1.3 API Request & Data Extraction

- **Overview:**  
  For each route, this block calls the Open Route Service API to get driving directions for trucks and extracts distance, duration, and number of steps.

- **Nodes Involved:**  
  - Request Open Route API (HTTP Request)  
  - Extract Results (Set)

- **Node Details:**

  1. **Request Open Route API**  
     - Type: HTTP Request  
     - Role: Sends GET request to Open Route Service Directions API endpoint for heavy goods vehicles (HGV).  
     - Configuration:  
       - URL: `https://api.openrouteservice.org/v2/directions/driving-hgv`  
       - Query Parameters:  
         - `api_key`: User must fill with their Open Route API key.  
         - `start`: Composed of longitude_departure and latitude_departure from current item (format "lon, lat").  
         - `end`: Composed of longitude_destination and latitude_destination (format "lon, lat").  
       - Headers: Content-Type and Accept set for JSON and geo+json content.  
       - Response Format: JSON.  
     - Input: Single route item from loop.  
     - Output: JSON response containing routing information.  
     - Edge Cases:  
       - API key missing or invalid → authentication error.  
       - Coordinates invalid or missing → API returns errors or empty results.  
       - API rate limits or timeouts.  
       - Network connectivity issues.

  2. **Extract Results**  
     - Type: Set Node  
     - Role: Transforms the raw API response JSON to extract key metrics.  
     - Configuration:  
       - Extracts three fields from the JSON response:  
         - `distance` (meters) from `features[0].properties.segments[0].distance`  
         - `duration` (seconds) from `features[0].properties.segments[0].duration`  
         - `n_steps` (number of navigation steps) from `features[0].properties.segments[0].steps.length`  
       - Clears one unused string field with empty value (possibly placeholder).  
     - Input: API response JSON.  
     - Output: Cleaned item with only relevant metrics.  
     - Edge Cases:  
       - If API response structure changes, expressions may fail.  
       - No routes returned → undefined properties, leading to errors.

#### 1.4 Results Saving & Throttling

- **Overview:**  
  Saves the extracted routing results back into the Google Sheet, updating corresponding rows, and waits 5 seconds before processing the next item to prevent API rate limiting.

- **Nodes Involved:**  
  - Save Results (Google Sheets)  
  - 5 sec (Wait)

- **Node Details:**

  1. **Save Results**  
     - Type: Google Sheets  
     - Role: Updates the original Google Sheet with API results: distance, duration, number of steps per route ID.  
     - Configuration:  
       - Uses "update" operation.  
       - Matches rows by `id` field extracted from original route item.  
       - Writes back fields: `distance`, `duration`, `n_steps`, keeping other fields intact.  
       - Requires Google Sheets OAuth2 credentials.  
       - Document ID and Sheet name must be set correctly (document ID not filled in JSON, user must specify).  
     - Input: Extracted results with route `id`.  
     - Output: Confirmation of update; triggers wait node.  
     - Edge Cases:  
       - Credentials issues for Google Sheets.  
       - Missing `id` field breaks matching.  
       - Concurrent updates could cause conflicts.

  2. **5 sec**  
     - Type: Wait  
     - Role: Pauses execution for 5 seconds after each route update to avoid hitting API rate limits.  
     - Configuration: Default 5 seconds.  
     - Input: After saving results.  
     - Output: Loops back to "Loop Over Items" for next iteration.  
     - Edge Cases: None significant; ensures pacing.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                        | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                          |
|-------------------------|-----------------------|-------------------------------------|---------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger        | Starts workflow manually              | -                         | Collect Routes           |                                                                                                     |
| Collect Routes          | Google Sheets         | Reads logistics routes from sheet    | When clicking ‘Test workflow’ | Loop Over Items          | - Setup API Credentials (Open Route API key) needed. <br> - Fields to map: city_departure, longitude_departure, latitude_departure, city_destination, longitude_destination, latitude_destination, distance, duration, n_steps.<br> [Google Sheet Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Loop Over Items         | Split In Batches      | Loops over each route item            | Collect Routes            | Request Open Route API (batch 2), Loop Over Items (batch 1) |                                                                                                     |
| Request Open Route API  | HTTP Request          | Calls Open Route Service API          | Loop Over Items (batch 2) | Extract Results          | - Fill API key.<br> - API endpoint for truck routes: driving-hgv.<br> [Open Route API Docs](https://openrouteservice.org/dev/#/api-docs) |
| Extract Results         | Set                   | Extracts distance, duration, steps   | Request Open Route API    | Save Results             |                                                                                                     |
| Save Results            | Google Sheets         | Updates sheet with results            | Extract Results           | 5 sec                    | - Requires Google Sheets credentials.<br> - Match by id field.                                       |
| 5 sec                   | Wait                  | Waits 5 seconds to avoid API limits   | Save Results              | Loop Over Items          |                                                                                                     |
| Sticky Note2            | Sticky Note           | Instruction on workflow trigger and Google Sheets setup | -                         | -                        | ### 1. Trigger the workflow<br>This starts by collecting all routes for truck driving time & distance.<br>See detailed setup for Google Sheets fields.<br>[Google Sheet Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Sticky Note             | Sticky Note           | Instructions for looping and API usage | -                         | -                        | ### 2. Loop to collect driving distances and time<br>Setup Open Route API key and mode.<br>Keep distance, duration, n_steps fields empty for API output.<br>[Open Route API Docs](https://openrouteservice.org/dev/#/api-docs) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Manually start the workflow.

2. **Add a Google Sheets node**  
   - Name: `Collect Routes`  
   - Operation: Read rows  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Document ID: Enter your Google Sheet file ID containing routes.  
   - Sheet Name: Set to the sheet containing route data (e.g., "Distance").  
   - Columns to read: `id`, `city_departure`, `longitude_departure`, `latitude_departure`, `city_destination`, `longitude_destination`, `latitude_destination`, `distance`, `duration`, `n_steps`.  
   - Connect output of `When clicking ‘Test workflow’` to this node.

3. **Add a Split In Batches node**  
   - Name: `Loop Over Items`  
   - Purpose: Iterate over each route item one by one.  
   - Default batch size (1) is fine.  
   - Connect output of `Collect Routes` to this node.

4. **Add an HTTP Request node**  
   - Name: `Request Open Route API`  
   - HTTP Method: GET  
   - URL: `https://api.openrouteservice.org/v2/directions/driving-hgv`  
   - Query Parameters:  
     - `api_key`: Your Open Route API key (must be filled).  
     - `start`: Expression `{{$json.longitude_departure}}, {{$json.latitude_departure}}`  
     - `end`: Expression `{{$json.longitude_destination}}, {{$json.latitude_destination}}`  
   - Headers:  
     - Content-Type: `application/json; charset=utf-8`  
     - Accept: `application/json, application/geo+json, application/gpx+xml, img/png; charset=utf-8`  
   - Connect second output (batch 2) of `Loop Over Items` to this node.

5. **Add a Set node**  
   - Name: `Extract Results`  
   - Purpose: Extract key response data from API JSON.  
   - Add assignments:  
     - `distance`: `={{ $json.features[0].properties.segments[0].distance }}`  
     - `duration`: `={{ $json.features[0].properties.segments[0].duration }}`  
     - `n_steps`: `={{ $json.features[0].properties.segments[0].steps.length }}`  
   - Connect output of `Request Open Route API` to this node.

6. **Add another Google Sheets node**  
   - Name: `Save Results`  
   - Operation: Update rows  
   - Credentials: Use Google Sheets OAuth2 credentials.  
   - Document ID: Same as `Collect Routes` or your target sheet.  
   - Sheet Name: Same sheet as before.  
   - Matching Column: `id` (to update correct row).  
   - Fields to update: `distance`, `duration`, `n_steps` (map from current JSON fields).  
   - Connect output of `Extract Results` to this node.

7. **Add a Wait node**  
   - Name: `5 sec`  
   - Duration: 5 seconds  
   - Connect output of `Save Results` to this node.

8. **Loop connection**  
   - Connect output of `5 sec` back to the first output (batch 1) of `Loop Over Items` to continue looping.

9. **Add sticky notes** (optional but recommended)  
   - To describe the workflow steps, setup instructions, API key requirements, and Google Sheets configuration.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Setup API key for Open Route Service from https://openrouteservice.org/dev/#/api-docs            | Open Route API documentation                                                                       |
| Google Sheets Node documentation and setup guide: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets | For configuring Google Sheets credentials and understanding operations                             |
| Workflow optimized for truck routing by using `driving-hgv` profile in Open Route API            | Heavy Goods Vehicle routing is different from car routing, suitable for logistics and trucking    |
| Wait node to avoid API rate limiting                                                             | Prevents hitting Open Route Service API limits by spacing requests                                |

---

This documentation enables comprehensive understanding, reproduction, and modification of the workflow, while highlighting potential failure points and setup requirements for reliable operation.