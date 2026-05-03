Send weather alerts to your mobile phone with OpenWeatherMap and SIGNL4

https://n8nworkflows.xyz/workflows/send-weather-alerts-to-your-mobile-phone-with-openweathermap-and-signl4-966


# Send weather alerts to your mobile phone with OpenWeatherMap and SIGNL4

### 1. Workflow Overview

This workflow automates the process of monitoring weather conditions daily and sending alerts to a SIGNL4 on-call team via push notification, SMS, or voice call. Its primary use case is to enable timely responses to specific weather situations such as freezing temperatures, snow, hail storms, or heat, aiding operational decisions like dispatching teams or protecting assets.

The workflow is logically organized into these main blocks:

- **1.1 Input Triggers:** Initiates the workflow either manually or on a scheduled daily basis.
- **1.2 Weather Data Retrieval:** Fetches current weather information for a specified city using OpenWeatherMap.
- **1.3 Conditional Evaluation:** Checks if temperature conditions meet alert criteria.
- **1.4 Alert Dispatch:** Sends a weather alert to SIGNL4 if conditions are met, including location data and alert resolution capabilities.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Triggers

- **Overview:** This block provides two entry points to start the workflow: manual trigger for testing and a scheduled trigger for daily automation.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Schedule Trigger

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - Type: Manual Trigger  
     - Role: Allows manual initiation of the workflow for testing or immediate execution.  
     - Configuration: Default, no parameters set.  
     - Inputs: None (trigger node)  
     - Outputs: Connects to OpenWeatherMap node.  
     - Edge cases: None significant; manual trigger is user-controlled.  

  2. **Schedule Trigger**  
     - Type: Schedule Trigger  
     - Role: Automatically initiates the workflow every day at 06:15 AM.  
     - Configuration: Set to trigger daily at 6:15 AM (hour:6, minute:15).  
     - Inputs: None (trigger node)  
     - Outputs: Connects to OpenWeatherMap node.  
     - Edge cases: Timezone differences may affect trigger time; ensure server timezone matches expected schedule.

---

#### Block 1.2: Weather Data Retrieval

- **Overview:** Retrieves the current weather data for Berlin from OpenWeatherMap API.
- **Nodes Involved:**  
  - OpenWeatherMap

- **Node Details:**

  1. **OpenWeatherMap**  
     - Type: OpenWeatherMap API Node  
     - Role: Fetches live weather data such as temperature and coordinates.  
     - Configuration:  
       - CityName: Berlin  
       - Credentials: Uses stored OpenWeatherMap API key (OAuth or API key).  
     - Key expressions: None (static city name input).  
     - Inputs: Connected from both triggers (manual and schedule).  
     - Outputs: Connects to the If node for condition evaluation.  
     - Edge cases:  
       - API authentication errors due to invalid or expired keys.  
       - Network timeouts or API rate limits.  
       - City name misspelling or unavailability could cause no data or errors.

---

#### Block 1.3: Conditional Evaluation

- **Overview:** Checks if the temperature is less than 25°C to determine if an alert should be sent.
- **Nodes Involved:**  
  - If

- **Node Details:**

  1. **If**  
     - Type: Conditional Node  
     - Role: Evaluates temperature from OpenWeatherMap response and routes the workflow accordingly.  
     - Configuration:  
       - Condition: temperature (accessed as `{{$json.main.temp}}`) < 25  
       - Type validation: strict number comparison  
       - Case sensitive: true (not relevant for number)  
     - Inputs: From OpenWeatherMap node.  
     - Outputs:  
       - True branch: connects to SIGNL4 node to send alert  
       - False branch: no further connection (alert not sent)  
     - Edge cases:  
       - If temperature data is missing or malformed, condition evaluation may fail.  
       - If the API response structure changes, the expression `{{$json.main.temp}}` might break.

---

#### Block 1.4: Alert Dispatch

- **Overview:** Sends a weather alert to the SIGNL4 on-call team including temperature and location, with support for automatic alert resolution using externalId.
- **Nodes Involved:**  
  - SIGNL4

- **Node Details:**

  1. **SIGNL4**  
     - Type: SIGNL4 Notification Node  
     - Role: Sends push, SMS, or voice call alerts to the SIGNL4 on-call team.  
     - Configuration:  
       - Message: `"Weather alert ❄️ Temperature: {{ $json.main.temp }} °C"` (dynamic temperature insertion)  
       - Title: `"Weather Alert from n8n"`  
       - External ID: `"weather-alert"` (used for automatic alert resolution)  
       - Location: passes latitude and longitude from OpenWeatherMap response (`{{$json.coord.lat}}`, `{{$json.coord.lon}}`)  
       - Credentials: SIGNL4 Webhook API credentials stored securely  
     - Inputs: From If node’s true branch.  
     - Outputs: None (end node)  
     - Edge cases:  
       - Authentication failure if SIGNL4 credentials expire or are invalid.  
       - Network errors or SIGNL4 service downtime.  
       - Incorrect or missing coordinates may cause location data to be invalid.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role           | Input Node(s)                 | Output Node(s)          | Sticky Note                                      |
|--------------------------|---------------------|--------------------------|------------------------------|-------------------------|-------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual start trigger     | None                         | OpenWeatherMap          |                                                 |
| Schedule Trigger         | Schedule Trigger    | Scheduled daily trigger  | None                         | OpenWeatherMap          |                                                 |
| OpenWeatherMap           | OpenWeatherMap API  | Fetch weather data       | When clicking ‘Test workflow’, Schedule Trigger | If                      |                                                 |
| If                       | If Condition        | Temperature evaluation   | OpenWeatherMap               | SIGNL4 (true branch)    |                                                 |
| SIGNL4                   | SIGNL4 Notification | Send alert to SIGNL4     | If (true condition)          | None                    | [SIGNL4](https://www.signl4.com) on-call alerts |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To allow manual execution and testing of the workflow.  
   - No additional configuration needed.

2. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure it to trigger every day at 06:15 AM (set hour to 6, minute to 15).  
   - No credentials required.

3. **Create an OpenWeatherMap Node**  
   - Type: OpenWeatherMap  
   - Set **City Name** parameter to `"Berlin"`.  
   - Configure OpenWeatherMap API credentials (API key) in n8n credentials manager.  
   - Connect outputs of both Manual Trigger and Schedule Trigger to this node's input.

4. **Create an If Node**  
   - Type: If Condition  
   - Configure a condition:  
     - Left Value: expression `{{$json.main.temp}}` (temperature from OpenWeatherMap data)  
     - Operator: less than (`<`)  
     - Right Value: `25` (degrees Celsius)  
   - Ensure type validation is set to strict number.  
   - Connect OpenWeatherMap node output to this node input.

5. **Create a SIGNL4 Node**  
   - Type: SIGNL4  
   - Configure message to:  
     `Weather alert ❄️ Temperature: {{ $json.main.temp }} °C` (dynamic insertion of temp)  
   - Set title to: `"Weather Alert from n8n"`  
   - Set external ID to `"weather-alert"` (to enable automatic alert resolution).  
   - Set location fields using expressions:  
     - Latitude: `{{$json.coord.lat}}`  
     - Longitude: `{{$json.coord.lon}}`  
   - Configure SIGNL4 API credentials in n8n credentials manager (Webhook account).  
   - Connect the If node’s **true** output to this node’s input.

6. **Finalize Connections**  
   - Connect the Manual Trigger node output to OpenWeatherMap node input.  
   - Connect the Schedule Trigger node output to OpenWeatherMap node input.  
   - Connect OpenWeatherMap node output to If node input.  
   - Connect If node’s true output to SIGNL4 node input.

7. **Save and Activate Workflow**  
   - Test manually using the manual trigger.  
   - Activate for scheduled daily execution.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                |
|------------------------------------------------------------------------------|------------------------------------------------|
| The workflow supports automatic alert resolution in SIGNL4 using externalId. | https://www.signl4.com                          |
| Use cases include dispatching snow removal, protecting cars during hail, and securing sails during high winds. | Workflow description                            |
| The workflow can be adapted easily for other weather alerts such as rain or hail storms. | Workflow description                            |
| Official SIGNL4 website for more details on alerting capabilities.            | https://www.signl4.com                          |
| GIF animation illustrating the alert sending process in SIGNL4.              | Embedded in original workflow description file |

---

This structured documentation should provide a clear understanding of the workflow logic, enable precise reproduction or modification, and highlight potential points of failure or integration considerations.