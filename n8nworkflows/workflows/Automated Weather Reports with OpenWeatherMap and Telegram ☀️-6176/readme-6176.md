Automated Weather Reports with OpenWeatherMap and Telegram ☀️

https://n8nworkflows.xyz/workflows/automated-weather-reports-with-openweathermap-and-telegram----6176


# Automated Weather Reports with OpenWeatherMap and Telegram ☀️

### 1. Workflow Overview

This workflow automates the generation and delivery of weather reports to a Telegram chat using OpenWeatherMap data. It is designed to run every 12 hours, fetch both current weather and a 5-day forecast for a specified location, compile a human-friendly markdown report, and send it via Telegram.

**Logical blocks:**

- **1.1 Input Initialization:** Set location coordinates and Telegram chat ID from environment variables.
- **1.2 Weather Data Retrieval:** Fetch current weather and 5-day forecast from OpenWeatherMap API using coordinates.
- **1.3 Data Aggregation:** Merge current and forecast weather data into a single dataset.
- **1.4 Report Preparation:** Process raw weather data to create a formatted markdown report suitable for Telegram.
- **1.5 Report Delivery:** Send the markdown weather report as a message to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block initializes key input variables such as latitude, longitude, and Telegram chat ID from environment variables for use throughout the workflow.

**Nodes Involved:**  
- Schedule Trigger  
- Set Location  
- Sticky Note (Set inputs)

**Node Details:**  

- **Schedule Trigger**  
  - Type: Trigger node for scheduled executions  
  - Configuration: Runs every 12 hours (interval in hours)  
  - Inputs: None (start of workflow)  
  - Outputs: Triggers “Set Location” node  
  - Edge cases: Scheduler downtime or misconfiguration could prevent execution  

- **Set Location**  
  - Type: Set node (assigns static or dynamic variables)  
  - Configuration: Assigns three variables from environment variables:  
    - `lat` (latitude)  
    - `long` (longitude)  
    - `telegram_chat_id` (Telegram destination chat)  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Data object with `lat`, `long`, `telegram_chat_id` fields passed to weather API nodes  
  - Edge cases: Missing or invalid environment variables would cause failure downstream  

- **Sticky Note (Set inputs)**  
  - Functional Role: Documentation for users explaining required inputs (`lat`, `long`, `telegram_chat_id`)  

#### 2.2 Weather Data Retrieval

**Overview:**  
Fetches current weather conditions and a 5-day weather forecast for the specified coordinates via OpenWeatherMap API.

**Nodes Involved:**  
- Weather: Current  
- Weather: 5Days  
- Sticky Note1 (Current Weather)  
- Sticky Note2 (5-Day Weather)

**Node Details:**  

- **Weather: Current**  
  - Type: OpenWeatherMap API node (current weather)  
  - Configuration:  
    - Language: English  
    - Location selection: Coordinates (`lat`, `long` from input)  
    - Credentials: OpenWeatherMap API key configured  
  - Inputs: Output of “Set Location”  
  - Outputs: Current weather JSON data  
  - Edge cases: API key invalid or quota exceeded; network timeout; invalid coordinates  

- **Weather: 5Days**  
  - Type: OpenWeatherMap API node (5-day, 3-hour interval forecast)  
  - Configuration: Similar to “Weather: Current” but operation set to “5DayForecast”  
  - Inputs: Output of “Set Location”  
  - Outputs: 5-day forecast JSON data  
  - Edge cases: Same as “Weather: Current”  

- **Sticky Note1 (Current Weather)** and **Sticky Note2 (5-Day Weather)**  
  - Functional Role: Visual grouping/documentation for respective API calls  

#### 2.3 Data Aggregation

**Overview:**  
Merges the current weather and 5-day forecast data into a single combined dataset for report generation.

**Nodes Involved:**  
- Merge

**Node Details:**  

- **Merge**  
  - Type: Merge node  
  - Configuration: Mode set to “Combine” by position, meaning it combines the two inputs (current and forecast) into a single output array with two items  
  - Inputs:  
    - Current weather data (index 0)  
    - 5-day forecast data (index 1)  
  - Outputs: Combined JSON object with both weather data sets  
  - Edge cases: If one input is missing or delayed, merge may produce incomplete data  

#### 2.4 Report Preparation

**Overview:**  
Transforms the merged weather data into a markdown-formatted report optimized for Telegram messaging. It aggregates daily highs/lows, picks representative weather icons, and formats a summary.

**Nodes Involved:**  
- Report: Prepare  
- Sticky Note3 (Report)

**Node Details:**  

- **Report: Prepare**  
  - Type: Code node (JavaScript)  
  - Configuration: Custom JavaScript code that:  
    - Parses input data from OpenWeatherMap (both current and forecast)  
    - Aggregates forecast data by day calculating daily low/high temperatures and selects weather icons closest to noon  
    - Formats text with temperature, condition, humidity, wind speed, and a 5-day forecast using Markdown syntax  
    - Uses emoji to visually represent weather conditions  
    - Escapes Markdown special characters to ensure correct rendering  
  - Inputs: Merged weather data  
  - Outputs: JSON with one key `markdown_report` containing the formatted markdown string  
  - Edge cases: Unexpected API response shape or missing fields could cause code errors; Markdown escaping may fail if unexpected characters present  

- **Sticky Note3 (Report)**  
  - Functional Role: Documentation label for the processing node  

#### 2.5 Report Delivery

**Overview:**  
Sends the markdown weather report to the specified Telegram chat using the Telegram Bot API.

**Nodes Involved:**  
- Send a text message  
- Sticky Note4 (Send)

**Node Details:**  

- **Send a text message**  
  - Type: Telegram node  
  - Configuration:  
    - Message text: Uses expression to pull `markdown_report` from “Report: Prepare” output  
    - Chat ID: Pulled from “Set Location” node output  
    - Additional fields: Markdown parse mode enabled, attribution disabled  
    - Credentials: Telegram Bot API OAuth2 credentials configured  
  - Inputs: Markdown report from “Report: Prepare”  
  - Outputs: Telegram message delivery confirmation  
  - Edge cases: Invalid chat ID or bot permissions; Telegram API rate limiting; network issues  

- **Sticky Note4 (Send)**  
  - Functional Role: Documentation label for the sending node  

---

### 3. Summary Table

| Node Name           | Node Type                | Functional Role               | Input Node(s)           | Output Node(s)            | Sticky Note                                      |
|---------------------|--------------------------|------------------------------|------------------------|---------------------------|-------------------------------------------------|
| Schedule Trigger    | Schedule Trigger          | Initiates workflow every 12h | None                   | Set Location              |                                                 |
| Set Location        | Set                      | Sets lat, long, telegram_chat_id from env | Schedule Trigger        | Weather: Current, Weather: 5Days | ## Set inputs - lat - long - telegram_chat_id    |
| Weather: Current    | OpenWeatherMap (Current) | Fetches current weather data | Set Location            | Merge                     | ## Current Weather                               |
| Weather: 5Days      | OpenWeatherMap (5-Day)   | Fetches 5-day weather forecast | Set Location            | Merge                     | ## 5-Day Weather                                 |
| Merge               | Merge                    | Combines current and forecast data | Weather: Current, Weather: 5Days | Report: Prepare           |                                                 |
| Report: Prepare     | Code                     | Creates markdown weather report | Merge                   | Send a text message       | ## Report                                        |
| Send a text message | Telegram                 | Sends report markdown to Telegram chat | Report: Prepare          | None                      | ## Send                                          |
| Sticky Note         | Sticky Note              | Documentation on inputs       | None                   | None                      | ## Set inputs \n- lat\n- long\n- telegram_chat_id |
| Sticky Note1        | Sticky Note              | Documentation for current weather | None                   | None                      | ## Current Weather                               |
| Sticky Note2        | Sticky Note              | Documentation for 5-day forecast | None                   | None                      | ## 5-Day Weather                                 |
| Sticky Note3        | Sticky Note              | Documentation for report node | None                   | None                      | ## Report                                        |
| Sticky Note4        | Sticky Note              | Documentation for send node  | None                   | None                      | ## Send                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to run every 12 hours (hours interval = 12)  
   - This node will start the workflow automatically on schedule  

2. **Create a Set node named "Set Location"**  
   - Add 3 string fields assigned via expressions from environment variables:  
     - `lat` = `={{ $env.lat }}`  
     - `long` = `={{ $env.long }}`  
     - `telegram_chat_id` = `={{ $env.telegram_chat_id }}`  
   - Connect Schedule Trigger output to this node's input  

3. **Create OpenWeatherMap node named "Weather: Current"**  
   - Operation: Current Weather (default)  
   - Language: English  
   - Location selection: Coordinates  
   - Latitude: `={{ $json.lat }}`  
   - Longitude: `={{ $json.long }}`  
   - Configure credentials with valid OpenWeatherMap API key  
   - Connect "Set Location" output to this node  

4. **Create OpenWeatherMap node named "Weather: 5Days"**  
   - Operation: 5-Day Forecast  
   - Language: English  
   - Location selection: Coordinates  
   - Latitude: `={{ $json.lat }}`  
   - Longitude: `={{ $json.long }}`  
   - Use the same OpenWeatherMap credentials as above  
   - Connect "Set Location" output to this node  

5. **Create Merge node named "Merge"**  
   - Mode: Combine  
   - Combine by: Position  
   - Connect "Weather: Current" output (main output 0) to input 0 of "Merge"  
   - Connect "Weather: 5Days" output (main output 0) to input 1 of "Merge"  

6. **Create Code node named "Report: Prepare"**  
   - Paste the JavaScript code provided below into the code editor:  

```javascript
const data = $json;                // full API response (array with two items expected)
const current = data[0];           // current weather data
const forecast = data[1];          // 5-day forecast

const city = forecast.city;        // city info from forecast
const blocks = forecast.list ?? []; // forecast blocks

// Helper functions
const esc = t => t.toString().replace(/[_*[\\]()~`>#+\\-=|{}.!]/g, '\\\\$&');
const fmtT = t => `${Math.round(t)}°C`;
const fmtDay = ts => new Date(ts * 1000)
    .toLocaleDateString('en-US', { weekday:'short', month:'short', day:'numeric' });
const emoji = id => (
    id === 800 ? '☀️' :
    id >= 801 && id <= 804 ? '☁️' :
    id >= 200 && id <= 232 ? '⛈️' :
    id >= 300 && id <= 321 ? '🌦️' :
    id >= 500 && id <= 531 ? '🌧️' :
    id >= 600 && id <= 622 ? '❄️'  :
    id >= 700 && id <= 781 ? '🌫️'  : '🌤️'
);

// Aggregate daily lows/highs and pick noon icon
const days = {};
for (const b of blocks) {
    const d = new Date(b.dt * 1000);
    const key = d.toISOString().slice(0,10);
    const bucket = days[key] ?? (days[key] = {
        tsNoon : b.dt,
        iconId : b.weather[0].id,
        desc   : b.weather[0].main,
        lo     : b.main.temp_min,
        hi     : b.main.temp_max
    });

    bucket.lo = Math.min(bucket.lo, b.main.temp_min);
    bucket.hi = Math.max(bucket.hi, b.main.temp_max);

    const hr = d.getHours();
    if (Math.abs(hr - 12) < Math.abs(new Date(bucket.tsNoon*1000).getHours() - 12)) {
        bucket.tsNoon = b.dt;
        bucket.iconId = b.weather[0].id;
        bucket.desc   = b.weather[0].main;
    }
}

const dayKeys = Object.keys(days).sort().slice(0,5);
const today = days[dayKeys[0]];

const ln = '\n';
const hr = '\n──────────\n';

let md  = `*${esc(city.name)}, ${esc(city.country)} – Weather*${hr}`;
md += '*Now*\n';
md += `🌡️ Temp: \`${fmtT(current.list[0].main.temp)}\`  (feels like \`${fmtT(current.list[0].main.feels_like)}\`)${ln}`;
md += `📉 Low / High today: \`${fmtT(today.lo)} – ${fmtT(today.hi)}\`${ln}`;
md += `🛰️ Condition: \`${esc(current.list[0].weather[0].description)}\`${ln}`;
md += `💧 Humidity: \`${current.list[0].main.humidity}%\`    `;
md += `💨 Wind: \`${Math.round(current.list[0].wind.speed*3.6)} km/h\`${hr}`;

md += '*5-Day Forecast*' + ln;
for (const k of dayKeys) {
    const d = days[k];
    md += `➡️ *${esc(fmtDay(d.tsNoon))}* `;
    md += `\`${fmtT(d.lo)} – ${fmtT(d.hi)}\` `;
    md += `${emoji(d.iconId)} _${esc(d.desc)}_\n`;
}

return { json: { markdown_report: md }};
```

   - Connect "Merge" output to this node  

7. **Create Telegram node named "Send a text message"**  
   - Set parameters:  
     - Text: `={{ $json.markdown_report }}` (expression referencing the previous node output)  
     - Chat ID: `={{ $('Set Location').item.json.telegram_chat_id }}` (expression referencing location node)  
     - Additional Fields:  
       - Parse mode: Markdown  
       - Append attribution: false  
   - Configure Telegram API credentials (Bot token with chat permissions)  
   - Connect "Report: Prepare" output to this node  

8. **Optionally, add Sticky Note nodes** for documentation at appropriate workflow positions, labeling inputs, API nodes, report generation, and send step for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| All input variables must be set as environment variables: `lat`, `long`, `telegram_chat_id`.    | Environment setup                                    |
| OpenWeatherMap API key and Telegram Bot API credentials must be properly configured and valid.  | Credential setup                                    |
| Markdown escaping in the code node is essential to prevent Telegram formatting issues.           | Report formatting                                   |
| The workflow runs every 12 hours by default; adjust Schedule Trigger as needed for frequency.    | Scheduling                                          |
| For troubleshooting, check API quotas and permissions for both OpenWeatherMap and Telegram APIs. | Error handling                                      |

---

**Disclaimer:** The provided workflow is an automated process built with n8n, adhering strictly to content policies and handling only legal, public data.