Monitor multi-city weather with OpenWeatherMap, GPT-4o-mini, and Discord

https://n8nworkflows.xyz/workflows/monitor-multi-city-weather-with-openweathermap--gpt-4o-mini--and-discord-12653


# Monitor multi-city weather with OpenWeatherMap, GPT-4o-mini, and Discord

## 1. Workflow Overview

**Purpose:** Monitor weather conditions across multiple cities on a fixed schedule. For each city, the workflow fetches **current weather**, **5-day forecast**, and **air quality (AQI)** from OpenWeatherMap, computes a **Comfort Index** and risk **alerts**, asks **GPT-4o-mini** for a short comparative briefing, then sends either an **urgent alert** or a **routine report** to **Discord**, and finally **archives** the result to **Google Sheets**.

**Target use cases:**
- Multi-city operations monitoring (travel, field teams, distributed offices)
- Daily/periodic weather briefings in Discord
- Historical logging for trend analysis in Sheets

### 1.1 Schedule & Location Setup (Part 1)
Runs every 6 hours and defines the list of monitored cities (lat/lon/timezone), then iterates city-by-city.

### 1.2 Parallel Data Collection (Part 2)
For each location, pulls:
- Current weather (OpenWeatherMap node)
- 5-day forecast (OpenWeatherMap node)
- AQI (direct HTTP call to OpenWeatherMap air_pollution endpoint)

### 1.3 Local Analytics + AI Briefing (Part 3)
Computes Comfort Index + alert flags in JavaScript, aggregates all cities into one payload, then requests GPT-4o-mini to write a concise comparative briefing.

### 1.4 Routing, Delivery & Logging (Part 4)
Formats a Discord-ready report, routes to either “Alert” or “Report” channel based on detected alerts, then appends to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Location Setup (Part 1)
**Overview:** Triggers every 6 hours, defines monitored locations as an array, then splits the list into per-city items for downstream API calls.  
**Nodes involved:** `Schedule Trigger`, `Set Monitoring Locations`, `Split Locations`

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger — entry point; starts executions periodically.
- **Configuration (interpreted):** Runs **every 6 hours**.
- **Connections:**
  - **Output →** `Set Monitoring Locations`
- **Failure/edge cases:**
  - Timezone is the n8n instance timezone unless otherwise configured globally.
  - Missed executions can occur if n8n is down; no catch-up by default.

#### Node: Set Monitoring Locations
- **Type / role:** Set — creates the list of cities to monitor.
- **Configuration (interpreted):**
  - Mode: outputs **raw JSON** which is an **array of objects**:
    - `city`, `lat`, `lon`, `timezone`
  - Current default cities: Tokyo, New York, London.
- **Key expressions/variables:**
  - `jsonOutput` is an expression returning an array literal.
- **Connections:**
  - **Input ←** `Schedule Trigger`
  - **Output →** `Split Locations`
- **Failure/edge cases:**
  - Invalid JSON or trailing commas will break execution.
  - Lat/lon are strings here; downstream nodes accept them, but numeric is safer.
  - `timezone` is not used later (currently informational only).

#### Node: Split Locations
- **Type / role:** Split Out — iterates over the locations list, producing one item per city.
- **Configuration (interpreted):**
  - “Field to split out” is set to `=` (this indicates it is splitting the **incoming array itself** rather than a named field; effectively “split root array” behavior).
- **Connections:**
  - **Input ←** `Set Monitoring Locations`
  - **Output →** `Get Current Weather`, `Get 5-Day Forecast`, `Get Air Quality Index` (fan-out in parallel)
- **Failure/edge cases:**
  - If the previous node does not output an array, split will fail or produce unexpected items.
  - If the split field configuration is changed incorrectly, downstream nodes will not receive `lat`/`lon`.

---

### Block 2 — Data Collection (Part 2)
**Overview:** For each city item, calls weather/forecast/AQI endpoints and merges results so analysis can be done on a single combined item stream.  
**Nodes involved:** `Get Current Weather`, `Get 5-Day Forecast`, `Get Air Quality Index`, `Merge Weather Data`, `Merge with Air Quality`

#### Node: Get Current Weather
- **Type / role:** OpenWeatherMap — fetches current conditions by coordinates.
- **Configuration (interpreted):**
  - Location selection: **coordinates**
  - Latitude: `{{ $json.lat }}`
  - Longitude: `{{ $json.lon }}`
  - Language: English
- **Connections:**
  - **Input ←** `Split Locations`
  - **Output →** `Merge Weather Data` (input index 0)
- **Failure/edge cases:**
  - OpenWeatherMap credential/auth errors.
  - Rate limiting if many cities or frequent schedule.
  - Missing/invalid lat/lon causes API errors.

#### Node: Get 5-Day Forecast
- **Type / role:** OpenWeatherMap — fetches 5-day forecast by coordinates.
- **Configuration (interpreted):**
  - Operation: **5DayForecast**
  - Coordinates via `{{ $json.lat }}`, `{{ $json.lon }}`
  - Language: English
- **Connections:**
  - **Input ←** `Split Locations`
  - **Output →** `Merge Weather Data` (input index 1)
- **Failure/edge cases:**
  - Same as current weather (auth/rate/invalid coords).
  - Forecast payload can be large; may affect memory if many cities.

#### Node: Get Air Quality Index
- **Type / role:** HTTP Request — calls OpenWeatherMap Air Pollution endpoint (AQI).
- **Configuration (interpreted):**
  - Method: GET (implicit)
  - URL (expression):
    - `http://api.openweathermap.org/data/2.5/air_pollution?lat={{ $json.lat }}&lon={{ $json.lon }}&appid={{ $credentials.openWeatherMapApi.apiKey }}`
  - Authentication: **Predefined credential type** `openWeatherMapApi`
- **Connections:**
  - **Input ←** `Split Locations`
  - **Output →** `Merge with Air Quality` (input index 1)
- **Version-specific notes:**
  - Node typeVersion 4.2 uses modern HTTP Request configuration; ensure “Authentication: predefinedCredentialType” exists in your n8n version.
- **Failure/edge cases:**
  - Using `http://` instead of `https://` may be blocked in some environments or leak metadata; prefer HTTPS.
  - `$credentials.openWeatherMapApi.apiKey` must exist and be accessible; otherwise expression fails.
  - API may return empty `list` depending on upstream issues.

#### Node: Merge Weather Data
- **Type / role:** Merge — combines current weather and forecast into one item.
- **Configuration (interpreted):**
  - Mode: **Combine**
  - Combine by: **Position**
- **Connections:**
  - **Input 0 ←** `Get Current Weather`
  - **Input 1 ←** `Get 5-Day Forecast`
  - **Output →** `Merge with Air Quality` (input index 0)
- **Failure/edge cases:**
  - Position-based merge assumes both branches return items in identical order/count; if one API errors or returns fewer items, merges misalign.

#### Node: Merge with Air Quality
- **Type / role:** Merge — combines merged weather data with AQI response.
- **Configuration (interpreted):**
  - Mode: **Combine**
  - Combine by: **Position**
- **Connections:**
  - **Input 0 ←** `Merge Weather Data`
  - **Input 1 ←** `Get Air Quality Index`
  - **Output →** `Process and Analyze Data`
- **Failure/edge cases:**
  - Same positional alignment risk as above (very important in multi-branch fan-out).

---

### Block 3 — AI Analysis (Part 3)
**Overview:** Computes derived metrics and alert flags per city, aggregates all city results into one payload, then asks OpenAI to produce a concise comparative weather briefing.  
**Nodes involved:** `Process and Analyze Data`, `Aggregate All Locations`, `AI Weather Analysis`

#### Node: Process and Analyze Data
- **Type / role:** Code — derives Comfort Index, alert flags, and normalizes the city summary object.
- **Configuration (interpreted):**
  - JavaScript loops over all incoming items and emits normalized `json` per location with:
    - `city`, `country`, `timestamp`
    - `current`: temperature, feelsLike, humidity, windSpeed, weather, weatherMain
    - `airQuality`: aqi + label (Good/Fair/Moderate/Poor/Very Poor)
    - `analysis`: comfortIndex (0–100), alerts array, hasAlerts boolean
- **Key logic details / variables:**
  - Temperature: `current.main?.temp || 0`
  - AQI: `current.list?.[0]?.main?.aqi || 1` (expects AQI payload merged into same `current` object)
  - Comfort Index formula:
    - `100 - |temp - 22| * 2 - (humidity > 70 ? humidity - 70 : 0)` then clamped 0–100
  - Alerts triggered when:
    - temp > 35, temp < 0, windSpeed > 15, AQI >= 4, Thunderstorm
- **Connections:**
  - **Input ←** `Merge with Air Quality`
  - **Output →** `Aggregate All Locations`
- **Failure/edge cases:**
  - Unit mismatch risk: OpenWeatherMap temps can be Kelvin by default depending on node settings. If not set to metric/°C, comfort/thresholds become incorrect.
  - Merge-shape ambiguity: this code assumes both weather and AQI fields are accessible under `item.json` after merge. If the Merge node nests data differently, `current.list` might not exist.
  - If `name` is missing, city becomes “Unknown” (it does not reuse the original city label from `Set Monitoring Locations`).

#### Node: Aggregate All Locations
- **Type / role:** Aggregate — collects all per-city items into a single array for a single AI request.
- **Configuration (interpreted):**
  - Operation: **aggregate all item data** into `data` array.
- **Connections:**
  - **Input ←** `Process and Analyze Data`
  - **Output →** `AI Weather Analysis`
- **Failure/edge cases:**
  - Large city lists increase payload size; may exceed OpenAI token limits when stringified.

#### Node: AI Weather Analysis
- **Type / role:** HTTP Request — calls OpenAI Chat Completions API.
- **Configuration (interpreted):**
  - URL: `https://api.openai.com/v1/chat/completions`
  - Method: POST
  - Auth: predefined credential type `openAiApi`
  - JSON body:
    - model: `gpt-4o-mini`
    - system: “professional meteorologist… under 200 words with emojis”
    - user: `Analyze this weather data: {{ JSON.stringify($json.data) }}`
- **Connections:**
  - **Input ←** `Aggregate All Locations`
  - **Output →** `Format Weather Report`
- **Failure/edge cases:**
  - OpenAI auth/permission errors (wrong key, missing billing).
  - Token/size errors if too many cities or verbose payload.
  - Response schema differences: code later expects `choices[0].message.content`.

---

### Block 4 — Delivery & Logging (Part 4)
**Overview:** Builds a human-readable multi-city report, checks whether any location has alerts, posts to the appropriate Discord webhook, and appends the output to Google Sheets.  
**Nodes involved:** `Format Weather Report`, `Check for Alerts`, `Send Discord Alert`, `Send Discord Report`, `Log to Google Sheets`

#### Node: Format Weather Report
- **Type / role:** Code — generates final Markdown report and alert flag for routing.
- **Configuration (interpreted):**
  - Reads aggregated data from: `$('Aggregate All Locations').first().json.data`
  - Reads AI text from current input (OpenAI response): `choices[0].message.content`
  - Builds a report with:
    - Header “WEATHER INTELLIGENCE REPORT”
    - Per city lines: temperature, comfort index, AQI label, alert list
    - AI analysis appended
  - Outputs:
    - `report` (string)
    - `hasAlerts` (boolean: any location has alerts)
- **Connections:**
  - **Input ←** `AI Weather Analysis`
  - **Output →** `Check for Alerts`
- **Failure/edge cases:**
  - If OpenAI call fails or returns unexpected structure, AI text becomes “Analysis unavailable”.
  - If `Aggregate All Locations` is empty, the report loop renders nothing; `hasAlerts` becomes false.

#### Node: Check for Alerts
- **Type / role:** IF — routes execution depending on `hasAlerts`.
- **Configuration (interpreted):**
  - Condition: `{{ $json.hasAlerts }}` equals `true` (strict boolean comparison)
- **Connections:**
  - **Input ←** `Format Weather Report`
  - **True →** `Send Discord Alert`
  - **False →** `Send Discord Report`
- **Failure/edge cases:**
  - If `hasAlerts` is not a boolean (e.g., string `"true"`), strict validation may route incorrectly.

#### Node: Send Discord Alert
- **Type / role:** Discord — posts urgent alert message via webhook.
- **Configuration (interpreted):**
  - Authentication: webhook
  - Content:
    - `🚨 **URGENT WEATHER ALERT**` + the generated report
- **Connections:**
  - **Input ←** `Check for Alerts` (true branch)
  - **Output →** `Log to Google Sheets`
- **Failure/edge cases:**
  - Webhook misconfigured/rotated; 401/404 from Discord.
  - Discord message length limit (report may need trimming if many cities).

#### Node: Send Discord Report
- **Type / role:** Discord — posts routine briefing via webhook.
- **Configuration (interpreted):**
  - Authentication: webhook
  - Content: the generated report
- **Connections:**
  - **Input ←** `Check for Alerts` (false branch)
  - **Output →** `Log to Google Sheets`
- **Failure/edge cases:**
  - Same webhook/message length risks as alert node.

#### Node: Log to Google Sheets
- **Type / role:** Google Sheets — appends a row to a spreadsheet to archive results.
- **Configuration (interpreted):**
  - Operation: **append**
  - Spreadsheet/document ID: placeholder `YOUR_SPREADSHEET_ID`
  - Sheet name: `Sheet1`
  - No explicit column mapping shown; uses default behavior (depends on node UI and incoming fields).
- **Connections:**
  - **Input ←** `Send Discord Alert` and `Send Discord Report`
  - **Output:** none
- **Failure/edge cases:**
  - Missing/invalid Google OAuth credentials.
  - Spreadsheet ID wrong or service account/user lacks access.
  - If no matching columns, append can fail or append partial/blank data. You typically must map `report`, `hasAlerts`, timestamp, etc., to columns.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | Documentation / overview | — | — | ## How it works / Setup steps (global description of credentials, locations, Sheets, Discord) |
| Part 1 Sticky | Sticky Note | Documentation for Part 1 | — | — | ## PART 1: Setup & Iteration — Trigger the flow and define the list of cities to monitor using coordinates. |
| Part 2 Sticky | Sticky Note | Documentation for Part 2 | — | — | ## PART 2: Data Collection — Fetches current weather, forecasts, and air pollution data simultaneously for each location. |
| Part 3 Sticky | Sticky Note | Documentation for Part 3 | — | — | ## PART 3: AI Analysis — Calculates comfort metrics and uses GPT-4 to generate human-like weather briefings. |
| Part 4 Sticky | Sticky Note | Documentation for Part 4 | — | — | ## PART 4: Delivery & Logging — Smart routing: sends urgent alerts or routine reports to Discord and logs results to Google Sheets. |
| Schedule Trigger | Schedule Trigger | Start workflow every 6 hours | — | Set Monitoring Locations | ## PART 1: Setup & Iteration — Trigger the flow and define the list of cities to monitor using coordinates. |
| Set Monitoring Locations | Set | Define city list (lat/lon/timezone) | Schedule Trigger | Split Locations | ## PART 1: Setup & Iteration — Trigger the flow and define the list of cities to monitor using coordinates. |
| Split Locations | Split Out | Convert array of cities into items | Set Monitoring Locations | Get Current Weather; Get 5-Day Forecast; Get Air Quality Index | ## PART 1: Setup & Iteration — Trigger the flow and define the list of cities to monitor using coordinates. |
| Get Current Weather | OpenWeatherMap | Fetch current weather by coordinates | Split Locations | Merge Weather Data | ## PART 2: Data Collection — Fetches current weather, forecasts, and air pollution data simultaneously for each location. |
| Get 5-Day Forecast | OpenWeatherMap | Fetch 5-day forecast by coordinates | Split Locations | Merge Weather Data | ## PART 2: Data Collection — Fetches current weather, forecasts, and air pollution data simultaneously for each location. |
| Get Air Quality Index | HTTP Request | Fetch AQI (air_pollution endpoint) | Split Locations | Merge with Air Quality | ## PART 2: Data Collection — Fetches current weather, forecasts, and air pollution data simultaneously for each location. |
| Merge Weather Data | Merge | Combine current + forecast (by position) | Get Current Weather; Get 5-Day Forecast | Merge with Air Quality | ## PART 2: Data Collection — Fetches current weather, forecasts, and air pollution data simultaneously for each location. |
| Merge with Air Quality | Merge | Combine weather bundle + AQI (by position) | Merge Weather Data; Get Air Quality Index | Process and Analyze Data | ## PART 2: Data Collection — Fetches current weather, forecasts, and air pollution data simultaneously for each location. |
| Process and Analyze Data | Code | Compute comfort index + alert flags | Merge with Air Quality | Aggregate All Locations | ## PART 3: AI Analysis — Calculates comfort metrics and uses GPT-4 to generate human-like weather briefings. |
| Aggregate All Locations | Aggregate | Aggregate all cities into one payload | Process and Analyze Data | AI Weather Analysis | ## PART 3: AI Analysis — Calculates comfort metrics and uses GPT-4 to generate human-like weather briefings. |
| AI Weather Analysis | HTTP Request | Call OpenAI for comparative briefing | Aggregate All Locations | Format Weather Report | ## PART 3: AI Analysis — Calculates comfort metrics and uses GPT-4 to generate human-like weather briefings. |
| Format Weather Report | Code | Build Discord-ready report + hasAlerts | AI Weather Analysis | Check for Alerts | ## PART 4: Delivery & Logging — Smart routing: sends urgent alerts or routine reports to Discord and logs results to Google Sheets. |
| Check for Alerts | IF | Route alert vs routine message | Format Weather Report | Send Discord Alert; Send Discord Report | ## PART 4: Delivery & Logging — Smart routing: sends urgent alerts or routine reports to Discord and logs results to Google Sheets. |
| Send Discord Alert | Discord | Post urgent report to Discord | Check for Alerts (true) | Log to Google Sheets | ## PART 4: Delivery & Logging — Smart routing: sends urgent alerts or routine reports to Discord and logs results to Google Sheets. |
| Send Discord Report | Discord | Post routine report to Discord | Check for Alerts (false) | Log to Google Sheets | ## PART 4: Delivery & Logging — Smart routing: sends urgent alerts or routine reports to Discord and logs results to Google Sheets. |
| Log to Google Sheets | Google Sheets | Append execution output to spreadsheet | Send Discord Alert; Send Discord Report | — | ## PART 4: Delivery & Logging — Smart routing: sends urgent alerts or routine reports to Discord and logs results to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Weather Monitoring Across Multiple Cities with OpenWeatherMap, GPT-4o-mini, and Discord*.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set rule: **Every 6 hours**
   - This is your entry node.

3. **Add “Set Monitoring Locations”**
   - Node: **Set**
   - Mode: **Raw JSON output**
   - Paste an array of location objects, e.g.:
     - `city` (string), `lat` (string/number), `lon` (string/number), `timezone` (string)
   - Connect: `Schedule Trigger` → `Set Monitoring Locations`

4. **Add “Split Locations”**
   - Node: **Split Out**
   - Configure to split the **incoming array** into items (split root array / equivalent).
   - Connect: `Set Monitoring Locations` → `Split Locations`

5. **Add OpenWeatherMap credentials**
   - In n8n Credentials, create **OpenWeatherMap API** credential (API key).
   - This will be used by OpenWeatherMap nodes and referenced by the AQI HTTP node.

6. **Add “Get Current Weather”**
   - Node: **OpenWeatherMap**
   - Operation: Current weather (default)
   - Location selection: **Coordinates**
   - Latitude: `{{ $json.lat }}`
   - Longitude: `{{ $json.lon }}`
   - Language: `en`
   - Connect: `Split Locations` → `Get Current Weather`

7. **Add “Get 5-Day Forecast”**
   - Node: **OpenWeatherMap**
   - Operation: **5 Day Forecast**
   - Location selection: **Coordinates**
   - Latitude: `{{ $json.lat }}`
   - Longitude: `{{ $json.lon }}`
   - Language: `en`
   - Connect: `Split Locations` → `Get 5-Day Forecast`

8. **Add “Get Air Quality Index” (AQI)**
   - Node: **HTTP Request**
   - Authentication: **Predefined credential type**
   - Credential type: **OpenWeatherMap API**
   - Method: GET
   - URL (expression), preferably HTTPS:
     - `https://api.openweathermap.org/data/2.5/air_pollution?lat={{ $json.lat }}&lon={{ $json.lon }}&appid={{ $credentials.openWeatherMapApi.apiKey }}`
   - Connect: `Split Locations` → `Get Air Quality Index`

9. **Add “Merge Weather Data”**
   - Node: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - `Get Current Weather` → `Merge Weather Data` (Input 1 / index 0)
     - `Get 5-Day Forecast` → `Merge Weather Data` (Input 2 / index 1)

10. **Add “Merge with Air Quality”**
    - Node: **Merge**
    - Mode: **Combine**
    - Combine by: **Position**
    - Connect:
      - `Merge Weather Data` → `Merge with Air Quality` (input index 0)
      - `Get Air Quality Index` → `Merge with Air Quality` (input index 1)

11. **Add “Process and Analyze Data”**
    - Node: **Code**
    - Paste the JS logic that:
      - extracts temperature/humidity/wind/weather
      - extracts AQI and maps label
      - computes comfortIndex
      - builds alerts and `hasAlerts`
    - Connect: `Merge with Air Quality` → `Process and Analyze Data`

12. **Add “Aggregate All Locations”**
    - Node: **Aggregate**
    - Operation/mode: **Aggregate all item data** into a single field (commonly `data`)
    - Connect: `Process and Analyze Data` → `Aggregate All Locations`

13. **Add OpenAI credentials**
    - In n8n Credentials, create **OpenAI API** credential.
    - Ensure it is compatible with HTTP Request “predefinedCredentialType” usage.

14. **Add “AI Weather Analysis”**
    - Node: **HTTP Request**
    - Method: POST
    - URL: `https://api.openai.com/v1/chat/completions`
    - Authentication: **Predefined credential type** → **OpenAI API**
    - Body type: **JSON**
    - JSON body includes:
      - `model: gpt-4o-mini`
      - system message instructing concise meteorologist comparison (under 200 words, with emojis)
      - user message containing: `{{ JSON.stringify($json.data) }}`
    - Connect: `Aggregate All Locations` → `AI Weather Analysis`

15. **Add “Format Weather Report”**
    - Node: **Code**
    - Build a markdown report using:
      - aggregated data from `Aggregate All Locations` (`.first().json.data`)
      - AI analysis from OpenAI response `choices[0].message.content`
    - Output fields:
      - `report` (string)
      - `hasAlerts` (boolean)
    - Connect: `AI Weather Analysis` → `Format Weather Report`

16. **Add “Check for Alerts”**
    - Node: **IF**
    - Condition: `{{ $json.hasAlerts }}` **equals** `true` (boolean)
    - Connect: `Format Weather Report` → `Check for Alerts`

17. **Add Discord webhook credentials / setup**
    - In Discord, create webhook URL(s) for:
      - an **alerts** channel (urgent)
      - a **reports** channel (routine)
    - In n8n, configure Discord nodes to use webhook authentication.

18. **Add “Send Discord Alert”**
    - Node: **Discord**
    - Auth: webhook
    - Content: `🚨 **URGENT WEATHER ALERT**` + `{{ $json.report }}`
    - Connect: `Check for Alerts` (true) → `Send Discord Alert`

19. **Add “Send Discord Report”**
    - Node: **Discord**
    - Auth: webhook
    - Content: `{{ $json.report }}`
    - Connect: `Check for Alerts` (false) → `Send Discord Report`

20. **Add Google Sheets logging**
    - Create/choose a spreadsheet and sheet (e.g., `Sheet1`).
    - In n8n Credentials, set up **Google Sheets OAuth2** (or service account, depending on your n8n setup).
    - Node: **Google Sheets**
      - Operation: **Append**
      - Document: set **Spreadsheet ID**
      - Sheet: set **Sheet name**
      - Map incoming fields to columns (recommended columns: `timestamp`, `report`, `hasAlerts`).
    - Connect:
      - `Send Discord Alert` → `Log to Google Sheets`
      - `Send Discord Report` → `Log to Google Sheets`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates global weather monitoring… triggers every 6 hours… calculates a ‘Comfort Index’… routes prioritized alerts… archives all data in Google Sheets…” | Sticky note: **Main Overview** (workflow purpose and flow summary) |
| Setup requirements: connect OpenWeatherMap, OpenAI, Discord webhook; edit locations; provide Spreadsheet ID and Sheet name; verify Discord webhook URL | Sticky note: **Main Overview** (setup checklist) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.