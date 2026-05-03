Wake up on time using Google Maps traffic, Twilio SMS, and iOS Shortcuts

https://n8nworkflows.xyz/workflows/wake-up-on-time-using-google-maps-traffic--twilio-sms--and-ios-shortcuts-13386


# Wake up on time using Google Maps traffic, Twilio SMS, and iOS Shortcuts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow checks weekday mornings whether you must “wake up now” to arrive at a destination by a target time (hard-coded as **9:00 AM**) given **current/pessimistic traffic** from Google Maps. If it’s time, it sends an SMS via Twilio. A Google Sheet row is used as a lightweight daily state store to avoid repeated alerts.

**Typical use cases:**
- Dynamic “traffic-aware” morning alarm for commuting.
- Automations that must trigger once per day based on travel duration and a fixed arrival deadline.
- Driving-time-based notifications to a phone that can be chained into iOS Shortcuts automations.

### Logical Blocks
**1.1 Scheduling & Today Context**  
Trigger every minute during a morning window; compute today’s date.

**1.2 Daily State Check/Reset (Google Sheets)**  
Read the stored date/status from a sheet; if the stored date is not today, reset it.

**1.3 Address Setup & Geocoding (Google Geocoding API)**  
Define origin/destination addresses; geocode both to lat/long.

**1.4 Route Duration with Traffic (Google Routes API)**  
Compute driving route duration using traffic-aware routing and a pessimistic traffic model.

**1.5 Arrival-Time Decision & Alarm State Gate**  
Check whether “now + route duration” is **after or equal to 9:00 AM** (meaning it’s time to leave to arrive by 9); then verify the sheet status is still `ini` for today.

**1.6 Notification & State Finalization**  
Send the “Wake Up!” SMS via Twilio and update the sheet status to `end` to prevent further alerts that day.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Today Context

**Overview:**  
Runs on a cron-like schedule on weekdays and produces a `date` string for today used throughout the workflow.

**Nodes involved:**  
- `Triggers at 7 AM till 8 AM every minute on weekdays `  
- `Get today's date`

#### Node: Triggers at 7 AM till 8 AM every minute on weekdays 
- **Type / role:** Schedule Trigger — entry point that runs periodically.
- **Configuration:** Cron expression `* 7 * * 1-5` (every minute during the 7:00 hour, Monday–Friday).
- **Outputs:** Triggers one execution per minute.
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings; if you expect local time, ensure n8n timezone is configured accordingly.
  - The node name says “7 AM till 8 AM” but cron only covers the **7:00 hour** (7:00–7:59).

#### Node: Get today's date
- **Type / role:** Set — creates a normalized “today” date string.
- **Key configuration:**
  - Sets field `date` to: `{{$now.toFormat('yyyy-MM-dd')}}`
- **Inputs:** From Schedule Trigger.
- **Outputs:** JSON item with `date`.
- **Edge cases:**
  - Date formatting depends on Luxon; if locale/timezone is wrong, comparisons against sheet values can fail.

---

### 2.2 Daily State Check/Reset (Google Sheets)

**Overview:**  
Reads a Google Sheet that stores the last processed date and current status. If the sheet is not already set to today, it resets the row to today with status `ini`.

**Nodes involved:**  
- `Get date & status of alarm`  
- `Check if date is today`  
- `Update alarm date & status`

#### Node: Get date & status of alarm
- **Type / role:** Google Sheets — reads sheet rows (operation not explicitly shown in JSON, default is typically “read/getAll” depending on node defaults).
- **Configuration choices:**
  - Document: “Alarm Status” (spreadsheet ID `113zCJm...`)
  - Sheet: `Sheet1` (`gid=0`)
- **Credentials:** Google Sheets OAuth2.
- **Inputs:** From `Get today's date`
- **Outputs:** Sheet data items (must contain at least `date` and `status` columns).
- **Edge cases / failures:**
  - OAuth permission / token expiry.
  - If the sheet is empty or columns mismatch, downstream expressions like `$json.date` / `$json.status` may be undefined.

#### Node: Check if date is today
- **Type / role:** IF — determines whether the sheet date equals today.
- **Conditions (strict validation):**
  - `{{$json.date}}` **equals** `{{ $('Get today\'s date').item.json.date }}`
- **Outputs / branching:**
  - **True branch (date is today):** goes to `Define Origin and Destination addresses`
  - **False branch (date is not today):** goes to `Update alarm date & status`
- **Edge cases:**
  - If the sheet returns multiple rows, this IF runs per item; the workflow assumes a single “state row” behavior but doesn’t explicitly enforce it.

#### Node: Update alarm date & status
- **Type / role:** Google Sheets Update — resets the daily state.
- **Configuration choices:**
  - Operation: **update**
  - Matching column: `row_number`
  - Updates:
    - `row_number: 2`
    - `date: {{ $('Get today\'s date').item.json.date }}`
    - `status: "ini"`
- **Inputs:** From the **false** output of `Check if date is today`
- **Outputs:** Updated row result, then continues to address definition.
- **Edge cases / failures:**
  - If row 2 doesn’t exist, update will fail.
  - If the node’s “Row Number” column is not present/returned by the sheet node, matching can fail.

---

### 2.3 Address Setup & Geocoding (Google Geocoding API)

**Overview:**  
Defines origin/destination addresses (user-provided strings) and geocodes them to coordinates using Google’s Geocoding API.

**Nodes involved:**  
- `Define Origin and Destination addresses`  
- `Get Latitude/Longitude of Origin Address`  
- `Get Latitude/Longitude of End Address`  
- `Merge all trajectories`

#### Node: Define Origin and Destination addresses
- **Type / role:** Set — injects the addresses used for routing.
- **Configuration:**
  - `start_address`: empty string (must be filled)
  - `end_address`: empty string (must be filled)
- **Inputs:** From `Check if date is today` (true path) OR from `Update alarm date & status`
- **Outputs:** The two fields, then fans out to both geocode calls.
- **Edge cases:**
  - Empty or malformed addresses produce zero geocoding results and break downstream coordinate expressions.

#### Node: Get Latitude/Longitude of Origin Address
- **Type / role:** HTTP Request — Google Geocoding API lookup.
- **Configuration:**
  - GET URL expression:  
    `https://maps.googleapis.com/maps/api/geocode/json?address={{ $json.start_address }}&key=...`
  - API key is hard-coded in the URL in the provided workflow (security risk).
- **Inputs:** From `Define Origin and Destination addresses`
- **Outputs:** Geocoding JSON; downstream expects `results[0].geometry.location.{lat,lng}`.
- **Edge cases / failures:**
  - Rate limits, invalid key, billing not enabled.
  - `results` array can be empty; referencing `[0]` will error or yield undefined.

#### Node: Get Latitude/Longitude of End Address
- **Type / role:** HTTP Request — Google Geocoding API lookup (destination).
- **Configuration:** Same pattern using `{{$json.end_address}}`.
- **Inputs:** From `Define Origin and Destination addresses`
- **Outputs:** Destination geocode JSON.
- **Edge cases:** Same as origin geocoding.

#### Node: Merge all trajectories
- **Type / role:** Merge — combines origin and destination geocode results into one item.
- **Configuration:**
  - Mode: **combine**
  - Combine by: **position**
- **Inputs:**
  - Input 1: `Get Latitude/Longitude of Origin Address`
  - Input 2: `Get Latitude/Longitude of End Address`
- **Outputs:** Single combined item to feed routing calculation.
- **Edge cases:**
  - If one geocode call fails/returns no items, combine-by-position can stall or output incomplete merges.

---

### 2.4 Route Duration with Traffic (Google Routes API)

**Overview:**  
Computes a driving route using Google Routes API v2, requesting traffic-aware optimal routing and returning duration/distance/polyline.

**Nodes involved:**  
- `Get real duration between the two addresses`

#### Node: Get real duration between the two addresses
- **Type / role:** HTTP Request — POST to Google Routes computeRoutes.
- **Configuration choices:**
  - URL: `https://routes.googleapis.com/directions/v2:computeRoutes`
  - Method: POST, raw JSON body.
  - Body uses lat/lng from the two geocoding nodes:
    - Origin latitude/longitude from `$('Get Latitude/Longitude of Origin Address').item.json.results[0].geometry.location`
    - Destination latitude/longitude from `$('Get Latitude/Longitude of End Address').item.json.results[0].geometry.location`
  - Travel:
    - `travelMode`: `DRIVE`
    - `routingPreference`: `TRAFFIC_AWARE_OPTIMAL`
    - `trafficModel`: `PESSIMISTIC`
    - `departureTime`: `{{$now.plus({ minutes: 20 })}}`
  - Headers:
    - `X-Goog-Api-Key`: placeholder `AIzaYOUR_GOOGLE_API_KEY_HERE` (must be replaced)
    - `X-Goog-FieldMask`: `routes.duration,routes.distanceMeters,routes.polyline.encodedPolyline`
- **Inputs:** From `Merge all trajectories`
- **Outputs:** Expects `routes[0].duration` (typically an ISO 8601 duration string like `"1234s"` depending on API response format/field mask).
- **Edge cases / failures:**
  - If the API returns duration in a non-integer format (e.g., `"1234s"`), downstream `parseInt()` works but may be fragile if format changes.
  - Invalid API key, Routes API not enabled, billing issues.
  - Departure time set to “now + 20 minutes” may not match your intended “leave immediately” use case.

---

### 2.5 Arrival-Time Decision & Alarm State Gate

**Overview:**  
Determines whether leaving now would make arrival at/after 9:00 AM (so it’s time to trigger), then checks the Google Sheet to ensure the alarm hasn’t already fired today (`status == ini`).

**Nodes involved:**  
- `Total duration < 9:00 AM?`  
- `Check date & status of alarm`  
- `Check if status is equal to 'ini'`

#### Node: Total duration < 9:00 AM?
- **Type / role:** IF — compares computed arrival time against 9:00 AM today.
- **Condition used:**
  - Left value:  
    ```
    {{
      $now.plus({
        seconds: parseInt($json.routes[0].duration, 10)
      }).toISO()
    }}
    ```
  - Operator: `afterOrEquals`
  - Right value: `{{ $now.set({ hour: 9, minute: 00, second: 0, millisecond: 0 }) }}`
- **Important behavior note:**  
  Despite its label (“Total duration < 9:00 AM?”), the logic checks **arrival >= 9:00 AM**. This effectively means: *“If I leave now (plus duration), I would arrive at/after 9:00; therefore it’s time to act.”*
- **Inputs:** From `Get real duration between the two addresses`
- **Outputs:** Only the **true** output is connected (to `Check date & status of alarm`).
- **Edge cases:**
  - If `routes[0].duration` is missing, parsing fails.
  - This compares against **today at 9:00**; if run after 9:00, it will always be true.

#### Node: Check date & status of alarm
- **Type / role:** Google Sheets — re-reads the sheet state right before sending.
- **Purpose:** Avoid sending if another parallel execution already updated the status.
- **Inputs:** From the true path of `Total duration < 9:00 AM?`
- **Outputs:** Rows with `date` and `status`.
- **Edge cases:** Same as earlier sheet read (multiple rows, missing columns, auth).

#### Node: Check if status is equal to 'ini'
- **Type / role:** IF — gates the send based on state row.
- **Conditions (both must match):**
  - `{{$json.status}}` equals `"ini"`
  - `{{$json.date}}` equals `{{ $('Get today\'s date').item.json.date }}`
- **Outputs:**
  - True -> `Send an SMS/MMS/WhatsApp message`
  - False path is not connected (workflow ends).
- **Edge cases:**
  - If multiple sheet rows exist, one row matching `ini` could still trigger even if another row indicates otherwise; workflow assumes a single authoritative row.

---

### 2.6 Notification & State Finalization

**Overview:**  
Sends the SMS alert via Twilio, then updates the sheet status to `end` so subsequent schedule ticks do not send again.

**Nodes involved:**  
- `Send an SMS/MMS/WhatsApp message`  
- `Update alarm status`

#### Node: Send an SMS/MMS/WhatsApp message
- **Type / role:** Twilio — sends a message.
- **Configuration:**
  - `message`: `"Wake Up!"`
  - `to`: empty (must be set)
  - `from`: empty (must be set; must be a Twilio number capable of SMS/WhatsApp depending on channel)
- **Inputs:** From `Check if status is equal to 'ini'` (true path)
- **Outputs:** Twilio send result, then goes to sheet update.
- **Edge cases / failures:**
  - Invalid From/To formatting (E.164 required for SMS).
  - Region/capability restrictions (SMS vs WhatsApp).
  - Twilio auth/permission errors.

#### Node: Update alarm status
- **Type / role:** Google Sheets Update — marks workflow as completed for the day.
- **Configuration:**
  - Operation: update
  - Matching on `row_number`
  - Sets:
    - `row_number: 2`
    - `status: "end"`
- **Inputs:** From Twilio node output.
- **Outputs:** Updated row.
- **Edge cases:**
  - If the Twilio node fails and workflow stops, status may not be updated (risk of repeated sends on next minute).
  - Same row-number existence requirement as earlier.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Triggers at 7 AM till 8 AM every minute on weekdays  | Schedule Trigger | Periodic weekday execution | — | Get today's date | # 🚀 Workflow Overview … (full overview + configuration requirements + optional iOS Shortcuts note) |
| Get today's date | Set | Compute `yyyy-MM-dd` string for today | Triggers at 7 AM till 8 AM every minute on weekdays  | Get date & status of alarm | # 🚀 Workflow Overview … (full overview + configuration requirements + optional iOS Shortcuts note) |
| Get date & status of alarm | Google Sheets | Read alarm “state” (date/status) | Get today's date | Check if date is today | # 🚀 Workflow Overview … (full overview + configuration requirements + optional iOS Shortcuts note) |
| Check if date is today | If | Decide whether to reset sheet state | Get date & status of alarm | Define Origin and Destination addresses; Update alarm date & status | # 🚀 Workflow Overview … (full overview + configuration requirements + optional iOS Shortcuts note) |
| Update alarm date & status | Google Sheets | Reset date=today and status=ini (row 2) | Check if date is today (false) | Define Origin and Destination addresses | # 🚀 Workflow Overview … (full overview + configuration requirements + optional iOS Shortcuts note) |
| Define Origin and Destination addresses | Set | Provide start/end address strings | Check if date is today (true); Update alarm date & status | Get Latitude/Longitude of End Address; Get Latitude/Longitude of Origin Address | ## The origin and destination addresses to be filled based on your preferences |
| Get Latitude/Longitude of Origin Address | HTTP Request | Geocode origin address | Define Origin and Destination addresses | Merge all trajectories | ## The origin and destination addresses to be filled based on your preferences |
| Get Latitude/Longitude of End Address | HTTP Request | Geocode destination address | Define Origin and Destination addresses | Merge all trajectories | ## The origin and destination addresses to be filled based on your preferences |
| Merge all trajectories | Merge | Combine origin+destination geocode results | Get Latitude/Longitude of Origin Address; Get Latitude/Longitude of End Address | Get real duration between the two addresses | ## The origin and destination addresses to be filled based on your preferences |
| Get real duration between the two addresses | HTTP Request | Compute traffic-aware driving route duration | Merge all trajectories | Total duration < 9:00 AM? | ## Time to be at the destination location. To be filled based on your preferences |
| Total duration < 9:00 AM? | If | Check if arrival time is >= 9:00 AM | Get real duration between the two addresses | Check date & status of alarm | ## Time to be at the destination location. To be filled based on your preferences |
| Check date & status of alarm | Google Sheets | Re-check alarm state before sending | Total duration < 9:00 AM? | Check if status is equal to 'ini' | ## Time to be at the destination location. To be filled based on your preferences |
| Check if status is equal to 'ini' | If | Gate to ensure only one send/day | Check date & status of alarm | Send an SMS/MMS/WhatsApp message | ## Time to be at the destination location. To be filled based on your preferences |
| Send an SMS/MMS/WhatsApp message | Twilio | Send “Wake Up!” message | Check if status is equal to 'ini' (true) | Update alarm status | ## Time to be at the destination location. To be filled based on your preferences |
| Update alarm status | Google Sheets | Set status=end (row 2) to prevent repeats | Send an SMS/MMS/WhatsApp message | — | ## Time to be at the destination location. To be filled based on your preferences |
| Sticky Note3 | Sticky Note | Documentation block | — | — | (itself) |
| Sticky Note1 | Sticky Note | Address configuration note | — | — | (itself) |
| Sticky Note | Sticky Note | Target arrival time note | — | — | (itself) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named: *Dynamic traffic alarm via Google Maps and Twilio*.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Rule: interval → **Cron expression** = `* 7 * * 1-5` (every minute during 7 AM, weekdays)
   - Connect to next node.

3) **Add “Get today’s date” (Set)**
   - Node: **Set**
   - Add field `date` (String) = `{{$now.toFormat('yyyy-MM-dd')}}`
   - Connect from Schedule Trigger → this Set.

4) **Prepare Google Sheet (“Alarm Status”)**
   - Create a spreadsheet with columns (row 1 headers): `date`, `status`
   - Ensure row 2 exists and is the single authoritative state row.
   - You may optionally keep `row_number` available via n8n’s Google Sheets node (n8n uses it internally/returned on reads).

5) **Add “Get date & status of alarm” (Google Sheets)**
   - Node: **Google Sheets**
   - Credentials: **Google Sheets OAuth2**
   - Document: select your spreadsheet
   - Sheet: `Sheet1`
   - Operation: configure to **read rows** (commonly “Get All”)
   - Connect: `Get today’s date` → `Get date & status of alarm`

6) **Add “Check if date is today” (IF)**
   - Condition: Date/Time **equals**
     - Left: `{{$json.date}}`
     - Right: `{{ $('Get today\\'s date').item.json.date }}`
   - True output → `Define Origin and Destination addresses`
   - False output → `Update alarm date & status`

7) **Add “Update alarm date & status” (Google Sheets Update)**
   - Operation: **Update**
   - Match column: `row_number`
   - Values:
     - `row_number` = `2`
     - `date` = `{{ $('Get today\\'s date').item.json.date }}`
     - `status` = `ini`
   - Connect from IF false output to this node
   - Then connect to `Define Origin and Destination addresses`

8) **Add “Define Origin and Destination addresses” (Set)**
   - Add `start_address` (String) = your origin address (e.g., “10 Downing St, London”)
   - Add `end_address` (String) = your destination address
   - Connect: IF true output → this node, and Update alarm date & status → this node

9) **Add two Geocoding HTTP Request nodes**
   - Node A: **HTTP Request** “Get Latitude/Longitude of Origin Address”
     - Method: GET
     - URL: `https://maps.googleapis.com/maps/api/geocode/json?address={{ $json.start_address }}&key=YOUR_GEOCODING_API_KEY`
   - Node B: **HTTP Request** “Get Latitude/Longitude of End Address”
     - Method: GET
     - URL: `https://maps.googleapis.com/maps/api/geocode/json?address={{ $json.end_address }}&key=YOUR_GEOCODING_API_KEY`
   - Connect `Define Origin and Destination addresses` → both nodes.

10) **Add Merge**
   - Node: **Merge** named “Merge all trajectories”
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - Origin geocode → Merge input 1
     - Destination geocode → Merge input 2

11) **Add Routes API HTTP Request**
   - Node: **HTTP Request** named “Get real duration between the two addresses”
   - Method: POST
   - URL: `https://routes.googleapis.com/directions/v2:computeRoutes`
   - Send Headers: enabled
     - `Content-Type: application/json`
     - `X-Goog-Api-Key: YOUR_ROUTES_API_KEY`
     - `X-Goog-FieldMask: routes.duration,routes.distanceMeters,routes.polyline.encodedPolyline`
   - Send Body: raw JSON, using expressions for coordinates:
     - Origin lat/lng from the origin geocode node: `results[0].geometry.location.lat/lng`
     - Destination lat/lng similarly
   - Recommended to keep:
     - `travelMode = DRIVE`
     - `routingPreference = TRAFFIC_AWARE_OPTIMAL`
     - `trafficModel = PESSIMISTIC`
     - `departureTime = {{$now.plus({ minutes: 20 })}}` (or change to `$now` if desired)
   - Connect Merge → this node.

12) **Add “Total duration < 9:00 AM?” (IF)**
   - Condition: Date/Time **afterOrEquals**
     - Left: `{{$now.plus({ seconds: parseInt($json.routes[0].duration, 10) }).toISO()}}`
     - Right: `{{$now.set({ hour: 9, minute: 0, second: 0, millisecond: 0 })}}`
   - Connect Routes node → this IF.
   - Use only **true** output onward.

13) **Add “Check date & status of alarm” (Google Sheets)**
   - Same spreadsheet + sheet as earlier
   - Operation: read rows (same as step 5)
   - Connect from IF (true) → this node.

14) **Add “Check if status is equal to 'ini'” (IF)**
   - Condition 1 (String equals): `{{$json.status}}` equals `ini`
   - Condition 2 (Date equals): `{{$json.date}}` equals `{{ $('Get today\\'s date').item.json.date }}`
   - True output → Twilio send node

15) **Add Twilio send node**
   - Node: **Twilio** → “Send SMS/MMS/WhatsApp message”
   - Credentials: Twilio (Account SID + Auth Token)
   - Set:
     - `From`: your Twilio number (or WhatsApp sender)
     - `To`: your phone number (E.164 format)
     - `Message`: `Wake Up!`
   - Connect IF (status ini) true → Twilio.

16) **Add “Update alarm status” (Google Sheets Update)**
   - Operation: **Update**
   - Match column: `row_number`
   - Values:
     - `row_number` = `2`
     - `status` = `end`
   - Connect Twilio → this node.

17) **(Optional) iOS Shortcuts automation**
   - On iPhone: create a Personal Automation:
     - Trigger: When message contains “Wake Up!”
     - Action: Set Alarm / enable Focus / etc.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This automation bridges the gap between your schedule and the unpredictable reality of morning traffic…” | Workflow sticky note “Workflow Overview” (embedded documentation) |
| Google Sheets columns required: `Date (YYYY-MM-DD)`, `Status (ini/end)` | Embedded configuration requirements in sticky note |
| Google Maps API requirement: Distance Matrix mentioned, but implementation uses **Geocoding API** + **Routes API (computeRoutes)** | Ensure the correct APIs are enabled in Google Cloud |
| Twilio: requires Account SID/Auth Token + From/To numbers | Embedded configuration requirements in sticky note |
| iOS Shortcuts optional: trigger when SMS contains “Wake up!” then set alarm | Embedded configuration requirements in sticky note |
| Author credit line: “Thank me later for waking up on time! -- Yorgo Haber” | Embedded documentation in sticky note |

