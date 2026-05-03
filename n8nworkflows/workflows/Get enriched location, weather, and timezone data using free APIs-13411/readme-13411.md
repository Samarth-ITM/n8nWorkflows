Get enriched location, weather, and timezone data using free APIs

https://n8nworkflows.xyz/workflows/get-enriched-location--weather--and-timezone-data-using-free-apis-13411


# Get enriched location, weather, and timezone data using free APIs

## 1. Workflow Overview

**Purpose:**  
This workflow accepts GPS coordinates (`lat`, `lon`) via a Webhook and returns **enriched location intelligence** as a single JSON response by aggregating data from **four mostly-free public APIs**: reverse geocoding (address), timezone, sunrise/sunset times, and current weather.

**Primary use cases:**
- Enriching raw coordinates in apps (mobile check-ins, logistics, travel, IoT)
- Building “location cards” (address + weather + sun times + local time)
- Normalizing multi-API location metadata into one stable output format

### 1.1 Input Reception
- Webhook receives query parameters and triggers the workflow.
- Fans out into parallel API calls.

### 1.2 Parallel Data Enrichment (4 sources)
- Reverse geocoding via Nominatim (OpenStreetMap)
- Timezone data via TimezoneDB
- Weather via OpenWeatherMap
- Sunrise/sunset via Sunrise-Sunset.org (depends on TimezoneDB output for `tzid`)

### 1.3 Aggregation & Normalization
- Merge combines the 4 API outputs into one item while handling field name collisions.
- A Set node reshapes and formats the final output (28 fields).

### 1.4 Response
- Respond to Webhook returns all fields as JSON (array of one object).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception & Fan-out
**Overview:** Receives `lat` and `lon` and starts three API requests in parallel (Nominatim, TimezoneDB, OpenWeatherMap).  
**Nodes involved:**  
- `Webhook - Accepts Coordinates`

#### Node: Webhook - Accepts Coordinates
- **Type / role:** `Webhook` trigger; entry point.
- **Key configuration:**
  - **Path:** `geo-details`
  - **Response mode:** `Using Respond to Webhook node` (responseNode)
  - Expects query parameters `lat` and `lon`.
- **Key expressions/variables used:** Outputs are referenced later as:
  - `$('Webhook - Accepts Coordinates').item.json.query.lat`
  - `$('Webhook - Accepts Coordinates').item.json.query.lon`
- **Connections:**
  - Outputs to:
    - `HTTP Request (Nominatim)`
    - `HTTP Request (TimezoneDB)`
    - `OpenWeatherMap`
- **Edge cases / failures:**
  - Missing/invalid `lat`/`lon` (strings, out of range) can cause downstream API errors or empty fields.
  - If the Webhook is hit with multiple items (rare for Webhook), later “combine by position” assumptions could break.
- **Version-specific notes:** Node `typeVersion 2.1`—behavior consistent with modern n8n webhook response-node pattern.

---

### Block 2.2 — Reverse Geocoding (Address)
**Overview:** Converts coordinates to a human-readable address and address components using OpenStreetMap’s Nominatim reverse geocoding endpoint.  
**Nodes involved:**  
- `HTTP Request (Nominatim)`

#### Node: HTTP Request (Nominatim)
- **Type / role:** `HTTP Request` to external REST API.
- **Configuration choices:**
  - **Method:** GET (implicit)
  - **URL:** `https://nominatim.openstreetmap.org/reverse`
  - **Query params:**
    - `lat = {{ $json.query.lat }}`
    - `lon = {{ $json.query.lon }}`
    - `format = json`
    - `accept-language = en`
  - No authentication (public endpoint).
- **Inputs:** From `Webhook - Accepts Coordinates` (the webhook item contains `.query.lat/.query.lon`).
- **Outputs:** JSON including `display_name` and nested `address` fields (e.g., suburb/state/country/postcode).
- **Connections:** Output goes to `Merge` input index 0.
- **Edge cases / failures:**
  - Strict usage policy and rate limiting (noted 1 req/sec): may return 429 / block if abused.
  - Some locations may not return certain address components (null/undefined); later Set node maps them directly.
  - If Nominatim returns an error payload, the merge and mapping can produce missing `display_name_1` / `address_1`.
- **Version-specific notes:** `typeVersion 4.2` (HTTP Request node). Ensure “Send Query Parameters” is enabled as configured.

---

### Block 2.3 — Timezone Lookup (and Dependency Provider)
**Overview:** Fetches timezone metadata for the coordinates; also provides the timezone ID used by Sunrise-Sunset for local-time calculation.  
**Nodes involved:**  
- `HTTP Request (TimezoneDB)`
- (dependency for) `HTTP Request (Sunrise-Sunset)`

#### Node: HTTP Request (TimezoneDB)
- **Type / role:** `HTTP Request` to TimezoneDB API.
- **Configuration choices:**
  - **URL:** `http://api.timezonedb.com/v2.1/get-time-zone`
  - **Query params:**
    - `by = position`
    - `lat = {{ $json.query.lat }}`
    - `lng = {{ $json.query.lon }}`
    - `format = json`
  - **Authentication:** `httpQueryAuth` via generic credential type (API key in query).
- **Credentials:**
  - `TimezoneDB API` (stored as `httpQueryAuth`)
- **Inputs:** From `Webhook - Accepts Coordinates`.
- **Outputs:** Fields such as `zoneName`, `abbreviation`, `formatted`, `timestamp`, `countryCode`, etc.
- **Connections:**
  - To `Merge` input index 1
  - To `HTTP Request (Sunrise-Sunset)` (provides `zoneName` used as `tzid`)
- **Edge cases / failures:**
  - Invalid/expired API key → 401/403 or error JSON (workflow likely fails unless “Continue On Fail” is enabled—it's not shown here).
  - Rate limit (noted 1 req/sec) can cause throttling.
  - Uses HTTP (not HTTPS) in the URL; some environments enforce HTTPS-only egress or may block mixed content.
- **Version-specific notes:** `typeVersion 4.2`.

---

### Block 2.4 — Astronomical Times (Sunrise/Sunset)
**Overview:** Calculates sunrise, sunset, and day length using coordinates plus timezone ID (`tzid`) to return times in the correct local timezone.  
**Nodes involved:**  
- `HTTP Request (Sunrise-Sunset)`

#### Node: HTTP Request (Sunrise-Sunset)
- **Type / role:** `HTTP Request` to Sunrise-Sunset.org.
- **Configuration choices:**
  - **URL:** `https://api.sunrise-sunset.org/json`
  - **Query params:**
    - `lat = {{ $('Webhook - Accepts Coordinates').item.json.query.lat }}`
    - `lng = {{ $('Webhook - Accepts Coordinates').item.json.query.lon }}`
    - `tzid = {{ $json.zoneName }}`
  - The `tzid` value is taken from **TimezoneDB** output (`$json.zoneName`) because this node is connected from TimezoneDB.
- **Inputs:** From `HTTP Request (TimezoneDB)` (primary), while also referencing Webhook values via node selector.
- **Outputs:** Response object containing `results.sunrise`, `results.sunset`, `results.day_length`, etc.
- **Connections:** Output goes to `Merge` input index 2.
- **Edge cases / failures:**
  - If TimezoneDB fails or returns no `zoneName`, the `tzid` is invalid → Sunrise-Sunset may fall back or return error/unexpected format.
  - Sunrise-Sunset returns times as strings; formatting expectations in downstream Set node assume typical keys exist under `results`.
- **Version-specific notes:** `typeVersion 4.2`.

---

### Block 2.5 — Weather Enrichment
**Overview:** Fetches current weather conditions for the coordinates using OpenWeatherMap.  
**Nodes involved:**  
- `OpenWeatherMap`

#### Node: OpenWeatherMap
- **Type / role:** `OpenWeatherMap` node (n8n integration).
- **Configuration choices:**
  - **Location selection:** `coordinates`
  - **Latitude:** `={{ $json.query.lat }}`
  - **Longitude:** `={{ $json.query.lon }}`
- **Credentials:**
  - `OpenWeatherMap account` (OpenWeatherMap API key)
- **Inputs:** From `Webhook - Accepts Coordinates`.
- **Outputs:** Typical OpenWeatherMap current weather object, including:
  - `main.temp`, `main.feels_like`, `main.pressure`, `main.humidity`
  - `weather[0].main`, `weather[0].description`, `weather[0].icon`
- **Connections:** Output goes to `Merge` input index 3.
- **Edge cases / failures:**
  - Wrong API key → 401
  - API quota/rate limit → 429
  - Some coordinates (e.g., over oceans) still return results but may vary; downstream assumes `weather[0]` exists.
- **Version-specific notes:** `typeVersion 1`.

---

### Block 2.6 — Merge the 4 Results
**Overview:** Combines the four parallel branches into one unified item, resolving naming conflicts by adding suffixes to clashing fields.  
**Nodes involved:**  
- `Merge`

#### Node: Merge
- **Type / role:** `Merge` node; aggregation join.
- **Configuration choices:**
  - **Mode:** `Combine`
  - **Combine by:** `Position` (combineByPosition)
  - **Number of inputs:** `4`
  - **Clash handling:** `Add suffix` to duplicate fields
- **Inputs:**
  1. Nominatim output (address)
  2. TimezoneDB output (timezone)
  3. Sunrise-Sunset output (sun times)
  4. OpenWeatherMap output (weather)
- **Outputs:** One combined object. Due to clash handling, fields are suffixed (e.g., `_1`, `_2`, `_3`, `_4`), which is later relied upon by the Set node.
- **Connections:** Output goes to `Edit Fields - Format & Structure Output`.
- **Edge cases / failures:**
  - **Item alignment risk:** “Combine by position” assumes each branch outputs the same number of items in the same order. If any API call returns zero items, multiple items, or errors, the merge can misalign or produce empty outputs.
  - Name collisions produce suffixes; any change in upstream payload shapes can shift which fields get suffixed and break downstream mappings.
- **Version-specific notes:** `typeVersion 3.2`.

---

### Block 2.7 — Normalize, Format, and Structure Output
**Overview:** Converts the merged, suffix-heavy object into a clean 28-field schema and formats date/time strings.  
**Nodes involved:**  
- `Edit Fields - Format & Structure Output`

#### Node: Edit Fields - Format & Structure Output
- **Type / role:** `Set` node; field mapping and transformation.
- **Configuration choices:**
  - **Dot notation:** enabled (allows nested assignments; here mainly used for reading nested fields).
  - Creates **28 fields** (lat/lon, timezone/time, address components, sun times, weather metrics, icon URLs).
- **Key expressions/variables used (representative):**
  - Coordinates from Webhook:
    - `lat = {{ $('Webhook - Accepts Coordinates').item.json.query.lat }}`
    - `lon = {{ $('Webhook - Accepts Coordinates').item.json.query.lon }}`
  - TimezoneDB fields (because of merge suffixing):
    - `timezone = {{ $json.zoneName_2 }}`
    - `timezone_short = {{ $json.abbreviation_2 }}`
    - `timestamp = {{ $json.timestamp_2 }}`
    - `current_time = {{ $json.formatted_2.toDateTime().format('hh:mm a') }}`
    - `current_date = {{ $json.formatted_2.toDateTime().format('dd-MMM-yyyy') }}`
  - Nominatim fields:
    - `address_full = {{ $json.display_name_1 }}`
    - `suburb = {{ $json.address_1.suburb }}`, etc.
  - Sunrise-Sunset fields:
    - `sunrise = {{ $json.results_3.sunrise }}`
    - `sunset = {{ $json.results_3.sunset }}`
    - `day_length = {{ $json.results_3.day_length }}`
  - OpenWeatherMap fields:
    - `temperature = {{ $json.main_4.temp }}`
    - `weather_main = {{ $json.weather[0].main }}`
  - Computed URLs:
    - `weather_icon = https://openweathermap.org/img/wn/{{ $json.weather[0].icon }}@2x.png`
    - `country_flag = https://flagsapi.com/{{ $json.countryCode_2 }}/flat/64.png`
- **Inputs:** From `Merge`.
- **Outputs:** A normalized object with consistent keys intended for clients.
- **Connections:** Output goes to `Respond to Webhook - Return JSON Response`.
- **Edge cases / failures:**
  - If `formatted_2` is missing or not parseable, `.toDateTime()` can fail.
  - If `weather` array is empty/missing, `$json.weather[0]...` fails.
  - If any upstream call changes or the merge suffixes change, references like `zoneName_2` or `display_name_1` can break.
- **Version-specific notes:** `typeVersion 3.4` (Set node).

---

### Block 2.8 — Return Response to Caller
**Overview:** Returns the final structured item(s) as the webhook HTTP response body.  
**Nodes involved:**  
- `Respond to Webhook - Return JSON Response`

#### Node: Respond to Webhook - Return JSON Response
- **Type / role:** `Respond to Webhook`; closes the request with JSON.
- **Configuration choices:**
  - **Respond with:** `All Incoming Items` (so the response is a JSON array)
  - **Response code:** `200`
- **Inputs:** From `Edit Fields - Format & Structure Output`.
- **Outputs:** HTTP response.
- **Edge cases / failures:**
  - If any upstream node errors and the workflow stops, the webhook call may time out or return an error depending on n8n settings.
- **Version-specific notes:** `typeVersion 1.4`.

---

### Block 2.9 — Documentation Notes (Sticky Notes)
**Overview:** Two sticky notes provide setup instructions and a sample request/response for users.  
**Nodes involved:**  
- `Sticky Note`
- `Sticky Note1`

#### Node: Sticky Note
- **Type / role:** `Sticky Note` (documentation only; no execution impact).
- **Content summary:** Explains purpose, how it works, and setup steps (webhook path, credentials, deploy).

#### Node: Sticky Note1
- **Type / role:** `Sticky Note` (documentation only).
- **Content summary:** Contains a sample `curl` request and a sample JSON response showing the 28 output fields.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Accepts Coordinates | Webhook | Entry point; receives `lat`/`lon` and triggers workflow | — | HTTP Request (Nominatim); HTTP Request (TimezoneDB); OpenWeatherMap | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| HTTP Request (Nominatim) | HTTP Request | Reverse geocode coordinates to address components | Webhook - Accepts Coordinates | Merge | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| HTTP Request (TimezoneDB) | HTTP Request | Fetch timezone data (zoneName, abbreviation, formatted time) | Webhook - Accepts Coordinates | Merge; HTTP Request (Sunrise-Sunset) | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| HTTP Request (Sunrise-Sunset) | HTTP Request | Get sunrise/sunset/day length using tzid | HTTP Request (TimezoneDB) | Merge | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| OpenWeatherMap | OpenWeatherMap | Fetch current weather for coordinates | Webhook - Accepts Coordinates | Merge | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| Merge | Merge | Combine 4 API outputs into one item; add suffix on clashes | HTTP Request (Nominatim); HTTP Request (TimezoneDB); HTTP Request (Sunrise-Sunset); OpenWeatherMap | Edit Fields - Format & Structure Output | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| Edit Fields - Format & Structure Output | Set | Map/format merged data into final 28-field schema | Merge | Respond to Webhook - Return JSON Response | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| Respond to Webhook - Return JSON Response | Respond to Webhook | Return JSON array response to caller | Edit Fields - Format & Structure Output | — | ## Get location insights using free APIs / Accepts `lat` and `lon` via webhook; fetches from 4 sources; returns 28 fields; setup: webhook path `geo-details`, add OpenWeatherMap + TimezoneDB keys, activate |
| Sticky Note | Sticky Note | Documentation (overview + setup) | — | — | ## Get location insights using free APIs … (full note content in section 5) |
| Sticky Note1 | Sticky Note | Documentation (sample request/response) | — | — | ## Sample Request & Response … (full note content in section 5) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Get location insights using free APIs** (or the provided title).

2. **Add Webhook trigger**
   - Add node: **Webhook**
   - Name: `Webhook - Accepts Coordinates`
   - **Path:** `geo-details`
   - **Response Mode:** *Using “Respond to Webhook” node* (responseNode)

3. **Add Nominatim reverse geocode request**
   - Add node: **HTTP Request**
   - Name: `HTTP Request (Nominatim)`
   - **Method:** GET
   - **URL:** `https://nominatim.openstreetmap.org/reverse`
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `lat` → `={{ $json.query.lat }}`
     - `lon` → `={{ $json.query.lon }}`
     - `format` → `json`
     - `accept-language` → `en`
   - Connect: `Webhook - Accepts Coordinates` → `HTTP Request (Nominatim)`

4. **Add TimezoneDB request (with API key)**
   - Add node: **HTTP Request**
   - Name: `HTTP Request (TimezoneDB)`
   - **Method:** GET
   - **URL:** `http://api.timezonedb.com/v2.1/get-time-zone`
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `by` → `position`
     - `lat` → `={{ $json.query.lat }}`
     - `lng` → `={{ $json.query.lon }}`
     - `format` → `json`
   - **Authentication:** Generic credential type → `HTTP Query Auth`
   - Create credentials (n8n Credentials):
     - Name: `TimezoneDB API`
     - Add your TimezoneDB key in the query-auth field (as required by your credential setup)
   - Connect: `Webhook - Accepts Coordinates` → `HTTP Request (TimezoneDB)`

5. **Add Sunrise-Sunset request**
   - Add node: **HTTP Request**
   - Name: `HTTP Request (Sunrise-Sunset)`
   - **Method:** GET
   - **URL:** `https://api.sunrise-sunset.org/json`
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `lat` → `={{ $('Webhook - Accepts Coordinates').item.json.query.lat }}`
     - `lng` → `={{ $('Webhook - Accepts Coordinates').item.json.query.lon }}`
     - `tzid` → `={{ $json.zoneName }}`
   - Connect: `HTTP Request (TimezoneDB)` → `HTTP Request (Sunrise-Sunset)`

6. **Add OpenWeatherMap node (with API key)**
   - Add node: **OpenWeatherMap**
   - Name: `OpenWeatherMap`
   - **Location Selection:** Coordinates
   - **Latitude:** `={{ $json.query.lat }}`
   - **Longitude:** `={{ $json.query.lon }}`
   - Create credentials:
     - Name: `OpenWeatherMap account`
     - Paste your OpenWeatherMap API key
   - Connect: `Webhook - Accepts Coordinates` → `OpenWeatherMap`

7. **Add Merge node to combine 4 inputs**
   - Add node: **Merge**
   - Name: `Merge`
   - **Mode:** Combine
   - **Combine By:** Position
   - **Number of Inputs:** 4
   - **Options → Clash Handling:** Add suffix to duplicate fields
   - Connect inputs to Merge in this order:
     1. `HTTP Request (Nominatim)` → `Merge` (input 1 / index 0)
     2. `HTTP Request (TimezoneDB)` → `Merge` (input 2 / index 1)
     3. `HTTP Request (Sunrise-Sunset)` → `Merge` (input 3 / index 2)
     4. `OpenWeatherMap` → `Merge` (input 4 / index 3)

8. **Add Set node to format and structure the final output**
   - Add node: **Set**
   - Name: `Edit Fields - Format & Structure Output`
   - Enable **Dot Notation**
   - Add the following fields (names + expressions):
     - `lat` (number): `={{ $('Webhook - Accepts Coordinates').item.json.query.lat }}`
     - `lon` (number): `={{ $('Webhook - Accepts Coordinates').item.json.query.lon }}`
     - `timezone`: `={{ $json.zoneName_2 }}`
     - `timezone_short`: `={{ $json.abbreviation_2 }}`
     - `current_time`: `={{ $json.formatted_2.toDateTime().format('hh:mm a') }}`
     - `current_date`: `={{ $json.formatted_2.toDateTime().format('dd-MMM-yyyy') }}`
     - `timestamp`: `={{ $json.timestamp_2 }}`
     - `address_full`: `={{ $json.display_name_1 }}`
     - `neighbourhood`: `={{ $json.address_1.neighbourhood }}`
     - `suburb`: `={{ $json.address_1.suburb }}`
     - `town`: `={{ $json.address_1.town }}`
     - `county`: `={{ $json.address_1.county }}`
     - `state_district`: `={{ $json.address_1.state_district }}`
     - `state`: `={{ $json.address_1.state }}`
     - `country`: `={{ $json.address_1.country }}`
     - `country_code`: `={{ $json.address_1.country_code }}`
     - `postcode`: `={{ $json.address_1.postcode }}`
     - `sunrise`: `={{ $json.results_3.sunrise }}`
     - `sunset`: `={{ $json.results_3.sunset }}`
     - `day_length`: `={{ $json.results_3.day_length }}`
     - `temperature`: `={{ $json.main_4.temp }}`
     - `feels_like`: `={{ $json.main_4.feels_like }}`
     - `temp_min`: `={{ $json.main_4.temp_min }}`
     - `temp_max`: `={{ $json.main_4.temp_max }}`
     - `pressure`: `={{ $json.main_4.pressure }}`
     - `humidity`: `={{ $json.main_4.humidity }}`
     - `weather_main`: `={{ $json.weather[0].main }}`
     - `weather_description`: `={{ $json.weather[0].description }}`
     - `weather_icon`: `=https://openweathermap.org/img/wn/{{ $json.weather[0].icon }}@2x.png`
     - `country_flag`: `=https://flagsapi.com/{{ $json.countryCode_2 }}/flat/64.png`
   - Connect: `Merge` → `Edit Fields - Format & Structure Output`

9. **Add Respond to Webhook**
   - Add node: **Respond to Webhook**
   - Name: `Respond to Webhook - Return JSON Response`
   - **Respond With:** All Incoming Items
   - **Response Code:** 200
   - Connect: `Edit Fields - Format & Structure Output` → `Respond to Webhook - Return JSON Response`

10. **(Optional) Add documentation sticky notes**
   - Add two sticky notes and paste the contents from section 5 for in-canvas documentation.

11. **Activate workflow**
   - Save and activate.
   - Test with:
     - `curl "https://<your-n8n-host>/webhook/geo-details?lat=27.1751495&lon=78.0395673"`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Get location insights using free APIs  \nThis workflow transforms GPS coordinates into rich location intelligence using free public APIs.\nIt aggregates data from OpenStreetMap, TimezoneDB, OpenWeatherMap, and Sunrise-Sunset.org to provide address details, timezone, weather, and astronomical times in a single call.\n\n## How it works\n1. **Input**: Accepts `lat` and `lon` via webhook.\n2. **Process**: Fetches data from 4 parallel sources.\n3. **Output**: Returns a unified JSON object with 28 enriched fields.\n\n## Setup steps\n1. **Configure Webhook**: Set the path to `geo-details`.\n2. **Add Credentials**: Add your OpenWeatherMap and TimezoneDB free API keys.\n3. **Deploy**: Activate the workflow and use the production URL. | Workflow canvas note (overview + setup) |
| ## Sample Request & Response\nLocation: Taj Mahal, Agra, India\n\nREQUEST:\n``` \ncurl \"https://n8n.example.com/webhook/geo-details?lat=27.1751495&lon=78.0395673\"\n```\n\nRESPONSE: JSON\n```\n[\n  {\n    \"lat\": 27.1751495,\n    \"lon\": 78.0395673,\n    \"timezone\": \"Asia/Kolkata\",\n    \"timezone_short\": \"IST\",\n    \"current_time\": \"10:25 PM\",\n    \"current_date\": \"15-Feb-2026\",\n    \"timestamp\": \"1771194349\",\n    \"address_full\": \"Cicuit House Road, Taj Ganj, Agra, Uttar Pradesh, 282004, India\",\n    \"neighbourhood\": null,\n    \"suburb\": \"Taj Ganj\",\n    \"town\": null,\n    \"county\": null,\n    \"state_district\": \"Agra\",\n    \"state\": \"Uttar Pradesh\",\n    \"country\": \"India\",\n    \"country_code\": \"in\",\n    \"postcode\": \"282004\",\n    \"sunrise\": \"6:53:17 AM\",\n    \"sunset\": \"6:10:35 PM\",\n    \"day_length\": \"11:17:18\",\n    \"temperature\": \"18.96\",\n    \"feels_like\": \"17.85\",\n    \"temp_min\": \"18.96\",\n    \"temp_max\": \"18.96\",\n    \"pressure\": \"1014\",\n    \"humidity\": \"36\",\n    \"weather_main\": \"Clear\",\n    \"weather_description\": \"clear sky\",\n    \"weather_icon\": \"https://openweathermap.org/img/wn/01n@2x.png\",\n    \"country_flag\": \"https://flagsapi.com/IN/flat/64.png\"\n  }\n]\n``` \nNote: Returns 28 fields with complete location data from 100% FREE APIs | Example usage and expected output |

---