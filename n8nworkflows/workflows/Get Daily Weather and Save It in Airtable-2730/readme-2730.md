Get Daily Weather and Save It in Airtable

https://n8nworkflows.xyz/workflows/get-daily-weather-and-save-it-in-airtable-2730


# Get Daily Weather and Save It in Airtable

### 1. Workflow Overview

This workflow automates the daily retrieval and storage of weather data for a specified location using the OpenWeatherMap API and Airtable. It is designed to run once every day at a scheduled time, fetch current weather metrics such as temperature, humidity, wind speed, and location timezone, then save these details into a predefined Airtable base and table. This creates a reliable historical weather dataset for analysis or reporting purposes.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow execution daily at a set hour.
- **1.2 Fetch Weather Data:** Calls the OpenWeatherMap API to retrieve current weather information.
- **1.3 Store Weather Data:** Parses and maps the API response data into Airtable fields for persistent storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block sets the workflow to run automatically once per day at a specific hour, triggering the entire data fetch and storage process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Role: Starts the workflow daily at a configured time  
    - Configuration:  
      - Cron-like schedule set to trigger at 10:00 AM every day (triggerAtHour: 10)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get Weather Data" node  
    - Version: 1.2  
    - Potential Failures: Misconfiguration of schedule time, time zone discrepancies, or n8n instance downtime at trigger time  
    - Notes: No credentials required  

#### 1.2 Fetch Weather Data

- **Overview:**  
  This block performs an HTTP request to the OpenWeatherMap API to retrieve the current weather data for a fixed geographic location (latitude 23.0059, longitude 72.5547). It uses query parameters to specify units and authenticates via query parameter API key.

- **Nodes Involved:**  
  - Get Weather Data (HTTP Request)

- **Node Details:**

  - **Get Weather Data**  
    - Type: HTTP Request node  
    - Role: Fetches weather data from OpenWeatherMap API  
    - Configuration:  
      - URL: `https://api.openweathermap.org/data/2.5/weather?lat=23.0059&lon=72.5547`  
      - Query Parameters: units=metric  
      - Authentication: HTTP Query Authentication using OpenWeatherMap API key (credential named "OpenWeatherMap Query Auth")  
      - HTTP Method: GET (default)  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Passes response JSON to "Store Weather Data" node  
    - Version: 4.2  
    - Potential Failures:  
      - API key invalid or expired (authentication error)  
      - Network timeout or connectivity issues  
      - API rate limiting or quota exceeded  
      - Unexpected API response structure causing parsing errors  
    - Notes: Multiple credentials are attached but only HTTP Query Auth is used here  

#### 1.3 Store Weather Data

- **Overview:**  
  This block takes the JSON response from the weather API, extracts relevant fields, and inserts them as a new record into a specified Airtable base and table.

- **Nodes Involved:**  
  - Store Weather Data (Airtable node)

- **Node Details:**

  - **Store Weather Data**  
    - Type: Airtable node  
    - Role: Inserts a new record into Airtable with weather data fields  
    - Configuration:  
      - Base: Airtable base identified by ID `appKtypfMptBIKStp` (named "WeatherData")  
      - Table: Airtable table identified by ID `tblfb3sJ84eQUlYJd` (named "Data")  
      - Operation: Create record  
      - Fields Mapped:  
        - Temp: `{{$json.main.temp}}` (temperature in Celsius)  
        - Humidity: `{{$json.main.humidity}}` (percentage)  
        - Location: `{{$json.name}}` (city name)  
        - Timezone: `{{$json.timezone}}` (timezone offset in seconds)  
        - Wind Speed: `{{$json.wind.speed}}` (meters per second)  
      - Mapping Mode: Explicit field mapping defined below  
    - Inputs: Receives JSON data from "Get Weather Data" node  
    - Outputs: None (end of workflow)  
    - Credentials: Airtable API token (configured externally)  
    - Version: 2.1  
    - Potential Failures:  
      - Invalid or missing Airtable API token (authentication failure)  
      - Incorrect base or table ID causing record creation failure  
      - Data type mismatches in mapped fields  
      - Airtable API rate limits exceeded  
    - Notes: Includes a sticky note describing its purpose  

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                  | Input Node(s)      | Output Node(s)      | Sticky Note                                                                                                                             |
|--------------------|---------------------|---------------------------------|--------------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger    | Initiates daily workflow trigger | None               | Get Weather Data    |                                                                                                                                         |
| Get Weather Data   | HTTP Request        | Fetches weather data from API    | Schedule Trigger   | Store Weather Data  | Fetching the weather data                                                                                                               |
| Store Weather Data | Airtable            | Saves weather data into Airtable | Get Weather Data   | None                | Store weather data in table                                                                                                             |
| Sticky Note        | Sticky Note         | Workflow title and description   | None               | None                | Automated Daily Weather Data Fetcher and Storage                                                                                        |
| Sticky Note1       | Sticky Note         | Workflow purpose explanation     | None               | None                | This workflow fetches weather data daily using the OpenWeatherMap API and stores the weather information in Airtable...                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Automated Daily Weather Data Fetcher and Storage".

2. **Add a Schedule Trigger node**:  
   - Set the trigger to run daily at 10:00 AM (set `triggerAtHour` to 10).  
   - No credentials needed.

3. **Add an HTTP Request node** named "Get Weather Data":  
   - Connect the output of the Schedule Trigger to this node.  
   - Set the HTTP Method to GET.  
   - URL: `https://api.openweathermap.org/data/2.5/weather?lat=23.0059&lon=72.5547`  
   - Add a query parameter: `units` = `metric`  
   - Configure authentication as HTTP Query Authentication:  
     - Create or select a credential with your OpenWeatherMap API key set as a query parameter (e.g., `appid=YOUR_API_KEY`).  
   - Save node.

4. **Add an Airtable node** named "Store Weather Data":  
   - Connect the output of "Get Weather Data" to this node.  
   - Set operation to "Create".  
   - Select your Airtable base (e.g., "WeatherData") and table (e.g., "Data").  
   - Map the fields as follows:  
     - Temp: `{{$json.main.temp}}`  
     - Humidity: `{{$json.main.humidity}}`  
     - Location: `{{$json.name}}`  
     - Timezone: `{{$json.timezone}}`  
     - Wind Speed: `{{$json.wind.speed}}`  
   - Configure Airtable credentials with your API token.  
   - Save node.

5. **Add Sticky Notes** (optional) for documentation purposes:  
   - One at the top with the workflow title.  
   - One describing the workflow purpose below.

6. **Save and activate** the workflow to enable daily execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow fetches weather data daily using the OpenWeatherMap API and stores the weather information in Airtable. The data can include temperature, humidity, wind speed, and other relevant weather details. | Workflow purpose explanation (Sticky Note1)                                                                     |
| Created by the AI team at WeblineIndia, specializing in custom automation solutions and software workflows.                                                     | https://www.weblineindia.com/process-automation-solutions.html                                                  |
| OpenWeatherMap API documentation for reference on query parameters and authentication: https://openweathermap.org/api                                          | External API documentation                                                                                       |
| Airtable API documentation for field mapping and authentication: https://airtable.com/api                                                                        | External API documentation                                                                                       |

---

This documentation provides a complete understanding of the workflow’s structure, configuration, and operational details, enabling reproduction or modification with confidence.