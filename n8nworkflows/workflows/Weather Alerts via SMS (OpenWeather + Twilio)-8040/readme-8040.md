Weather Alerts via SMS (OpenWeather + Twilio)

https://n8nworkflows.xyz/workflows/weather-alerts-via-sms--openweather---twilio--8040


# Weather Alerts via SMS (OpenWeather + Twilio)

### 1. Workflow Overview

This workflow automates weather alert notifications via SMS using OpenWeather and Twilio APIs. It is designed for users who want periodic weather updates and urgent alerts for extreme or severe weather conditions delivered directly to their phones.

**Target Use Cases:**

- Automated weather monitoring for a specified city (default: New York, US).
- Timely alerts for rain, snow, extreme temperatures, high winds, and severe weather phenomena.
- SMS delivery of concise, actionable weather alerts to predefined phone numbers.
- Scheduled execution every 6 hours to maintain up-to-date notifications.

**Logical Blocks:**

- **1.1 Setup & Configuration:** Instructions and prerequisites for API keys and recipients.
- **1.2 Scheduled Trigger:** Cron node to initiate the workflow every 6 hours.
- **1.3 Weather Data Retrieval:** HTTP requests to OpenWeather API for current weather and forecast data.
- **1.4 Weather Data Analysis:** JavaScript code node to normalize data, detect alert conditions, and prepare structured output.
- **1.5 Alert Decision:** Conditional check on whether any alert conditions were triggered.
- **1.6 SMS Formatting:** Code node to compose the SMS message with current conditions, alerts, and short forecast.
- **1.7 SMS Sending:** Twilio node to send the alert message to configured recipients.
- **1.8 Logging:** Code node to log details of sent alerts for monitoring and auditing.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Configuration

- **Overview:** Provides detailed setup instructions for users to configure necessary API keys, phone numbers, and customize alert conditions.
- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)
- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note (n8n-nodes-base.stickyNote)  
    - Role: Documentation and user guidance embedded inside the workflow.  
    - Content:  
      - OpenWeather API key acquisition and usage limits.  
      - Twilio account setup and credential instructions.  
      - Recipient phone number format and update instructions.  
      - Alert condition customization hints.  
      - Run interval information (every 6 hours).  
    - Input/Output: None (informational only).  
    - Edge Cases: None.

#### 2.2 Scheduled Trigger

- **Overview:** Triggers the workflow execution every 6 hours to ensure regular weather monitoring.
- **Nodes Involved:**  
  - Check Every 6 Hours (Cron)
- **Node Details:**

  - **Check Every 6 Hours**  
    - Type: Cron Trigger (n8n-nodes-base.cron)  
    - Configuration: Default (runs every 6 hours).  
    - Input: None (trigger node).  
    - Output: Triggers downstream HTTP requests for weather data.  
    - Edge Cases: Cron misconfiguration could cause missed runs.  
    - Version: Standard cron node, no special requirements.

#### 2.3 Weather Data Retrieval

- **Overview:** Fetches current weather and 5-day forecast data (limited to next 8 intervals, approx. 24 hours) from OpenWeather API for the specified city.
- **Nodes Involved:**  
  - Get Current Weather (HTTP Request)  
  - Get Weather Forecast (HTTP Request)
- **Node Details:**

  - **Get Current Weather**  
    - Type: HTTP Request  
    - URL: `https://api.openweathermap.org/data/2.5/weather`  
    - Query Parameters:  
      - `q`: City and country (default "New York,US")  
      - `appid`: API key placeholder `"YOUR_OPENWEATHER_API_KEY"` (must be replaced)  
      - `units`: `"imperial"` (Fahrenheit)  
    - Output: JSON with current weather data.  
    - Edge Cases: API key invalid, rate limits, network failures, city not found.  
    - Version: 4.1.

  - **Get Weather Forecast**  
    - Type: HTTP Request  
    - URL: `https://api.openweathermap.org/data/2.5/forecast`  
    - Query Parameters: Same as above plus:  
      - `cnt`: `"8"` (limit to next 8 forecast intervals)  
    - Output: JSON with forecast data.  
    - Edge Cases: Same as above.  
    - Version: 4.1.

#### 2.4 Weather Data Analysis

- **Overview:** Processes and normalizes the weather data, evaluates alert conditions based on temperature, precipitation, wind, and severe weather, and aggregates alerts.
- **Nodes Involved:**  
  - Analyze Weather Data (Code)
- **Node Details:**

  - **Analyze Weather Data**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Reads JSON from both weather nodes via expressions (`$('Get Current Weather').item.json`, `$('Get Weather Forecast').item.json`).  
      - Extracts and rounds key parameters: temperature, feels like, humidity, weather conditions, wind speed, location.  
      - Generates a list of upcoming 24-hour forecast entries.  
      - Checks multiple alert conditions:  
        - Extreme heat (≥ 95°F) or extreme cold (≤ 20°F).  
        - Severe weather (`thunderstorm`, `tornado`).  
        - Precipitation alerts (`rain`, `drizzle`, `snow`).  
        - High winds (≥ 25 mph).  
        - Upcoming severe weather or high rain probability (≥ 70%).  
      - Constructs an alerts array and a weatherData object summarizing all relevant info.  
      - Logs summary info to console.  
    - Inputs: JSON from two HTTP nodes.  
    - Outputs: Single JSON object with weather data, alerts, and metadata.  
    - Edge Cases:  
      - Missing fields in API responses.  
      - Unexpected data formats.  
      - Timezone differences affecting date parsing.  
      - Expression failures if node names change.  
    - Version: 2.

#### 2.5 Alert Decision

- **Overview:** Checks if any alerts were detected and routes the workflow accordingly.
- **Nodes Involved:**  
  - Alert Needed? (IF)
- **Node Details:**

  - **Alert Needed?**  
    - Type: IF node  
    - Condition: Checks if `has_alerts` property in JSON is `true`.  
    - True branch: proceeds to format SMS alert.  
    - False branch: workflow ends silently (no SMS sent).  
    - Inputs: Output from Analyze Weather Data.  
    - Outputs: True → Format SMS Alert; False → no further action.  
    - Edge Cases: Missing or malformed `has_alerts` property causing false negatives.

#### 2.6 SMS Formatting

- **Overview:** Creates a human-readable SMS message summarizing current weather, alerts, and a short forecast preview.
- **Nodes Involved:**  
  - Format SMS Alert (Code)
- **Node Details:**

  - **Format SMS Alert**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Reads weatherData from input JSON.  
      - Builds SMS message string with:  
        - Location header.  
        - Current temperature and description.  
        - List of all alerts.  
        - Next 3 hours forecast with time and conditions.  
        - Timestamp of alert.  
      - Determines alert type from first alert string.  
      - Sets urgency as "HIGH" if any alert includes "EXTREME" or "SEVERE", otherwise "MEDIUM".  
      - Logs formatted alert details.  
    - Inputs: Weather data JSON.  
    - Outputs: JSON containing message text, location, alert type, urgency.  
    - Edge Cases:  
      - SMS length exceeding 160 characters (not explicitly split here, but noted).  
      - Empty alerts list (should not occur due to prior IF).  
      - Time format assumptions for slicing time strings.  
    - Version: 2.

#### 2.7 SMS Sending

- **Overview:** Sends the formatted SMS alert using Twilio credentials and configured recipient numbers.
- **Nodes Involved:**  
  - Send Weather SMS (Twilio)
- **Node Details:**

  - **Send Weather SMS**  
    - Type: Twilio node  
    - Configuration:  
      - Message content from formatted alert JSON property `message`.  
      - Recipients set within node credentials or parameters (not visible in JSON).  
      - Uses Twilio credentials linked in n8n (Account SID and Auth Token).  
    - Inputs: Formatted SMS JSON.  
    - Outputs: Status of SMS send operation.  
    - Edge Cases:  
      - Invalid Twilio credentials.  
      - Insufficient balance or unverified phone numbers.  
      - SMS message size limits or format issues.  
      - Network or API downtime.  
    - Version: 1.

#### 2.8 Logging

- **Overview:** Logs confirmation of successful alert sending along with metadata for audit or monitoring.
- **Nodes Involved:**  
  - Log Alert Sent (Code)
- **Node Details:**

  - **Log Alert Sent**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Reads formatted alert data from previous node.  
      - Logs success messages with location, alert type, urgency.  
      - Returns JSON with status, summary string, urgency, and timestamp (ISO).  
    - Inputs: Output from Send Weather SMS.  
    - Outputs: JSON status object.  
    - Edge Cases: None critical; logging failures do not affect workflow outcome.  
    - Version: 2.

---

### 3. Summary Table

| Node Name            | Node Type            | Functional Role              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                                |
|----------------------|----------------------|-----------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions    | Sticky Note          | Setup and user guidance      | None                         | None                        | 🌤️ **SETUP REQUIRED:** 1. OpenWeather API key, 2. Twilio setup, 3. Recipients, 4. Alert conditions, runs every 6 hours                      |
| Check Every 6 Hours   | Cron Trigger         | Scheduled workflow trigger  | None                         | Get Current Weather, Get Weather Forecast |                                                                                                                                            |
| Get Current Weather   | HTTP Request         | Fetch current weather data  | Check Every 6 Hours           | Analyze Weather Data         |                                                                                                                                            |
| Get Weather Forecast  | HTTP Request         | Fetch weather forecast data | Check Every 6 Hours           | Analyze Weather Data         |                                                                                                                                            |
| Analyze Weather Data  | Code (JavaScript)    | Process and analyze weather | Get Current Weather, Get Weather Forecast | Alert Needed?              |                                                                                                                                            |
| Alert Needed?         | IF                   | Check if alerts exist       | Analyze Weather Data          | Format SMS Alert             |                                                                                                                                            |
| Format SMS Alert      | Code (JavaScript)    | Compose SMS message         | Alert Needed? (True branch)   | Send Weather SMS             |                                                                                                                                            |
| Send Weather SMS      | Twilio               | Send SMS alert              | Format SMS Alert              | Log Alert Sent               |                                                                                                                                            |
| Log Alert Sent        | Code (JavaScript)    | Log alert sending status    | Send Weather SMS              | None                        |                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note "Setup Instructions"**  
   - Content: Include setup instructions for OpenWeather API, Twilio account, recipients, alert customization, and schedule info.  
   - Position: Top-left for visibility.

2. **Add Cron Node "Check Every 6 Hours"**  
   - Type: Cron Trigger  
   - Set to trigger every 6 hours (default configuration).  
   - No inputs.

3. **Add HTTP Request Node "Get Current Weather"**  
   - URL: `https://api.openweathermap.org/data/2.5/weather`  
   - Query Parameters:  
     - `q`: City and country (e.g., "New York,US")  
     - `appid`: Your OpenWeather API key (replace placeholder)  
     - `units`: "imperial"  
   - Connect output of "Check Every 6 Hours" → "Get Current Weather".

4. **Add HTTP Request Node "Get Weather Forecast"**  
   - URL: `https://api.openweathermap.org/data/2.5/forecast`  
   - Query Parameters same as above plus:  
     - `cnt`: "8" (limit forecast entries to 8)  
   - Connect output of "Check Every 6 Hours" → "Get Weather Forecast".

5. **Add Code Node "Analyze Weather Data"**  
   - Language: JavaScript  
   - Code: See below (adapt as needed):  
     ```javascript
     const currentWeather = $('Get Current Weather').item.json;
     const forecast = $('Get Weather Forecast').item.json;
     
     const current = {
       temperature: Math.round(currentWeather.main.temp),
       feels_like: Math.round(currentWeather.main.feels_like),
       humidity: currentWeather.main.humidity,
       description: currentWeather.weather[0].description,
       main_condition: currentWeather.weather[0].main.toLowerCase(),
       wind_speed: Math.round(currentWeather.wind.speed),
       city: currentWeather.name,
       country: currentWeather.sys.country
     };

     const upcoming = [];
     forecast.list.slice(0, 8).forEach(item => {
       upcoming.push({
         time: new Date(item.dt * 1000).toLocaleString(),
         temp: Math.round(item.main.temp),
         condition: item.weather[0].main.toLowerCase(),
         description: item.weather[0].description,
         rain_probability: Math.round((item.pop || 0) * 100)
       });
     });

     const alerts = [];

     if (current.temperature >= 95) {
       alerts.push(`🔥 EXTREME HEAT: ${current.temperature}°F (feels like ${current.feels_like}°F)`);
     } else if (current.temperature <= 20) {
       alerts.push(`🥶 EXTREME COLD: ${current.temperature}°F (feels like ${current.feels_like}°F)`);
     }

     if (['thunderstorm', 'tornado'].includes(current.main_condition)) {
       alerts.push(`⛈️ SEVERE WEATHER: ${current.description}`);
     }

     if (['rain', 'drizzle'].includes(current.main_condition)) {
       alerts.push(`🌧️ RAIN ALERT: ${current.description}`);
     } else if (current.main_condition === 'snow') {
       alerts.push(`❄️ SNOW ALERT: ${current.description}`);
     }

     if (current.wind_speed >= 25) {
       alerts.push(`💨 HIGH WINDS: ${current.wind_speed} mph`);
     }

     const upcomingBad = upcoming.filter(item =>
       ['thunderstorm', 'snow', 'rain'].includes(item.condition) ||
       item.rain_probability >= 70
     );

     if (upcomingBad.length > 0) {
       alerts.push(`⚠️ UPCOMING: ${upcomingBad[0].description} at ${upcomingBad[0].time.split(',')[1]} (${upcomingBad[0].rain_probability}% chance)`);
     }

     const weatherData = {
       location: `${current.city}, ${current.country}`,
       current_weather: current,
       forecast_24h: upcoming,
       alerts: alerts,
       has_alerts: alerts.length > 0,
       alert_count: alerts.length,
       timestamp: new Date().toLocaleString()
     };

     console.log(`Weather check for ${weatherData.location}: ${weatherData.alert_count} alerts found`);

     return {
       json: weatherData
     };
     ```
   - Connect outputs of both weather HTTP nodes → "Analyze Weather Data".

6. **Add IF Node "Alert Needed?"**  
   - Condition: Check if `{{$json.has_alerts}}` equals `true`.  
   - Connect output of "Analyze Weather Data" → "Alert Needed?".

7. **Add Code Node "Format SMS Alert"**  
   - JavaScript to compose SMS message:  
     ```javascript
     const weatherData = $input.first().json;

     let smsMessage = `🌤️ WEATHER ALERT - ${weatherData.location}\n\n`;

     smsMessage += `NOW: ${weatherData.current_weather.temperature}°F, ${weatherData.current_weather.description}\n\n`;

     smsMessage += `🚨 ALERTS (${weatherData.alert_count}):\n`;
     weatherData.alerts.forEach(alert => {
       smsMessage += `${alert}\n`;
     });

     const next3Hours = weatherData.forecast_24h.slice(0, 3);
     smsMessage += `\n📅 NEXT 3 HOURS:\n`;
     next3Hours.forEach(forecast => {
       const time = forecast.time.split(' ')[1];
       smsMessage += `${time}: ${forecast.temp}°F, ${forecast.description}\n`;
     });

     smsMessage += `\n⏰ Alert sent: ${weatherData.timestamp}`;

     const formattedAlert = {
       message: smsMessage,
       location: weatherData.location,
       alert_type: weatherData.alerts[0] ? weatherData.alerts[0].split(':')[0] : 'GENERAL',
       urgency: weatherData.alerts.some(alert =>
         alert.includes('EXTREME') || alert.includes('SEVERE')
       ) ? 'HIGH' : 'MEDIUM'
     };

     console.log(`Formatted SMS alert (${formattedAlert.urgency} urgency):`, formattedAlert.alert_type);

     return {
       json: formattedAlert
     };
     ```
   - Connect True output of "Alert Needed?" → "Format SMS Alert".

8. **Add Twilio Node "Send Weather SMS"**  
   - Configure with Twilio credentials (Account SID and Auth Token).  
   - Set recipient phone numbers (format: +1234567890).  
   - Message: Set to `{{$json.message}}` from previous node.  
   - Connect output of "Format SMS Alert" → "Send Weather SMS".

9. **Add Code Node "Log Alert Sent"**  
   - JavaScript to log alert sending:  
     ```javascript
     const alertData = $('Format SMS Alert').item.json;

     console.log(`✅ Weather alert sent successfully!`);
     console.log(`📍 Location: ${alertData.location}`);
     console.log(`⚠️ Alert Type: ${alertData.alert_type}`);
     console.log(`🔥 Urgency: ${alertData.urgency}`);

     return {
       json: {
         status: 'sent',
         alert_summary: `Weather alert sent for ${alertData.location} - ${alertData.alert_type}`,
         urgency: alertData.urgency,
         timestamp: new Date().toISOString()
       }
     };
     ```
   - Connect output of "Send Weather SMS" → "Log Alert Sent".

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                          |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| OpenWeather API key is free with a 1000 calls/day limit. Sign up at https://openweathermap.org                  | Setup Instructions sticky note           |
| Twilio account required for SMS sending, including account SID, Auth Token, and a purchased phone number       | Setup Instructions sticky note           |
| Phone numbers must be formatted in international format, e.g., +1234567890                                     | Setup Instructions sticky note           |
| Workflow runs every 6 hours to provide timely alerts without overloading API quotas                            | Setup Instructions sticky note           |
| Alert conditions currently include rain, snow, extreme temperatures, high winds, thunderstorm, and tornadoes  | Setup Instructions sticky note           |
| SMS messages are constructed to be concise but may exceed 160 characters; consider splitting if needed         | Format SMS Alert node comment             |
| Logging provides confirmation and metadata for sent alerts, useful for monitoring and debugging                | Log Alert Sent node comment               |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. All data processed is legal and public. This workflow complies with content policies and contains no illegal, offensive, or protected elements.