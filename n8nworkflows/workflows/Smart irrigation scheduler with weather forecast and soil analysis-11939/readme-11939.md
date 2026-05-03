Smart irrigation scheduler with weather forecast and soil analysis

https://n8nworkflows.xyz/workflows/smart-irrigation-scheduler-with-weather-forecast-and-soil-analysis-11939


# Smart irrigation scheduler with weather forecast and soil analysis

## 1. Workflow Overview

**Purpose:**  
This workflow computes a daily (or manually triggered) irrigation plan per garden zone using **OpenWeatherMap current weather + forecast**, a **soil moisture simulation**, and **plant-type watering rules**. It then generates a **time-ordered schedule**, optionally **logs to Google Sheets**, **sends IoT commands** to an irrigation hub, and **notifies Slack** with a formatted report. If no watering is needed, it records a “no action” status and (if triggered via webhook) returns a response.

**Primary use cases:**
- Home/estate irrigation with multiple zones and smart valves
- Greenhouses/nurseries that want weather-aware watering
- Water conservation + automation with a human “manual override” endpoint

### 1.1 Triggering & Zone Input
- Runs daily at 06:00 and can be triggered via a POST webhook.
- Defines a static list of irrigation zones (lat/lon, plant type, soil type, last watered date, threshold).

### 1.2 Weather Collection (Per Zone)
- Splits zones into items.
- Fetches **current weather** and **5‑day forecast** for each zone via OpenWeatherMap.
- Merges current + forecast into a single combined item.

### 1.3 Irrigation Analysis (Per Zone)
- Calculates simplified evapotranspiration, estimates soil moisture from last watered, evaluates rain expectation in next 24h, and decides whether to water (with urgency, amount, and best time).

### 1.4 Scheduling, Execution, Logging, and Notification
- Filters zones that need watering and generates a sequential schedule.
- Branches: if schedule exists → Slack + IoT + Sheets + webhook response; else → set “no action” + webhook response.

---

## 2. Block-by-Block Analysis

### Block 1 — Triggering & Zone Configuration
**Overview:** Receives execution from either the daily schedule or a manual webhook call, then loads a static JSON list of irrigation zones and splits it into one item per zone.

**Nodes involved:**
- Daily Morning Check
- Manual Override Trigger
- Merge Triggers
- Define Irrigation Zones
- Split Zones

#### Node: **Daily Morning Check**
- **Type / Role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — time-based entry point.
- **Config choices:** Runs at **06:00** (hour-based trigger).
- **Outputs:** To **Merge Triggers** (branch index 0).
- **Version notes:** typeVersion **1.2**.
- **Failure modes:** Generally none besides n8n scheduler being disabled or timezone misunderstandings (n8n instance timezone vs desired local time).

#### Node: **Manual Override Trigger**
- **Type / Role:** Webhook (`n8n-nodes-base.webhook`) — manual entry point.
- **Config choices:**
  - **POST** `/<baseUrl>/webhook/irrigation-check` (path: `irrigation-check`)
  - `responseMode: responseNode` (response is sent by “Respond to Webhook”)
  - `onError: continueRegularOutput` (workflow continues even if webhook node errors)
- **Outputs:** To **Merge Triggers** (branch index 1).
- **Version notes:** typeVersion **2**.
- **Failure modes / edge cases:**
  - Missing/invalid webhook URL exposure (reverse proxy, n8n public URL, authentication).
  - Caller expects immediate response but downstream nodes are slow (weather APIs).
  - Because response is handled by a later node, failures before “Respond to Webhook” may cause timeouts for the HTTP caller.

#### Node: **Merge Triggers**
- **Type / Role:** Merge (`n8n-nodes-base.merge`) — unifies multiple entry points.
- **Config choices:** `mode: chooseBranch` (passes through whichever trigger fires).
- **Inputs:** Daily Morning Check, Manual Override Trigger.
- **Outputs:** Define Irrigation Zones.
- **Version notes:** typeVersion **3**.
- **Failure modes:** If both branches run simultaneously (rare), behavior depends on n8n execution; typically each execution is separate.

#### Node: **Define Irrigation Zones**
- **Type / Role:** Set (`n8n-nodes-base.set`) — provides zone configuration.
- **Config choices:** `mode: raw` with `jsonOutput` containing an **array** of zones, each including:
  - `zone`, `lat`, `lon`, `plantType`, `soilType`, `lastWatered`, `waterThreshold`
- **Output:** To Split Zones.
- **Version notes:** typeVersion **3.4**.
- **Edge cases:**
  - `lat/lon` are strings; downstream nodes accept expressions but APIs usually tolerate strings—still, numeric is safer.
  - `lastWatered` is static in this workflow; without updating it after watering, soil moisture estimation will drift and decisions may become wrong.
  - `waterThreshold` is defined but not used later (potential design gap).

#### Node: **Split Zones**
- **Type / Role:** Split Out (`n8n-nodes-base.splitOut`) — converts the zones array into multiple items (one per zone).
- **Config choices:** `fieldToSplitOut: "="` (unusual; typically this should point to the array field. Because the Set node outputs the array as the whole JSON, this configuration may be attempting to split the root.)
- **Outputs:** To both weather nodes: Get Current Weather and Get 5-Day Forecast.
- **Version notes:** typeVersion **1**.
- **Failure modes / edge cases:**
  - If `fieldToSplitOut` is misconfigured, the node may not split correctly. Recommended: ensure the zones array is under a known key (e.g., `zones`) and split that key.

**Sticky note(s) covering this block:**
- “Workflow Overview” (general)
- “Step 1: Triggers & Zone Config”

---

### Block 2 — Weather Data Collection (Per Zone)
**Overview:** For each zone item, calls OpenWeatherMap twice (current + forecast) and merges the results into a single combined item for analysis.

**Nodes involved:**
- Get Current Weather
- Get 5-Day Forecast
- Merge Weather Data

#### Node: **Get Current Weather**
- **Type / Role:** OpenWeatherMap (`n8n-nodes-base.openWeatherMap`) — API call for current conditions.
- **Config choices:**
  - `locationSelection: coordinates`
  - `latitude: {{ $json.lat }}`
  - `longitude: {{ $json.lon }}`
- **Input:** From Split Zones.
- **Output:** To Merge Weather Data (input 0).
- **Version notes:** typeVersion **1**.
- **Credentials required:** OpenWeatherMap API key in n8n credentials.
- **Failure modes:**
  - Invalid API key / quota exceeded.
  - Coordinates invalid.
  - Network/API timeouts.

#### Node: **Get 5-Day Forecast**
- **Type / Role:** OpenWeatherMap — forecast retrieval.
- **Config choices:**
  - `operation: 5DayForecast`
  - Coordinates via the same expressions as above.
- **Input:** From Split Zones.
- **Output:** To Merge Weather Data (input 1).
- **Version notes:** typeVersion **1**.
- **Failure modes:** Same as current weather; additionally forecast payload structure differs from current weather (important for analysis logic).

#### Node: **Merge Weather Data**
- **Type / Role:** Merge — combines current + forecast results.
- **Config choices:**
  - `mode: combine`
  - `combineBy: combineByPosition` (pairs item 1 from current with item 1 from forecast)
- **Inputs:** Get Current Weather (index 0), Get 5-Day Forecast (index 1)
- **Output:** Analyze Irrigation Need
- **Version notes:** typeVersion **3**.
- **Failure modes / edge cases:**
  - If either API returns a different item count or fails for a zone, pairing by position can mismatch zones.
  - Consider `Merge by Key` (e.g., zone name) if you store zone identifiers through the weather calls.

**Sticky note(s) covering this block:**
- “Step 2: Weather Data Collection”

---

### Block 3 — Irrigation Analysis & Decisioning (Per Zone)
**Overview:** Computes watering need using weather inputs and zone attributes. Produces a structured decision object (shouldWater, urgency, amount, bestTime, reason) plus analysis metrics.

**Nodes involved:**
- Analyze Irrigation Need
- Filter Zones Needing Water
- Aggregate All Results

#### Node: **Analyze Irrigation Need**
- **Type / Role:** Code (`n8n-nodes-base.code`) — decision engine.
- **Config choices:** Custom JavaScript producing one output item per analyzed input.
- **Key variables/expressions:**
  - `const items = $input.all();`
  - `const zoneData = $('Split Zones').item.json;` (pulls zone data from Split Zones)
  - Reads weather fields:
    - `current.main.temp`, `current.main.humidity`, `current.clouds.all`, `current.wind.speed`
    - forecast list: `current.list` (but see edge case below)
  - Forecast rain check over next 24h:
    - `next24h = forecastList.slice(0, 8)` (8 × 3h intervals)
    - `rainExpected` based on weather main or `rain['3h']`
    - `totalExpectedRain` sum of `rain['3h']`
  - Evapotranspiration approximation:
    - `solarRadiation = (100 - clouds) / 100 * 20`
    - `evapotranspiration = 0.0023 * (temp + 17.8) * solarRadiation * 0.5`
  - Soil moisture simulation:
    - `daysSinceWater` from `zoneData.lastWatered`
    - `soilMoistureEstimate = 100 - daysSinceWater*15 - evapotranspiration*5`
  - Plant needs map by `plantType` with `min`, `ideal`, `frequency`
  - Decision rules:
    1. If heavy rain expected (>5mm) → skip
    2. If moisture < min → high urgency
    3. If moisture < ideal and temp > 25 → medium
    4. If daysSinceWater >= frequency and not rainExpected → low
- **Inputs:** From Merge Weather Data.
- **Outputs:** To Filter Zones Needing Water and Aggregate All Results (in parallel).
- **Version notes:** typeVersion **2**.
- **Major edge cases / potential logic issues:**
  - **Current vs forecast merge structure mismatch:** After merging, each item is typically an object that contains both current and forecast data. The code treats `item.json` as `current` and expects both `main` and `list` to exist on the same object. In many OpenWeatherMap responses:
    - current weather has `main`, `wind`, `clouds`
    - forecast has `list`
    - merged item may not flatten them; it may nest them (or choose one side’s JSON depending on merge behavior). If `list` is missing, `forecastList` becomes empty and rain is never expected.
  - **Zone reference bug:** `zoneData` is taken from `$('Split Zones').item.json` once, not per item. In multi-item execution this can inadvertently always use the “current” item in that node context, leading to wrong zone info applied across items. Safer pattern: carry zone fields forward with each item (e.g., merge zone data into weather results) or reference `$json` zone fields directly.
  - `evapotranspiration` is a string after `toFixed(2)`; later used in arithmetic (`evapotranspiration * 5`) which coerces to number in JS, but this is implicit.
  - Soil type is not used in calculations (although included in output).
  - `waterThreshold` is unused.

#### Node: **Filter Zones Needing Water**
- **Type / Role:** Filter (`n8n-nodes-base.filter`) — keeps only zones where watering is required.
- **Config choices:** Boolean equals:
  - `{{ $json.decision.shouldWater }} == true`
- **Input:** From Analyze Irrigation Need.
- **Output:** To Generate Irrigation Schedule.
- **Version notes:** typeVersion **2.2**.
- **Failure modes:** If `decision.shouldWater` is missing/null due to code issues, strict validation may drop items unexpectedly.

#### Node: **Aggregate All Results**
- **Type / Role:** Aggregate (`n8n-nodes-base.aggregate`) — stores all zone results for reporting even when none need watering.
- **Config choices:** `aggregateAllItemData` (collects all incoming items into `json.data`).
- **Input:** From Analyze Irrigation Need.
- **Output:** To Generate Irrigation Schedule.
- **Version notes:** typeVersion **1**.
- **Edge cases:** If analysis node outputs zero items (due to code error), `first().json.data` in the next code node may fail.

**Sticky note(s) covering this block:**
- “Step 3: Irrigation Analysis”

---

### Block 4 — Scheduling, Execution, Logging, Notification, and Webhook Response
**Overview:** Builds a sequential watering schedule with gaps, generates a human-readable report, branches on whether there are tasks, then logs/sends commands/notifies, and responds to webhook calls.

**Nodes involved:**
- Generate Irrigation Schedule
- Has Irrigation Tasks?
- Send Slack Report
- Log to Google Sheets
- Send IoT Commands
- Log No Action
- Respond to Webhook

#### Node: **Generate Irrigation Schedule**
- **Type / Role:** Code — builds schedule + report text.
- **Config choices / logic:**
  - `zonesToWater = $input.all()` (this node receives items from **Filter Zones Needing Water** *and* also a single aggregated item from **Aggregate All Results**; see edge case)
  - `allZones = $('Aggregate All Results').first().json.data`
  - Start time fixed at 06:00; duration = `max(10, waterAmount*2)` minutes; 5-minute gap between zones.
  - Produces:
    - `schedule[]` entries with start/end/duration, urgency, amount, reason
    - `summary` totals
    - `allZoneStatus` for reporting when no irrigation needed
    - `report` Markdown-like message (contains icons)
    - `hasIrrigation` boolean
- **Inputs:** From Filter Zones Needing Water and Aggregate All Results.
- **Output:** To Has Irrigation Tasks?
- **Version notes:** typeVersion **2**.
- **Edge cases / failure modes:**
  - **Mixed inputs issue:** Because this node has *two inbound connections*, `$input.all()` may include the aggregated “all results” item as well (depending on how n8n merges inputs), which is not a zone object with `decision`. That can break scheduling unless n8n delivers only the Filter branch items on the main input and the aggregate is accessed via node reference (best practice is to avoid dual inputs for this pattern and instead use Merge/Wait or always rely on node reference for allZones).
  - The report string includes emoji/icons; if sending to systems that don’t render them, formatting may degrade (Slack usually supports).
  - Uses liters (`L`) as unit but waterAmount is an abstract value; IoT controller may expect different units.

#### Node: **Has Irrigation Tasks?**
- **Type / Role:** IF (`n8n-nodes-base.if`) — branching.
- **Condition:** `{{ $json.hasIrrigation }} == true`
- **True branch outputs:** Log to Google Sheets, Send IoT Commands, Send Slack Report
- **False branch output:** Log No Action
- **Version notes:** typeVersion **2**.
- **Edge cases:** If `hasIrrigation` missing due to upstream failure, strict boolean comparison routes to “false”.

#### Node: **Send Slack Report**
- **Type / Role:** Slack (`n8n-nodes-base.slack`) — notification.
- **Config choices:**
  - Sends `text: {{ $json.report }}`
  - Channel: `#garden` (selected by name)
- **Input:** True branch of IF.
- **Output:** None (terminal).
- **Version notes:** typeVersion **2.3**.
- **Credentials required:** Slack OAuth/token credential with permission to post to the channel.
- **Failure modes:** Auth/scopes, channel not found, formatting not accepted, rate limiting.

#### Node: **Log to Google Sheets**
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append log row(s).
- **Config choices:** `operation: append`, but **documentId and sheetName are empty** (must be configured).
- **Input:** True branch of IF.
- **Output:** None (terminal).
- **Version notes:** typeVersion **4.5**.
- **Credentials required:** Google OAuth2 / Service Account depending on setup.
- **Failure modes:** Missing document/sheet configuration, auth errors, invalid row mapping (not defined in shown config).

#### Node: **Send IoT Commands**
- **Type / Role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls irrigation controller/hub API.
- **Config choices:**
  - POST `https://your-iot-hub.com/api/irrigation`
  - JSON body: `{{ JSON.stringify($json.schedule) }}`
  - `specifyBody: json`, `sendBody: true`
- **Input:** True branch of IF.
- **Output:** To Respond to Webhook.
- **Version notes:** typeVersion **4.2**.
- **Edge cases / failure modes:**
  - If the node expects an object for JSON body, sending a pre-stringified body can cause double-encoding. Prefer passing the array/object directly (not stringified) when using JSON mode.
  - IoT API auth not configured (no headers/token in node).
  - Partial failures: one zone command fails; currently no retry/backoff logic.

#### Node: **Log No Action**
- **Type / Role:** Set — creates a simple status payload when nothing to do.
- **Config choices:** Sets:
  - `status = "No irrigation needed"`
  - `timestamp = {{ $now.toISO() }}`
- **Input:** False branch of IF.
- **Output:** To Respond to Webhook.
- **Version notes:** typeVersion **3.4**.

#### Node: **Respond to Webhook**
- **Type / Role:** Respond to Webhook (`n8n-nodes-base.respondToWebhook`) — returns HTTP response for manual calls.
- **Config choices:**
  - Respond with JSON:
    - `{ success: true, schedule: $json.schedule, summary: $json.summary }`
- **Inputs:** From Send IoT Commands **and** from Log No Action.
- **Version notes:** typeVersion **1.1**.
- **Edge cases:**
  - If execution was started by the Schedule Trigger (not webhook), this node may be unnecessary; in some setups it can error because there is no webhook request context (behavior depends on n8n version/config). Consider guarding it (separate path) or using “Respond to Webhook” only in webhook executions.

**Sticky note(s) covering this block:**
- “Step 4: Schedule & Execute”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Morning Check | Schedule Trigger | Daily entry point at 06:00 | — | Merge Triggers | ### Step 1: Triggers & Zone Config<br>Daily morning check at 6 AM plus manual webhook trigger. Define irrigation zones with coordinates, plant types, and soil conditions. |
| Manual Override Trigger | Webhook | Manual entry point (POST) | — | Merge Triggers | ### Step 1: Triggers & Zone Config<br>Daily morning check at 6 AM plus manual webhook trigger. Define irrigation zones with coordinates, plant types, and soil conditions. |
| Merge Triggers | Merge | Unify trigger branches | Daily Morning Check; Manual Override Trigger | Define Irrigation Zones | ### Step 1: Triggers & Zone Config<br>Daily morning check at 6 AM plus manual webhook trigger. Define irrigation zones with coordinates, plant types, and soil conditions. |
| Define Irrigation Zones | Set | Static zone configuration | Merge Triggers | Split Zones | ### Step 1: Triggers & Zone Config<br>Daily morning check at 6 AM plus manual webhook trigger. Define irrigation zones with coordinates, plant types, and soil conditions. |
| Split Zones | Split Out | One item per zone | Define Irrigation Zones | Get Current Weather; Get 5-Day Forecast | ### Step 1: Triggers & Zone Config<br>Daily morning check at 6 AM plus manual webhook trigger. Define irrigation zones with coordinates, plant types, and soil conditions. |
| Get Current Weather | OpenWeatherMap | Fetch current conditions per zone | Split Zones | Merge Weather Data | ### Step 2: Weather Data Collection<br>Fetch current conditions and 5-day forecast from OpenWeatherMap for each zone. Checks temperature, humidity, wind, and rain probability. |
| Get 5-Day Forecast | OpenWeatherMap | Fetch forecast per zone | Split Zones | Merge Weather Data | ### Step 2: Weather Data Collection<br>Fetch current conditions and 5-day forecast from OpenWeatherMap for each zone. Checks temperature, humidity, wind, and rain probability. |
| Merge Weather Data | Merge | Combine current + forecast | Get Current Weather; Get 5-Day Forecast | Analyze Irrigation Need | ### Step 2: Weather Data Collection<br>Fetch current conditions and 5-day forecast from OpenWeatherMap for each zone. Checks temperature, humidity, wind, and rain probability. |
| Analyze Irrigation Need | Code | Compute soil moisture estimate + decision | Merge Weather Data | Filter Zones Needing Water; Aggregate All Results | ### Step 3: Irrigation Analysis<br>Calculates evapotranspiration, estimates soil moisture, checks plant requirements, and determines if watering is needed with urgency level. |
| Filter Zones Needing Water | Filter | Keep only zones requiring watering | Analyze Irrigation Need | Generate Irrigation Schedule | ### Step 3: Irrigation Analysis<br>Calculates evapotranspiration, estimates soil moisture, checks plant requirements, and determines if watering is needed with urgency level. |
| Aggregate All Results | Aggregate | Preserve all zone outcomes for reporting | Analyze Irrigation Need | Generate Irrigation Schedule | ### Step 3: Irrigation Analysis<br>Calculates evapotranspiration, estimates soil moisture, checks plant requirements, and determines if watering is needed with urgency level. |
| Generate Irrigation Schedule | Code | Build timed schedule + report + summary | Filter Zones Needing Water; Aggregate All Results | Has Irrigation Tasks? | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |
| Has Irrigation Tasks? | IF | Branch on `hasIrrigation` | Generate Irrigation Schedule | (true) Log to Google Sheets; Send IoT Commands; Send Slack Report / (false) Log No Action | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |
| Send Slack Report | Slack | Send formatted report to Slack | Has Irrigation Tasks? (true) | — | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |
| Log to Google Sheets | Google Sheets | Append execution log | Has Irrigation Tasks? (true) | — | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |
| Send IoT Commands | HTTP Request | Send schedule to irrigation hub | Has Irrigation Tasks? (true) | Respond to Webhook | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |
| Log No Action | Set | Create “no action” payload | Has Irrigation Tasks? (false) | Respond to Webhook | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |
| Respond to Webhook | Respond to Webhook | Return JSON to webhook caller | Send IoT Commands; Log No Action | — |  |
| Workflow Overview | Sticky Note | Documentation block | — | — | ## Smart Irrigation Scheduler with Weather Forecast and Soil Analysis<br><br>**What this workflow does:**<br>Automatically determines optimal irrigation schedules based on weather forecasts, soil conditions, plant types, and evapotranspiration calculations. Prevents overwatering by checking rain forecasts and adjusts for hot/windy conditions.<br><br>**Key Features:**<br>- Multi-zone irrigation management<br>- Weather-based decision making (rain forecast, temperature, wind)<br>- Evapotranspiration calculation for accurate water needs<br>- Plant-type specific watering requirements<br>- Soil type consideration (clay, loam, sandy)<br>- Urgency-based scheduling (high/medium/low priority)<br>- IoT integration ready for smart valves<br>- Historical logging for optimization<br><br>**Who is this for:**<br>- Home gardeners with smart irrigation systems<br>- Commercial greenhouses and nurseries<br>- Agricultural operations<br>- Landscaping companies<br>- Property managers with large grounds<br>- Anyone wanting to conserve water while maintaining healthy plants<br><br>**How to set up:**<br>1. Configure your irrigation zones with coordinates<br>2. Add OpenWeatherMap API credentials<br>3. Set plant types and soil types per zone<br>4. Connect to your IoT irrigation controller<br>5. Configure Slack for notifications<br>6. (Optional) Connect Google Sheets for logging<br><br>**APIs Used:**<br>- OpenWeatherMap API (current + forecast)<br>- IoT Hub API (irrigation commands)<br>- Slack API (notifications)<br>- Google Sheets API (logging) |
| Step 1 | Sticky Note | Documentation block | — | — | ### Step 1: Triggers & Zone Config<br>Daily morning check at 6 AM plus manual webhook trigger. Define irrigation zones with coordinates, plant types, and soil conditions. |
| Step 2 | Sticky Note | Documentation block | — | — | ### Step 2: Weather Data Collection<br>Fetch current conditions and 5-day forecast from OpenWeatherMap for each zone. Checks temperature, humidity, wind, and rain probability. |
| Step 3 | Sticky Note | Documentation block | — | — | ### Step 3: Irrigation Analysis<br>Calculates evapotranspiration, estimates soil moisture, checks plant requirements, and determines if watering is needed with urgency level. |
| Step 4 | Sticky Note | Documentation block | — | — | ### Step 4: Schedule & Execute<br>Generates timed schedule, sends commands to IoT controllers, logs to Sheets, and notifies via Slack with detailed report. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Smart Irrigation Scheduler with Weather Forecast and Soil Analysis**
- (Optional) Set workflow setting **Execution Order** to `v1` (matches the workflow settings).

2) **Add Trigger #1: Schedule**
- Add node: **Schedule Trigger** → name **Daily Morning Check**
- Configure: trigger at **06:00** (daily).

3) **Add Trigger #2: Webhook**
- Add node: **Webhook** → name **Manual Override Trigger**
- Method: **POST**
- Path: `irrigation-check`
- Response mode: **Using “Respond to Webhook” node** (response node mode)

4) **Merge the triggers**
- Add node: **Merge** → name **Merge Triggers**
- Mode: **Choose Branch**
- Connect:
  - Daily Morning Check → Merge Triggers (Input 1 / index 0)
  - Manual Override Trigger → Merge Triggers (Input 2 / index 1)

5) **Define zones**
- Add node: **Set** → name **Define Irrigation Zones**
- Mode: **Raw**
- Paste the zones array (or create your own) containing at least:
  - `zone`, `lat`, `lon`, `plantType`, `soilType`, `lastWatered`, `waterThreshold`
- Connect: Merge Triggers → Define Irrigation Zones

6) **Split zones into items**
- Add node: **Split Out** → name **Split Zones**
- Configure to split the zones array correctly:
  - Recommended approach: in the Set node, wrap the array under a key like `zones`, then set Split Out “Field to split out” = `zones`.
- Connect: Define Irrigation Zones → Split Zones

7) **Fetch current weather**
- Add node: **OpenWeatherMap** → name **Get Current Weather**
- Location selection: **Coordinates**
- Latitude: `{{ $json.lat }}`
- Longitude: `{{ $json.lon }}`
- Configure **OpenWeatherMap credentials** (API key).
- Connect: Split Zones → Get Current Weather

8) **Fetch forecast**
- Add node: **OpenWeatherMap** → name **Get 5-Day Forecast**
- Operation: **5 Day Forecast**
- Location selection: **Coordinates**
- Latitude/Longitude same expressions as above
- Connect: Split Zones → Get 5-Day Forecast

9) **Merge current + forecast**
- Add node: **Merge** → name **Merge Weather Data**
- Mode: **Combine**
- Combine by: **Position**
- Connect:
  - Get Current Weather → Merge Weather Data (Input 1)
  - Get 5-Day Forecast → Merge Weather Data (Input 2)

10) **Analyze irrigation need**
- Add node: **Code** → name **Analyze Irrigation Need**
- Paste the decision JavaScript (adapt if you change merge structure).
- Connect: Merge Weather Data → Analyze Irrigation Need

11) **Filter zones that need watering**
- Add node: **Filter** → name **Filter Zones Needing Water**
- Condition: Boolean equals
  - Left: `{{ $json.decision.shouldWater }}`
  - Right: `true`
- Connect: Analyze Irrigation Need → Filter Zones Needing Water

12) **Aggregate all analysis results**
- Add node: **Aggregate** → name **Aggregate All Results**
- Mode: **Aggregate all item data**
- Connect: Analyze Irrigation Need → Aggregate All Results

13) **Generate schedule + report**
- Add node: **Code** → name **Generate Irrigation Schedule**
- Paste the schedule/report JavaScript.
- Connect:
  - Filter Zones Needing Water → Generate Irrigation Schedule
  - Aggregate All Results → Generate Irrigation Schedule  
  (If you encounter mixed-input issues, prefer: connect only Filter → Generate, and still reference Aggregate via `$('Aggregate All Results')...`.)

14) **Branch based on whether schedule exists**
- Add node: **IF** → name **Has Irrigation Tasks?**
- Condition: `{{ $json.hasIrrigation }} == true`
- Connect: Generate Irrigation Schedule → Has Irrigation Tasks?

15) **Slack notification (true branch)**
- Add node: **Slack** → name **Send Slack Report**
- Channel: `#garden` (or your channel)
- Text: `{{ $json.report }}`
- Add Slack credentials (OAuth/token with chat:write).
- Connect: Has Irrigation Tasks? (true) → Send Slack Report

16) **Google Sheets logging (true branch, optional but configured in workflow)**
- Add node: **Google Sheets** → name **Log to Google Sheets**
- Operation: **Append**
- Configure:
  - Document ID (spreadsheet)
  - Sheet name (tab)
  - Map columns/fields (e.g., date, totalWater, totalDuration, schedule JSON)
- Add Google credentials.
- Connect: Has Irrigation Tasks? (true) → Log to Google Sheets

17) **Send IoT commands (true branch)**
- Add node: **HTTP Request** → name **Send IoT Commands**
- Method: **POST**
- URL: `https://your-iot-hub.com/api/irrigation`
- Body type: JSON
- Body: ideally set it to the schedule object/array (avoid double-stringifying). If you keep the original approach, set body to `{{ JSON.stringify($json.schedule) }}` but ensure the receiver expects a string.
- Add auth headers/token as required by your IoT hub.
- Connect: Has Irrigation Tasks? (true) → Send IoT Commands

18) **No-action path (false branch)**
- Add node: **Set** → name **Log No Action**
- Set fields:
  - `status` = `No irrigation needed`
  - `timestamp` = `{{ $now.toISO() }}`
- Connect: Has Irrigation Tasks? (false) → Log No Action

19) **Respond to webhook**
- Add node: **Respond to Webhook** → name **Respond to Webhook**
- Respond with: JSON
- Body: `{{ {success: true, schedule: $json.schedule, summary: $json.summary} }}`
- Connect:
  - Send IoT Commands → Respond to Webhook
  - Log No Action → Respond to Webhook
- Note: If you want clean behavior for schedule-trigger runs, consider separating webhook execution into its own branch (or make response conditional).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Smart Irrigation Scheduler with Weather Forecast and Soil Analysis — description, features, setup steps, and APIs used | Sticky note “Workflow Overview” |
| Step 1: Triggers & Zone Config | Sticky note “Step 1” |
| Step 2: Weather Data Collection | Sticky note “Step 2” |
| Step 3: Irrigation Analysis | Sticky note “Step 3” |
| Step 4: Schedule & Execute | Sticky note “Step 4” |