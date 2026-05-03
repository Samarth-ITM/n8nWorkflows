Track Hourly Weather Conditions with OpenWeatherMap and Google Sheets

https://n8nworkflows.xyz/workflows/track-hourly-weather-conditions-with-openweathermap-and-google-sheets-4066


# Track Hourly Weather Conditions with OpenWeatherMap and Google Sheets

### 1. Workflow Overview

This workflow is designed to track hourly weather conditions in a specified city using OpenWeatherMap's API and log select data to a Google Sheet. It suits use cases such as monitoring weather-sensitive operations, environmental data collection, or learning API-to-spreadsheet integrations.

The logic is organized into these main blocks:

- **1.1 Schedule Trigger:** Activates the workflow automatically every hour.

- **1.2 Weather Data Retrieval:** Fetches current weather data for the target city from OpenWeatherMap.

- **1.3 Condition Evaluation:** Checks if it is raining or if the temperature is below a defined threshold.

- **1.4 Data Formatting:** Extracts relevant weather details and prepares them for logging.

- **1.5 Google Sheets Logging:** Appends the formatted weather data to a designated Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:** Initiates the workflow automatically every hour to ensure periodic weather data collection.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger` (event node)  
    - Configuration: Set to trigger every hour using interval field targeting "hours"  
    - Expressions/Variables: None  
    - Inputs: None (start node)  
    - Outputs: Connects to the HTTP Request node for weather data  
    - Version: 1.2  
    - Edge Cases: Trigger could fail if n8n server is down or time settings misconfigured  
    - Notes: Reliable periodic activation is essential for consistent data logging

#### 2.2 Weather Data Retrieval

- **Overview:** Fetches real-time weather data from OpenWeatherMap API for a fixed city ("Pasay,ph").

- **Nodes Involved:**  
  - Get Weather Data from OpenWeatherMap  
  - Sticky Note (documentation)

- **Node Details:**  
  - **Get Weather Data from OpenWeatherMap**  
    - Type: `HTTP Request`  
    - Configuration:  
      - URL uses an expression to build API call:  
        `https://api.openweathermap.org/data/2.5/weather?q=Pasay,ph&APPID={{ $credentials.openWeatherMap.apiKey }}`  
      - Method defaults to GET  
      - No additional request options set  
    - Expressions: Uses OpenWeatherMap API key from node credentials  
    - Input: From Schedule Trigger  
    - Output: JSON weather data to "If is raining" node  
    - Version: 4.2  
    - Edge Cases:  
      - API key invalid or expired → auth error  
      - City name typo or no data → HTTP 404 or empty response  
      - API rate limits exceeded → HTTP 429 error  
      - Network timeout  
    - Credentials: Uses OpenWeatherMap API key configured in n8n credentials

  - **Sticky Note**  
    - Content: "### Get Weather information and see if it is raining"  
    - Role: Provides contextual documentation for this logical block  
    - Position: Near weather data retrieval and evaluation nodes

#### 2.3 Condition Evaluation

- **Overview:** Evaluates if the weather condition contains "rain" or if the temperature is below 303 K (~30°C), signaling notable weather events.

- **Nodes Involved:**  
  - If is raining

- **Node Details:**  
  - **If is raining**  
    - Type: `If` node  
    - Configuration:  
      - Combinator: OR  
      - Conditions (version 2 syntax):  
        - Temperature less than 303 (Kelvin) → `{{$json.main.temp}} < 303`  
        - Weather description contains "rain" → `{{$json.weather[0].description}}` contains "rain" (case-sensitive)  
    - Input: Weather JSON from HTTP Request  
    - Output:  
      - True branch: Condition met (rain or low temp) → leads to data formatting  
      - False branch: Condition not met → ends workflow (no further action)  
    - Version: 2.2  
    - Edge Cases:  
      - Weather array empty or missing description → expression failure  
      - Case sensitivity may miss some rain-related descriptions if differently cased  
    - Notes: Strict type validation avoids type mismatch errors

#### 2.4 Data Formatting

- **Overview:** Constructs a structured data object with city, temperature, humidity, weather conditions, and a fixed status message to prepare for spreadsheet logging.

- **Nodes Involved:**  
  - Format the data

- **Node Details:**  
  - **Format the data**  
    - Type: `Set` node  
    - Configuration: Assigns new fields:  
      - city: from `{{$json.name}}`  
      - temperature (K): from `{{$json.main.temp}}`  
      - humidity: from `{{$json.main.humidity}}`  
      - conditions: from `{{$json.weather[0].description}}`  
      - status: fixed string "higher than average temperature"  
    - Input: True output from If node  
    - Output: Structured JSON for Google Sheets  
    - Version: 3.4  
    - Edge Cases: Missing or malformed input data → output incomplete or errors  
    - Notes: All fields assigned as strings, which Google Sheets can parse accordingly

#### 2.5 Google Sheets Logging

- **Overview:** Appends the formatted weather data as a new row into the specified Google Sheet using service account authentication.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Google Sheets**  
    - Type: `Google Sheets` node  
    - Operation: Append (adds new rows)  
    - Sheet Name & Document ID: Configured via resource selector (linked to specific Google Sheet)  
    - Authentication: Service Account credential configured in n8n (`Google Service Account training`)  
    - Input: JSON from "Format the data" node  
    - Output: None (end node)  
    - Version: 4.5  
    - Edge Cases:  
      - Credential expiration or permission errors → auth failure  
      - Incorrect document ID or sheet name → append failure  
      - Rate limits on Google Sheets API  
      - Network issues  
    - Notes: Requires prior setup of Google Service Account with correct sheet access permissions

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                      | Input Node(s)                      | Output Node(s)                       | Sticky Note                                |
|----------------------------------|-------------------------|------------------------------------|----------------------------------|------------------------------------|--------------------------------------------|
| Schedule Trigger                 | Schedule Trigger        | Periodic trigger every hour         | —                                | Get Weather Data from OpenWeatherMap |                                            |
| Get Weather Data from OpenWeatherMap | HTTP Request           | Fetches weather data from API       | Schedule Trigger                 | If is raining                      | ### Get Weather information and see if it is raining |
| If is raining                   | If                     | Checks for rain or low temperature  | Get Weather Data from OpenWeatherMap | Format the data (true branch)      | ### Get Weather information and see if it is raining |
| Format the data                 | Set                    | Formats weather data for logging    | If is raining (true branch)      | Google Sheets                     | ### Get Weather information and see if it is raining |
| Google Sheets                  | Google Sheets           | Appends data to Google Sheet        | Format the data                  | —                                  |                                            |
| Sticky Note                    | Sticky Note             | Documentation                      | —                                | —                                  | ### Get Weather information and see if it is raining |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 1 hour  
   - No credentials needed

2. **Add HTTP Request node to fetch weather:**  
   - Name: Get Weather Data from OpenWeatherMap  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL:  
     ```
     https://api.openweathermap.org/data/2.5/weather?q=Pasay,ph&APPID={{ $credentials.openWeatherMap.apiKey }}
     ```  
   - Credentials: Create/Open OpenWeatherMap API credential in n8n with your API key, assign to this node  
   - Connect Schedule Trigger output to this node input

3. **Add If node to evaluate weather conditions:**  
   - Name: If is raining  
   - Type: If  
   - Conditions:  
     - Use OR combinator  
     - Condition 1: Number comparison: `{{$json.main.temp}} < 303`  
     - Condition 2: String contains (case sensitive): `{{$json.weather[0].description}}` contains "rain"  
   - Connect HTTP Request output to this node input

4. **Add Set node to format data:**  
   - Name: Format the data  
   - Type: Set  
   - Assign fields:  
     - city → `{{$json.name}}` (string)  
     - temperature (K) → `{{$json.main.temp}}` (string)  
     - humidity → `{{$json.main.humidity}}` (string)  
     - conditions → `{{$json.weather[0].description}}` (string)  
     - status → `"higher than average temperature"` (string)  
   - Connect If node’s **true** output to this node

5. **Add Google Sheets node to append data:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Select or enter your Google Sheet document ID  
   - Sheet Name: Select the sheet with columns: city, temperature (K), humidity, conditions, status  
   - Authentication: Use Google Service Account credentials configured in n8n  
   - Connect Set node output to this node

6. **Create Sticky Note (optional):**  
   - Content: "### Get Weather information and see if it is raining"  
   - Position near the weather data retrieval and condition nodes for clarity

7. **Activate and test:**  
   - Save workflow, activate it, and verify hourly runs append correct data to your sheet  
   - Adjust city or API key as needed

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                         |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Google Sheet must have columns: city, temperature (K), humidity, conditions, status                 | Workflow data structure requirement                     |
| Set up Google Service Account credentials with write access to your Google Sheet                    | Google Cloud Console → Service Accounts → Credentials  |
| Replace OpenWeatherMap API key in credentials before running                                       | https://openweathermap.org/api                          |
| Default city is "Pasay,ph"; modify URL in HTTP Request node to track other locations               | OpenWeatherMap city query parameter                      |
| Workflow triggers every hour by default; adjust frequency in Schedule Trigger node if needed       | n8n Schedule Trigger docs                               |

---

**Disclaimer:**  
The provided text stems exclusively from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.