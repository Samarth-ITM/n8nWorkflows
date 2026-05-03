Daily Weather Reports with OpenWeather API, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/daily-weather-reports-with-openweather-api--google-sheets--and-gmail-6738


# Daily Weather Reports with OpenWeather API, Google Sheets, and Gmail

### 1. Workflow Overview

This workflow automates the process of generating daily weather reports by fetching weather data from the OpenWeather API, storing it in a Google Sheet, and emailing a formatted weather report via Gmail. It is designed to run once daily, at 10 AM IST, providing timely and structured weather information for a specified location.

The workflow can be logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every day at 10 AM IST.
- **1.2 Weather Data Retrieval:** Fetches current weather data from the OpenWeather API based on geographic coordinates.
- **1.3 Data Storage:** Appends or updates the fetched weather data into a Google Sheet for historical tracking.
- **1.4 Email Report Generation:** Formats the weather data into a styled HTML email template.
- **1.5 Email Dispatch:** Sends the generated weather report email to predetermined recipients using Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block starts the workflow daily at a fixed time.
- **Nodes Involved:**  
  - Trigger Daily at 10 AM

- **Node Details:**  
  - **Node Name:** Trigger Daily at 10 AM  
  - **Type:** Schedule Trigger  
  - **Role:** Initiates the workflow every day at 10:00 AM IST.  
  - **Configuration:** Scheduled with an interval trigger set to hour 10 (10 AM).  
  - **Expressions/Variables:** None.  
  - **Input Connections:** None (entry point).  
  - **Output Connections:** Connected to "Fetch Weather from OpenWeather".  
  - **Version Requirements:** Requires n8n version supporting Schedule Trigger v1.2 or later.  
  - **Potential Failures:** Scheduling misconfiguration; timezone issues if n8n instance timezone differs from IST.  
  - **Sub-workflow:** Not applicable.

#### 1.2 Weather Data Retrieval

- **Overview:** Fetches real-time weather data from OpenWeather API using HTTP request with query parameters for location and API key.
- **Nodes Involved:**  
  - Fetch Weather from OpenWeather

- **Node Details:**  
  - **Node Name:** Fetch Weather from OpenWeather  
  - **Type:** HTTP Request  
  - **Role:** Queries OpenWeather API to retrieve current weather data for specified coordinates.  
  - **Configuration:**  
    - Method: GET (default for HTTP Request).  
    - URL: `https://api.openweathermap.org/data/2.5/weather?`  
    - Query Parameters:  
      - `lat`: Latitude coordinate (value expected to be set or dynamically injected; currently empty in config).  
      - `lon`: Longitude coordinate (value expected to be set or dynamically injected; currently empty).  
      - Authentication: HTTP Query Auth using OpenWeather API key credential.  
    - Units: Metric assumed from description but not explicit in node (could be passed as part of query).  
  - **Expressions/Variables:**  
    - Query parameters `lat` and `lon` are placeholders; must be set or injected for proper API call.  
  - **Input Connections:** Receives trigger from "Trigger Daily at 10 AM".  
  - **Output Connections:** Passes API response data to "Append Weather to Sheet".  
  - **Version Requirements:** HTTP Request node version 4.2 or higher.  
  - **Potential Failures:**  
    - Network errors or API downtime.  
    - Authentication failures if API key is invalid.  
    - Missing or invalid latitude/longitude parameters leading to API errors.  
    - Rate limits imposed by OpenWeather API.  
  - **Sub-workflow:** Not applicable.

#### 1.3 Data Storage

- **Overview:** Appends or updates the daily weather data snapshot into a specific Google Sheet for record-keeping and future reference.
- **Nodes Involved:**  
  - Append Weather to Sheet

- **Node Details:**  
  - **Node Name:** Append Weather to Sheet  
  - **Type:** Google Sheets node  
  - **Role:** Writes the weather data fields into a Google Sheet document, either appending new entries or updating existing rows based on matching criteria.  
  - **Configuration:**  
    - Operation: Append or Update.  
    - Document ID: Set to a specific Google Sheet document (`1cQ-TBf3-dqo7njDYzYpxpASYFvEp8lIzH7vpIqTLcwc`).  
    - Sheet Name/ID: Refers to sheet with ID `380635808` named "weather_data".  
    - Mapping Mode: Defined below with explicit column mappings.  
    - Columns Mapped: Includes comprehensive weather parameters such as Country, Coordinates, Temperature, Humidity, Pressure, Visibility, Wind details, Cloudiness, Sunrise/Sunset times, and Date Time in UTC.  
    - Matching Columns: Location used as key for update detection.  
    - Data Type Conversion: Disabled (fields stored as strings).  
  - **Expressions/Variables:**  
    - Uses JSON path expressions to extract specific data from API response, e.g., `{{$json.main.temp}}`, `{{$json.wind.speed}}`.  
  - **Input Connections:** Receives data from "Fetch Weather from OpenWeather".  
  - **Output Connections:** Forwards data to "Generate Weather Email HTML".  
  - **Version Requirements:** Google Sheets node v4.6 or higher.  
  - **Potential Failures:**  
    - Authentication or permission errors with Google Sheets OAuth2 credentials.  
    - Google Sheets API rate limits or quota exhaustion.  
    - Mismatched or missing columns causing data append/update errors.  
    - Connectivity issues.  
  - **Sub-workflow:** Not applicable.

#### 1.4 Email Report Generation

- **Overview:** Constructs a visually formatted HTML email report using the weather data stored in the previous step.
- **Nodes Involved:**  
  - Generate Weather Email HTML

- **Node Details:**  
  - **Node Name:** Generate Weather Email HTML  
  - **Type:** HTML node  
  - **Role:** Generates an HTML string with inline styling to represent weather data in a human-friendly email layout.  
  - **Configuration:**  
    - Inline HTML template with CSS styling.  
    - Uses n8n expressions to inject dynamic weather data into the HTML table cells.  
    - Converts UNIX timestamps (for sunrise, sunset, report time) to UTC human-readable format using JavaScript Date object.  
    - Displays key weather indicators: coordinates, temperature, humidity, pressure, visibility, wind, cloudiness, sunrise/sunset, and report time.  
  - **Expressions/Variables:**  
    - Example: `{{ $json["Temperature (°C)"] }}`,  
    - Date conversions: `{{ new Date($json["Sunrise (UTC)"] * 1000).toUTCString() }}`  
  - **Input Connections:** Data from "Append Weather to Sheet".  
  - **Output Connections:** Sends formatted HTML to "Send Weather Update Email".  
  - **Version Requirements:** HTML node v1.2 or higher.  
  - **Potential Failures:**  
    - Expression evaluation errors if expected JSON fields are missing or undefined.  
    - Date conversion errors if timestamp fields are invalid or null.  
  - **Sub-workflow:** Not applicable.

#### 1.5 Email Dispatch

- **Overview:** Sends the generated weather report email to designated recipients via Gmail.
- **Nodes Involved:**  
  - Send Weather Update Email

- **Node Details:**  
  - **Node Name:** Send Weather Update Email  
  - **Type:** Gmail node  
  - **Role:** Sends an email containing the HTML weather report as the message body.  
  - **Configuration:**  
    - Recipient(s): Not explicitly set in node parameters; must be configured in `sendTo` field.  
    - Subject: Currently set as "Daily Weather Update" (description suggests dynamic date inclusion could be added).  
    - Message Body: Uses the HTML generated by the prior node (`{{$json.html}}`).  
    - Credentials: Gmail OAuth2 credentials configured.  
  - **Expressions/Variables:**  
    - Uses expression to pass HTML content dynamically.  
  - **Input Connections:** Receives data from "Generate Weather Email HTML".  
  - **Output Connections:** None (terminal node).  
  - **Version Requirements:** Gmail node v2.1 or higher.  
  - **Potential Failures:**  
    - Authentication errors with Gmail OAuth2 token.  
    - Email sending limits (Gmail daily quotas).  
    - Missing or invalid recipient email addresses.  
  - **Sub-workflow:** Not applicable.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                      | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                                      |
|-----------------------------|---------------------|------------------------------------|----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note         | Title / Label                      | None                       | None                          | ## Daily Weather Reports with OpenWeather API, Google Sheets, and Gmail                                                        |
| Sticky Note1                | Sticky Note         | Workflow description and node breakdown | None                       | None                          | ## Node Breakdown & Descriptions: <br>- The workflow starts with a Schedule Trigger node named "Trigger Daily at 10 AM", which runs every day at 10:00 AM IST to initiate the weather reporting process.<br><br>- The next node, named "Fetch Weather from OpenWeather", is an HTTP Request node that fetches the latest weather data from the OpenWeather API using coordinates, API key, and metric units.<br><br>- The workflow then proceeds to a Google Sheets node named "Append Weather to Sheet", which appends the fetched weather data into a predefined Google Sheet. The stored fields include: Country, Location Latitude, Location Longitude, Temperature (°C), Feels Like (°C), Min Temp (°C), Max Temp (°C), Humidity (%), Pressure (hPa), Sea Level (hPa), Ground Level (hPa), Visibility (m), Wind Speed (m/s), Wind Direction (°), Wind Gust (m/s), Cloudiness (%), Sunrise (UTC) (stored as UNIX timestamp or formatted date), Sunset (UTC) (stored as UNIX timestamp or formatted date),Date Time (UTC) (report generated time, in UNIX or formatted UTC), Each row in the sheet represents a daily weather snapshot with all key climate indicators.<br><br>- After storing the data, the workflow uses a Set or Function node named "Generate Weather Email HTML", which creates a styled HTML email template by formatting the weather data into a visually readable format.<br><br>- Finally, the Gmail node named "Send Weather Update Email" sends the formatted weather report to a predefined email list, with a subject line like “Daily Weather Report – {{ $now.toFormat('dd MMM yyyy') }}” and the generated HTML as the message body. |
| Trigger Daily at 10 AM      | Schedule Trigger    | Initiates workflow daily at 10 AM | None                       | Fetch Weather from OpenWeather |                                                                                                                                 |
| Fetch Weather from OpenWeather | HTTP Request       | Fetches weather data from OpenWeather API | Trigger Daily at 10 AM      | Append Weather to Sheet         | Fetching the weather data                                                                                                       |
| Append Weather to Sheet     | Google Sheets       | Stores weather data in Google Sheet | Fetch Weather from OpenWeather | Generate Weather Email HTML     |                                                                                                                                 |
| Generate Weather Email HTML | HTML                | Generates HTML email report        | Append Weather to Sheet     | Send Weather Update Email       |                                                                                                                                 |
| Send Weather Update Email   | Gmail               | Sends the weather report email     | Generate Weather Email HTML | None                          |                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Trigger Daily at 10 AM"  
   - Configure the trigger rule to run once daily at 10:00 AM IST (hour 10).  
   - No credentials needed.

2. **Create the HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Fetch Weather from OpenWeather"  
   - Set the HTTP method to GET (default).  
   - URL: `https://api.openweathermap.org/data/2.5/weather?`  
   - Add Query Parameters:  
     - `lat`: Enter the latitude of the location (e.g., `28.6139` for New Delhi).  
     - `lon`: Enter the longitude of the location (e.g., `77.2090`).  
     - Add other parameters if needed (e.g., `units=metric`).  
   - Authentication: Configure HTTP Query Auth with your OpenWeather API key:  
     - Credential Type: HTTP Query Auth  
     - Parameter Name: `appid` (as per OpenWeather API)  
     - API Key: Your OpenWeather API key  
   - Connect "Trigger Daily at 10 AM" → "Fetch Weather from OpenWeather".

3. **Create the Google Sheets Node**  
   - Type: Google Sheets  
   - Name: "Append Weather to Sheet"  
   - Operation: Append or Update  
   - Document ID: Set to your Google Sheet document ID where weather data will be stored.  
   - Sheet Name/ID: Set the target sheet/tab name or ID (e.g., "weather_data").  
   - Mapping Mode: Define Below  
   - Map the following columns to corresponding JSON fields from OpenWeather API response:  
     - Country → `$json.sys.country`  
     - Location → `$json.coord` (latitude and longitude object)  
     - Temperature (°C) → `$json.main.temp`  
     - Feels Like (°C) → `$json.main.feels_like`  
     - Min Temp (°C) → `$json.main.temp_min`  
     - Max Temp (°C) → `$json.main.temp_max`  
     - Humidity (%) → `$json.main.humidity`  
     - Pressure (hPa) → `$json.main.pressure`  
     - Sea Level (hPa) → `$json.main.sea_level`  
     - Ground Level (hPa) → `$json.main.grnd_level`  
     - Visibility (m) → `$json.visibility`  
     - Wind Speed (m/s) → `$json.wind.speed`  
     - Wind Direction (°) → `$json.wind.deg`  
     - Wind Gust (m/s) → `$json.wind.gust`  
     - Cloudiness (%) → `$json.clouds.all`  
     - Sunrise (UTC) → `$json.sys.sunrise` (UNIX timestamp)  
     - Sunset (UTC) → `$json.sys.sunset` (UNIX timestamp)  
     - Date Time (UTC) → `$json.dt` (UNIX timestamp)  
   - Matching Columns: Set "Location" to be used for update detection.  
   - Credentials: Configure Google Sheets OAuth2 credentials with access to the target spreadsheet.  
   - Connect "Fetch Weather from OpenWeather" → "Append Weather to Sheet".

4. **Create the HTML Node**  
   - Type: HTML  
   - Name: "Generate Weather Email HTML"  
   - Paste the provided HTML template into the HTML parameter, which includes inline CSS styling and uses expressions to inject weather data dynamically.  
   - Ensure expressions correctly reference the incoming JSON fields from the Google Sheets node (same field names as mapped).  
   - Connect "Append Weather to Sheet" → "Generate Weather Email HTML".

5. **Create the Gmail Node**  
   - Type: Gmail  
   - Name: "Send Weather Update Email"  
   - Configure the following:  
     - Send To: Enter recipient email addresses (comma separated if multiple).  
     - Subject: Set as "Daily Weather Update" or use expression like `"Daily Weather Report – " + $now.toFormat('dd MMM yyyy')` for dynamic date.  
     - Message: Set as expression `{{$json.html}}` to use the generated HTML content.  
   - Credentials: Set up Gmail OAuth2 credentials with permission to send emails.  
   - Connect "Generate Weather Email HTML" → "Send Weather Update Email".

6. **Activate the Workflow**  
   - Ensure all nodes have valid credentials and parameters set.  
   - Save and activate the workflow.  
   - Verify the timezone setting in n8n matches expectations for the daily trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow was designed to run daily at 10 AM IST; adjust schedule trigger timezone if your n8n instance timezone differs.                                                                                                  | Scheduling considerations             |
| Gmail node requires OAuth2 credentials with Gmail API enabled and proper scopes for sending emails.                                                                                                                           | Gmail OAuth2 setup                    |
| OpenWeather API usage may require an active subscription depending on usage volume; monitor API quotas and rate limits.                                                                                                        | https://openweathermap.org/api        |
| Google Sheets node requires OAuth2 credentials with edit access to the target spreadsheet; ensure the sheet ID and tab are correctly set.                                                                                     | Google Sheets API documentation       |
| The HTML email template uses inline CSS and expressions to format the weather data; modifications to the template should maintain correct JSON field references to avoid rendering errors.                                      | HTML email formatting best practices  |
| UNIX timestamp fields are converted to UTC string format in the email using JavaScript Date conversions inside expressions (`new Date(timestamp * 1000).toUTCString()`).                                                        | Date-time conversions                  |
| To extend this workflow, consider parameterizing latitude and longitude inputs via environment variables or external triggers for multi-location reports.                                                                       | Workflow scalability                   |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.