National Weather Service 7-Day Forecast in Slack

https://n8nworkflows.xyz/workflows/national-weather-service-7-day-forecast-in-slack-2947


# National Weather Service 7-Day Forecast in Slack

### 1. Workflow Overview

This workflow enables Slack users to retrieve a 7-day weather forecast for any city by typing a custom slash command `/weather [cityname]` in Slack. It integrates Slack with geolocation and weather APIs to convert a city name into geographic coordinates, fetch detailed weather data from the National Weather Service (NWS), and post a formatted weather summary back to Slack.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the Slack slash command input via a webhook.
- **1.2 Geocoding:** Converts the city name into latitude and longitude using OpenStreetMap’s Nominatim API.
- **1.3 Weather Data Retrieval:** Uses the coordinates to query the NWS API for location-specific weather grid data and forecast.
- **1.4 Formatting and Slack Notification:** Parses the forecast data and posts a structured weather report back to Slack.

This modular design allows easy adaptation for other communication platforms or output formats.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests triggered by the Slack slash command `/weather`. It captures the city name parameter from the Slack command text.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Entry point for the workflow; receives POST requests from Slack’s slash command.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/slack1` (endpoint exposed by n8n)  
    - Key Expressions/Variables:  
      - Extracts the city name from `body.text` in the incoming Slack payload.  
    - Input: HTTP POST request from Slack slash command  
    - Output: JSON containing Slack command data, including the city name  
    - Edge Cases:  
      - Missing or empty city name parameter  
      - Invalid HTTP method or incorrect webhook URL  
      - Slack command payload format changes  
    - Version Requirements: n8n webhook node v2 or higher

#### 2.2 Geocoding

- **Overview:**  
  Converts the city name string into geographic coordinates (latitude and longitude) using OpenStreetMap’s Nominatim API.

- **Nodes Involved:**  
  - OpenStreetMap

- **Node Details:**

  - **OpenStreetMap**  
    - Type: HTTP Request  
    - Role: Calls the Nominatim API to geocode the city name.  
    - Configuration:  
      - URL: `https://nominatim.openstreetmap.org/search`  
      - Query Parameters:  
        - `q`: city name extracted from `Webhook` node (`{{$node["Webhook"].item.json.body.text}}`)  
        - `format`: `json` (to get JSON response)  
      - Headers:  
        - `User-Agent`: Custom user agent string (`alexk1919 (alex@alexk1919.com)`) as required by Nominatim usage policy  
      - Response: Full HTTP response enabled for detailed data access  
    - Input: JSON from `Webhook` node containing city name  
    - Output: JSON array of location results with lat/lon  
    - Edge Cases:  
      - No results found for city name (empty array)  
      - API rate limits or service downtime  
      - Malformed city names causing no matches  
    - Version Requirements: HTTP Request node v4.2 or higher

#### 2.3 Weather Data Retrieval

- **Overview:**  
  Uses the latitude and longitude from geocoding to query the National Weather Service API for gridpoint metadata and then fetches the 7-day forecast.

- **Nodes Involved:**  
  - NWS  
  - NWS1

- **Node Details:**

  - **NWS**  
    - Type: HTTP Request  
    - Role: Retrieves gridpoint metadata (gridId, gridX, gridY) for the given coordinates.  
    - Configuration:  
      - URL: `https://api.weather.gov/points/{{ $json.body[0].lat }},{{ $json.body[0].lon }}`  
        - Uses the first geocoding result’s latitude and longitude from `OpenStreetMap` node output.  
      - Headers:  
        - `User-Agent`: `alexk1919 (alex@alexk1919.com)` (required by NWS API)  
      - Response: Full HTTP response enabled  
    - Input: JSON from `OpenStreetMap` node with lat/lon  
    - Output: JSON containing gridpoint metadata  
    - Edge Cases:  
      - Invalid or missing lat/lon coordinates  
      - API rate limits or downtime  
      - Unexpected response structure  
    - Version Requirements: HTTP Request node v4.2 or higher

  - **NWS1**  
    - Type: HTTP Request  
    - Role: Fetches the detailed 7-day weather forecast using gridpoint metadata.  
    - Configuration:  
      - URL:  
        ```
        https://api.weather.gov/gridpoints/{{ $json.data ? JSON.parse($json.data).properties.gridId : "" }}/{{ $json.data ? JSON.parse($json.data).properties.gridX : "" }},{{ $json.data ? JSON.parse($json.data).properties.gridY : "" }}/forecast
        ```  
        - Extracts `gridId`, `gridX`, and `gridY` from the parsed JSON data of the previous `NWS` node.  
      - Headers:  
        - `User-Agent`: `alexk1919 (alex@alexk1919.com)`  
      - Response: Full HTTP response enabled  
    - Input: JSON from `NWS` node containing gridpoint metadata  
    - Output: JSON with detailed forecast data  
    - Edge Cases:  
      - Missing or malformed gridpoint data  
      - API errors or empty forecast data  
      - Parsing errors due to unexpected JSON structure  
    - Version Requirements: HTTP Request node v4.2 or higher

#### 2.4 Formatting and Slack Notification

- **Overview:**  
  Formats the forecast data into a human-readable message and posts it back to the Slack channel where the command was issued.

- **Nodes Involved:**  
  - Slack

- **Node Details:**

  - **Slack**  
    - Type: Slack node (Slack API integration)  
    - Role: Sends the formatted weather forecast message to a Slack channel.  
    - Configuration:  
      - Text: Uses a JavaScript expression to parse the forecast periods from `NWS1` node JSON response and formats each day’s forecast with:  
        - Day name (e.g., Monday)  
        - Temperature with unit  
        - Wind speed and direction  
        - Short forecast summary  
      - Channel: Fixed channel ID (`C0889718P8S`) selected from Slack channel list  
      - Authentication: OAuth2 with Slack credentials (`AlexK Slack account`)  
    - Key Expression:  
      ```js
      {{
        JSON.parse($node["NWS1"].json.data).properties.periods
        .map(period => 
          `*${period.name}*\n` +
          `Temp: ${period.temperature}°${period.temperatureUnit}\n` +
          `Wind: ${period.windSpeed} ${period.windDirection}\n` +
          `Forecast: ${period.shortForecast}`
        )
        .join("\n\n")
      }}
      ```  
    - Input: JSON forecast data from `NWS1` node  
    - Output: Slack message posted to channel  
    - Edge Cases:  
      - Slack API authentication failures  
      - Invalid or missing forecast data causing expression errors  
      - Channel permission issues or invalid channel ID  
    - Version Requirements: Slack node v2.3 or higher  
    - Credentials: Slack OAuth2 with message posting scope

---

### 3. Summary Table

| Node Name    | Node Type           | Functional Role                  | Input Node(s)     | Output Node(s) | Sticky Note                                                                                          |
|--------------|---------------------|--------------------------------|-------------------|----------------|----------------------------------------------------------------------------------------------------|
| Webhook      | Webhook Trigger     | Receives Slack slash command   | -                 | OpenStreetMap  |                                                                                                    |
| OpenStreetMap| HTTP Request        | Geocodes city name to lat/lon  | Webhook           | NWS            | Uses OpenStreetMap Nominatim API with User-Agent header as per usage policy                         |
| NWS          | HTTP Request        | Gets gridpoint metadata from NWS API | OpenStreetMap     | NWS1           | Requires User-Agent header for NWS API requests                                                    |
| NWS1         | HTTP Request        | Fetches 7-day weather forecast | NWS               | Slack          | Parses gridId, gridX, gridY from previous node’s JSON data                                         |
| Slack        | Slack API           | Posts formatted forecast to Slack channel | NWS1              | -              | Posts to fixed Slack channel using OAuth2 credentials; formats message with forecast details       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `slack1`  
   - Purpose: Receive Slack slash command POST requests containing the city name.  

2. **Create HTTP Request Node for OpenStreetMap**  
   - Name: OpenStreetMap  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://nominatim.openstreetmap.org/search`  
   - Query Parameters:  
     - `q`: Expression: `{{$node["Webhook"].item.json.body.text}}` (extract city name from Slack command)  
     - `format`: `json`  
   - Headers:  
     - `User-Agent`: `alexk1919 (alex@alexk1919.com)` (required by Nominatim)  
   - Connect Webhook node output to this node input.

3. **Create HTTP Request Node for NWS Gridpoint Metadata**  
   - Name: NWS  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: Expression:  
     ```
     https://api.weather.gov/points/{{$json.body[0].lat}},{{$json.body[0].lon}}
     ```  
     (Uses first geocoding result from OpenStreetMap)  
   - Headers:  
     - `User-Agent`: `alexk1919 (alex@alexk1919.com)` (required by NWS API)  
   - Connect OpenStreetMap node output to this node input.

4. **Create HTTP Request Node for NWS Forecast**  
   - Name: NWS1  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: Expression:  
     ```
     https://api.weather.gov/gridpoints/{{ $json.data ? JSON.parse($json.data).properties.gridId : "" }}/{{ $json.data ? JSON.parse($json.data).properties.gridX : "" }},{{ $json.data ? JSON.parse($json.data).properties.gridY : "" }}/forecast
     ```  
     (Extracts gridId, gridX, gridY from previous node’s JSON data)  
   - Headers:  
     - `User-Agent`: `alexk1919 (alex@alexk1919.com)`  
   - Connect NWS node output to this node input.

5. **Create Slack Node to Post Message**  
   - Name: Slack  
   - Type: Slack  
   - Authentication: OAuth2 credentials with permission to post messages (`AlexK Slack account`)  
   - Channel: Select or enter Slack channel ID (e.g., `C0889718P8S`)  
   - Text: Use the following expression to format the forecast:  
     ```js
     {{
       JSON.parse($node["NWS1"].json.data).properties.periods
       .map(period => 
         `*${period.name}*\n` +
         `Temp: ${period.temperature}°${period.temperatureUnit}\n` +
         `Wind: ${period.windSpeed} ${period.windDirection}\n` +
         `Forecast: ${period.shortForecast}`
       )
       .join("\n\n")
     }}
     ```  
   - Connect NWS1 node output to Slack node input.

6. **Set Execution Order and Activate Workflow**  
   - Ensure connections follow:  
     Webhook → OpenStreetMap → NWS → NWS1 → Slack  
   - Activate the workflow in n8n.

7. **Slack App Setup**  
   - Create a Slack app at https://api.slack.com/apps  
   - Add a Slash Command `/weather` with the webhook URL from n8n (e.g., `https://your-n8n-instance.com/webhook/slack1`)  
   - Add OAuth scopes to allow posting messages (`chat:write`)  
   - Install the app to your Slack workspace and obtain OAuth credentials for n8n Slack node.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                  |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Slack slash command `/weather [cityname]` triggers this workflow to fetch and post weather updates.| Slack Custom Slash Commands                      |
| OpenStreetMap Nominatim API requires a custom User-Agent header to comply with usage policy.       | https://operations.osmfoundation.org/policies/nominatim/ |
| National Weather Service API requires User-Agent header identifying the requester.                   | https://www.weather.gov/documentation/services-web-api |
| Slack OAuth2 credentials must have `chat:write` scope to post messages.                             | https://api.slack.com/scopes/chat:write          |
| Workflow can be adapted to send weather updates via email, Discord, Microsoft Teams, or Telegram.  | Customizable output node                          |
| For detailed Slack app creation and slash command setup, see: https://api.slack.com/interactivity/slash-commands | Slack API Documentation                          |

---

This documentation fully describes the "Weather via Slack" workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.