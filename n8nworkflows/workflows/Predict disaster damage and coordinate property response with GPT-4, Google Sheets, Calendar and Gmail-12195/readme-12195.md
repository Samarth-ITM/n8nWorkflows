Predict disaster damage and coordinate property response with GPT-4, Google Sheets, Calendar and Gmail

https://n8nworkflows.xyz/workflows/predict-disaster-damage-and-coordinate-property-response-with-gpt-4--google-sheets--calendar-and-gmail-12195


# Predict disaster damage and coordinate property response with GPT-4, Google Sheets, Calendar and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Predict disaster damage and coordinate property response with GPT-4, Google Sheets, Calendar and Gmail  
**Workflow name (in JSON):** Autonomous Disaster Response and Property Damage Prediction System  
**Purpose:** Periodically monitor multiple disaster alert sources (weather API + seismic and flood RSS), identify properties potentially impacted based on a property database in Google Sheets, estimate damages and insurance claims, schedule maintenance visits in Google Calendar, and send AI-assisted incident reports via Gmail to property owners and insurers.

### 1.1 Alert ingestion & configuration
- Runs every hour.
- Loads workflow configuration (API keys, feeds, sheets, monitored coordinates, insurer email).
- Fetches alerts from OpenWeather (One Call) + USGS/NOAA RSS feeds.
- Merges alert streams.

### 1.2 Alert normalization & severity filtering
- Parses heterogeneous alert payloads into a common schema.
- Filters to “active + severe/extreme/warning” alerts only.
- Extracts area/coordinates/description and timestamps.

### 1.3 Property impact identification & mapping
- Loads/updates properties from Google Sheets.
- Checks each property against each active alert (simple distance/area matching).
- Outputs only affected properties, with attached relevant alerts and computed “highestSeverity”.

### 1.4 AI damage prediction + claim estimation
- Sends affected properties into a LangChain Agent powered by GPT‑4 (node is present but prompt/tools are not configured in the JSON).
- Independently computes estimated repair cost and claim amount (deductible applied), classifies damage level, and schedules a maintenance time window.

### 1.5 Maintenance scheduling & stakeholder notifications
- Creates a Google Calendar event for the maintenance window.
- Sends AI-generated reports (agent node present, but prompt/tools are not configured) via Gmail:
  - To each property owner
  - To an insurer email configured in workflow config

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling + Workflow Configuration + Alert Fetching

**Overview:** Triggers hourly, sets configuration values, and fetches weather/seismic/flood alerts from configured sources.

**Nodes involved:**
- ⏰ Check for Disaster Alerts
- ⚙️ Workflow Configuration
- 🌤️ Fetch Weather Alerts
- 🌍 Fetch Seismic Alerts (RSS)
- 💧 Fetch Flood Alerts (RSS)
- 🔀 Combine All Alerts

#### Node: ⏰ Check for Disaster Alerts
- **Type / role:** Schedule Trigger; entry point.
- **Configuration:** Runs on an interval of **every 1 hour**.
- **Connections:** Outputs to **⚙️ Workflow Configuration**.
- **Edge cases/failures:** None typical (unless n8n scheduler disabled or workflow inactive).

#### Node: ⚙️ Workflow Configuration
- **Type / role:** Set node; centralizes configuration values consumed by later nodes.
- **Configuration choices:** In the provided JSON, the Set node has **no explicit fields defined** (empty options). However, downstream nodes reference these fields via expressions, meaning you must add them manually for the workflow to function.
- **Expected variables referenced elsewhere:**
  - `monitoredLocations` (string formatted like `"lat,lon|lat,lon|..."`)
  - `openWeatherApiKey`
  - `usgsEarthquakeFeed` (RSS URL)
  - `noaaFloodFeed` (RSS URL)
  - `propertySheetId`
  - `propertySheetName`
  - `insurerEmail`
- **Connections:** Sends output in parallel to all three fetch nodes.
- **Edge cases/failures:**
  - If required fields are missing, expressions in downstream nodes will evaluate to `undefined` and cause wrong URLs/requests or blank recipients.

#### Node: 🌤️ Fetch Weather Alerts
- **Type / role:** HTTP Request; pulls weather data from OpenWeather One Call endpoint.
- **Configuration:**
  - **URL:** `https://api.openweathermap.org/data/2.5/onecall`
  - **Auth:** “genericCredentialType” with `queryAuth` behavior (but API key is actually passed via query parameter `appid`).
  - **Query parameters (expression-driven):**
    - `lat` = first monitored location latitude: `monitoredLocations.split('|')[0].split(',')[0]`
    - `lon` = first monitored location longitude: `...split(',')[1]`
    - `exclude` = `minutely,hourly,daily`
    - `appid` = `openWeatherApiKey`
  - **Response handling:** `neverError: true` + JSON response format. This prevents the node from throwing on HTTP errors, but you must handle bad responses downstream.
- **Connections:** To **🔀 Combine All Alerts** (input index 0).
- **Edge cases/failures:**
  - Invalid API key → JSON error payload; workflow continues due to `neverError`, but downstream parsing may output no alerts.
  - `monitoredLocations` format issues (missing `|` or `,`) → expression errors or wrong lat/lon.
  - One Call API plan limitations/endpoint changes could return different structures.

#### Node: 🌍 Fetch Seismic Alerts (RSS)
- **Type / role:** RSS Feed Read; reads earthquake feed items.
- **Configuration:** URL from `usgsEarthquakeFeed`.
- **Connections:** To **🔀 Combine All Alerts** (input index 1).
- **Edge cases/failures:** Invalid/slow RSS URL, network errors, empty feed.

#### Node: 💧 Fetch Flood Alerts (RSS)
- **Type / role:** RSS Feed Read; reads flood feed items.
- **Configuration:** URL from `noaaFloodFeed`.
- **Connections:** **Not connected** to the Merge node in the provided connections graph.
  - This is a functional issue: flood alerts are fetched but never merged/used.
- **Edge cases/failures:** Same as RSS above; also “dead branch” risk since results are ignored.

#### Node: 🔀 Combine All Alerts
- **Type / role:** Merge node in **combine** mode; combines multiple alert streams into one item.
- **Configuration:** `mode: combine` (merges input data into a single output).
- **Connections:** Outputs to **🔍 Parse and Filter Active Alerts**.
- **Edge cases/failures:**
  - If one input is missing/empty, combined output shape can vary.
  - Because **💧 Fetch Flood Alerts (RSS)** isn’t connected, flood data won’t appear unless you connect it.

---

### Block 2 — Alert Normalization & Filtering

**Overview:** Converts alerts to a unified schema and keeps only active and severe alerts, outputting one item per affected area.

**Nodes involved:**
- 🔍 Parse and Filter Active Alerts

#### Node: 🔍 Parse and Filter Active Alerts
- **Type / role:** Code node (JavaScript); normalizes and filters alert objects.
- **Key logic:**
  - Reads from `$input.first().json.weatherAlerts`, `.seismicAlerts`, `.floodAlerts`.
  - Combines them into `allAlerts`, tagging each with `type`.
  - Filters to alerts considered:
    - **Active:** `status === 'active'` OR `active === true`
    - **Severe:** JSON string contains `severe|extreme|warning`
  - Maps to `affectedAreas` schema:
    - `alertId`, `alertType`, `severity`, `area`, `coordinates`, `description`, `timestamp`
  - Returns **one output item per area**.
- **Connections:** Outputs to **📊 Get Property Database**.
- **Critical integration note:** The merge node output does **not** inherently create keys named `weatherAlerts`, `seismicAlerts`, `floodAlerts`. With the current upstream nodes:
  - OpenWeather returns a structure like `alerts` (if present), not `weatherAlerts`.
  - RSS nodes return arrays of items, not `seismicAlerts`/`floodAlerts`.
  - Therefore, as written, this Code node will likely see empty arrays and output nothing unless you transform upstream data into those keys.
- **Edge cases/failures:**
  - If `$input.first().json` doesn’t have those keys, it safely falls back to empty arrays, but the workflow effectively does nothing.
  - Severity detection via substring search can produce false positives/negatives.

---

### Block 3 — Property Database + Impact Mapping

**Overview:** Uses Google Sheets as the property database, then checks which properties fall into affected areas and emits only impacted properties.

**Nodes involved:**
- 📊 Get Property Database
- 🗺️ Map Affected Properties

#### Node: 📊 Get Property Database
- **Type / role:** Google Sheets node; configured as **appendOrUpdate**.
- **Configuration:**
  - **Operation:** Append or Update
  - **Matching column:** `property_id`
  - **Sheet:** name from `propertySheetName`
  - **Document ID:** from `propertySheetId`
  - **Mapping mode:** auto-map input data
- **Connections:** Outputs to **🗺️ Map Affected Properties**.
- **Important behavior note:** “appendOrUpdate” is typically used to write rows, not to read the full database. However, the next Code node calls `$('📊 Get Property Database').all()` as if it contains the full property list. Unless this node is actually being fed property rows to upsert (not shown), this may not behave as intended.
- **Edge cases/failures:**
  - Google auth/permission failures.
  - Sheet name/id mismatch.
  - If no incoming `property_id` exists, updates can fail or append duplicates.

#### Node: 🗺️ Map Affected Properties
- **Type / role:** Code node; joins alerts to properties and determines which properties are impacted.
- **Key logic:**
  - `const alerts = $input.first().json;` (only the first incoming alert item is read)
  - `const properties = $('📊 Get Property Database').all();`
  - `isPropertyAffected`:
    - If coordinates + property lat/long exist: Euclidean distance threshold `< 0.5` (approx comment says “~50km”, but this is not a correct geodesic conversion).
    - Else: checks if `alert.area` contains `property.city`.
  - For each property, loops alerts (but alerts variable is likely a single object, so it wraps into `[alerts]`).
  - Outputs items only for properties with `relevantAlerts.length > 0`.
  - Computes `highestSeverity` as numeric max derived from `severity` strings.
- **Connections:** Outputs to **🤖 Damage Prediction Agent**.
- **Edge cases/failures:**
  - Because it reads `$input.first()` it may ignore other alert items. If the previous node outputs multiple alert items, they won’t all be considered.
  - If property data doesn’t include `latitude/longitude/city`, matching quality degrades.
  - Severity mapping assumes exact strings `extreme`/`severe`; other values become “1”.

---

### Block 4 — AI Damage Prediction + Claim Calculation

**Overview:** Passes impacted properties through an AI agent (GPT‑4) and then computes deterministic claim values and maintenance scheduling windows.

**Nodes involved:**
- 🤖 Damage Prediction Agent
- 🧠 OpenAI GPT-4
- 💰 Calculate Insurance Claims

#### Node: 🧠 OpenAI GPT-4
- **Type / role:** LangChain Chat Model (OpenAI).
- **Configuration:** Options empty in JSON; must set:
  - Model (e.g., `gpt-4` / `gpt-4o` depending on your n8n/OpenAI availability)
  - OpenAI credentials (API key)
- **Connections:** Connected via `ai_languageModel` to **🤖 Damage Prediction Agent**.
- **Edge cases/failures:** Invalid key, model not available, rate limits, timeouts.

#### Node: 🤖 Damage Prediction Agent
- **Type / role:** LangChain Agent; intended to produce a damage assessment.
- **Configuration:** Empty options in JSON; **no prompt, tools, or output schema are defined** here.
- **Connections:**
  - Input: from **🗺️ Map Affected Properties**
  - AI model: from **🧠 OpenAI GPT-4**
  - Output: to **💰 Calculate Insurance Claims**
- **Practical implication:** As configured, it may pass through input or output unpredictable content depending on defaults. For deterministic automation, you’d normally configure:
  - System instructions/prompt
  - Structured output (JSON)
  - Tooling (optional)
- **Edge cases/failures:** Non-JSON responses, hallucinated fields, token limits if property/alerts are large.

#### Node: 💰 Calculate Insurance Claims
- **Type / role:** Code node; deterministic claim/maintenance calculations.
- **Key outputs added per property:**
  - `estimated_repair_cost` = property value × severity multiplier (30%/15%/5%)
  - `insurance_claim_amount` = max(0, estimatedDamage − deductible)
  - `damage_level` = Critical/Severe/Moderate based on severity
  - Maintenance window:
    - Critical: starts ~4 hours
    - Severe: starts ~24 hours
    - Moderate: starts ~72 hours
    - Duration: 2 hours
  - `disaster_type` from first alert’s type
  - `disaster_description` concatenation of alert descriptions
- **Connections:** Outputs to **📅 Schedule Maintenance Teams**.
- **Edge cases/failures:**
  - Missing `property_value` / `insurance_deductible` handled via defaults.
  - `highestSeverity` missing → defaults to 5% logic.
  - If `affectedAlerts` missing or empty, `disaster_type` becomes `unknown`.

---

### Block 5 — Maintenance Scheduling + AI Report Generation + Emailing

**Overview:** Creates calendar events for inspections and sends email reports to owners and insurers. An AI agent is present for report generation but not configured in JSON.

**Nodes involved:**
- 📅 Schedule Maintenance Teams
- 📝 Report Generation Agent
- 🧠 OpenAI GPT-4 for Reports
- ✉️ Send Report to Property Owners
- ✉️ Send Report to Insurers

#### Node: 📅 Schedule Maintenance Teams
- **Type / role:** Google Calendar node; creates an event (implicitly; operation not shown but node requires calendar and start/end).
- **Configuration:**
  - `start` = `{{$json.maintenance_start_time}}`
  - `end` = `{{$json.maintenance_end_time}}`
  - `calendar` value is empty in JSON → must select a calendar.
- **Connections:** To **📝 Report Generation Agent**.
- **Edge cases/failures:** Calendar not selected, auth failures, invalid ISO timestamps, timezone expectations.

#### Node: 🧠 OpenAI GPT-4 for Reports
- **Type / role:** LangChain Chat Model (OpenAI) dedicated to reporting.
- **Configuration:** Empty; must set model + credentials.
- **Connections:** Provides `ai_languageModel` to **📝 Report Generation Agent**.
- **Edge cases/failures:** Same as other OpenAI model node.

#### Node: 📝 Report Generation Agent
- **Type / role:** LangChain Agent to generate structured incident/impact reports.
- **Configuration:** Empty in JSON; no prompt/schema defined.
- **Connections:**
  - Input: from **📅 Schedule Maintenance Teams**
  - AI model: from **🧠 OpenAI GPT-4 for Reports**
  - Outputs: to both Gmail nodes (owners + insurer)
- **Practical implication:** The Gmail nodes expect fields like `damage_assessment_details` and `recommended_actions`, but those fields are not created anywhere else except potentially by this Agent—so you must configure the agent to output them.
- **Edge cases/failures:** Missing expected fields → email templates show blanks; non-structured output.

#### Node: ✉️ Send Report to Property Owners
- **Type / role:** Gmail node; sends HTML email to each property owner.
- **Configuration:**
  - **To:** `{{$json.property_owner_email}}`
  - **Subject:** “🚨 Property Alert: … {{ property_address }}”
  - **Message:** HTML template referencing:
    - `property_owner_name`, `property_address`
    - `damage_level`, `estimated_repair_cost`, `insurance_claim_amount`
    - `disaster_description`, `recommended_actions`
- **Connections:** terminal.
- **Edge cases/failures:** Missing owner email/name/address; Gmail credential/scopes; rate limits; HTML injection risk if data not sanitized.

#### Node: ✉️ Send Report to Insurers
- **Type / role:** Gmail node; sends claim notice to insurer.
- **Configuration:**
  - **To:** `{{ $('⚙️ Workflow Configuration').item.json.insurerEmail }}`
  - **Subject:** “Insurance Claim Notification: {{ property_address }}”
  - **Message:** HTML referencing:
    - `property_id`, `property_address`, `property_owner_name`
    - `estimated_repair_cost`, `insurance_claim_amount`, `damage_level`, `disaster_type`
    - `damage_assessment_details` (expected from the report agent)
- **Connections:** terminal.
- **Edge cases/failures:** Missing insurerEmail in config; missing `damage_assessment_details`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏰ Check for Disaster Alerts | scheduleTrigger | Hourly workflow trigger | — | ⚙️ Workflow Configuration | ## How It Works  This automated disaster response workflow streamlines emergency management by monitoring multiple alert sources and coordinating property protection teams. Designed for property managers, insurance companies, and emergency response organizations, it solves the critical challenge of rapidly identifying at-risk properties and deploying resources during disasters.The system continuously monitors weather, seismic, and flood alerts from authoritative sources. When threats are detected, it cross-references property databases to identify affected locations, calculates insurance exposure, and generates damage assessments using OpenAI's GPT-4. Teams receive automated maintenance schedules while property owners and insurers get instant email notifications with comprehensive reports. This eliminates manual monitoring, reduces response time from hours to minutes, and ensures no vulnerable properties are overlooked during emergencies. |
| ⚙️ Workflow Configuration | set | Stores shared configuration variables | ⏰ Check for Disaster Alerts | 🌤️ Fetch Weather Alerts; 🌍 Fetch Seismic Alerts (RSS); 💧 Fetch Flood Alerts (RSS) | ## How It Works  This automated disaster response workflow streamlines emergency management by monitoring multiple alert sources and coordinating property protection teams. Designed for property managers, insurance companies, and emergency response organizations, it solves the critical challenge of rapidly identifying at-risk properties and deploying resources during disasters.The system continuously monitors weather, seismic, and flood alerts from authoritative sources. When threats are detected, it cross-references property databases to identify affected locations, calculates insurance exposure, and generates damage assessments using OpenAI's GPT-4. Teams receive automated maintenance schedules while property owners and insurers get instant email notifications with comprehensive reports. This eliminates manual monitoring, reduces response time from hours to minutes, and ensures no vulnerable properties are overlooked during emergencies. |
| 🌤️ Fetch Weather Alerts | httpRequest | Pulls weather alerts/data from OpenWeather | ⚙️ Workflow Configuration | 🔀 Combine All Alerts | ## How It Works  This automated disaster response workflow streamlines emergency management by monitoring multiple alert sources and coordinating property protection teams. Designed for property managers, insurance companies, and emergency response organizations, it solves the critical challenge of rapidly identifying at-risk properties and deploying resources during disasters.The system continuously monitors weather, seismic, and flood alerts from authoritative sources. When threats are detected, it cross-references property databases to identify affected locations, calculates insurance exposure, and generates damage assessments using OpenAI's GPT-4. Teams receive automated maintenance schedules while property owners and insurers get instant email notifications with comprehensive reports. This eliminates manual monitoring, reduces response time from hours to minutes, and ensures no vulnerable properties are overlooked during emergencies. |
| 🌍 Fetch Seismic Alerts (RSS) | rssFeedRead | Pulls earthquake alerts from RSS | ⚙️ Workflow Configuration | 🔀 Combine All Alerts | ## How It Works  This automated disaster response workflow streamlines emergency management by monitoring multiple alert sources and coordinating property protection teams. Designed for property managers, insurance companies, and emergency response organizations, it solves the critical challenge of rapidly identifying at-risk properties and deploying resources during disasters.The system continuously monitors weather, seismic, and flood alerts from authoritative sources. When threats are detected, it cross-references property databases to identify affected locations, calculates insurance exposure, and generates damage assessments using OpenAI's GPT-4. Teams receive automated maintenance schedules while property owners and insurers get instant email notifications with comprehensive reports. This eliminates manual monitoring, reduces response time from hours to minutes, and ensures no vulnerable properties are overlooked during emergencies. |
| 💧 Fetch Flood Alerts (RSS) | rssFeedRead | Pulls flood alerts from RSS | ⚙️ Workflow Configuration | — | ## How It Works  This automated disaster response workflow streamlines emergency management by monitoring multiple alert sources and coordinating property protection teams. Designed for property managers, insurance companies, and emergency response organizations, it solves the critical challenge of rapidly identifying at-risk properties and deploying resources during disasters.The system continuously monitors weather, seismic, and flood alerts from authoritative sources. When threats are detected, it cross-references property databases to identify affected locations, calculates insurance exposure, and generates damage assessments using OpenAI's GPT-4. Teams receive automated maintenance schedules while property owners and insurers get instant email notifications with comprehensive reports. This eliminates manual monitoring, reduces response time from hours to minutes, and ensures no vulnerable properties are overlooked during emergencies. |
| 🔀 Combine All Alerts | merge | Combines alert streams into one payload | 🌤️ Fetch Weather Alerts; 🌍 Fetch Seismic Alerts (RSS) | 🔍 Parse and Filter Active Alerts | ## How It Works  This automated disaster response workflow streamlines emergency management by monitoring multiple alert sources and coordinating property protection teams. Designed for property managers, insurance companies, and emergency response organizations, it solves the critical challenge of rapidly identifying at-risk properties and deploying resources during disasters.The system continuously monitors weather, seismic, and flood alerts from authoritative sources. When threats are detected, it cross-references property databases to identify affected locations, calculates insurance exposure, and generates damage assessments using OpenAI's GPT-4. Teams receive automated maintenance schedules while property owners and insurers get instant email notifications with comprehensive reports. This eliminates manual monitoring, reduces response time from hours to minutes, and ensures no vulnerable properties are overlooked during emergencies. |
| 🔍 Parse and Filter Active Alerts | code | Normalizes alerts and filters active/severe ones | 🔀 Combine All Alerts | 📊 Get Property Database | ## Setup Steps  1. Configure alert fetch nodes with weather/seismic/flood   2. Connect property database credentials   3. Add OpenAI API key for GPT-4 damage assessments 4. Set up Gmail/SMTP credentials for owner 5. Customize insurance calculation formulas |
| 📊 Get Property Database | googleSheets | Writes/updates property rows (configured) and supplies data to mapping | 🔍 Parse and Filter Active Alerts | 🗺️ Map Affected Properties | ## Setup Steps  1. Configure alert fetch nodes with weather/seismic/flood   2. Connect property database credentials   3. Add OpenAI API key for GPT-4 damage assessments 4. Set up Gmail/SMTP credentials for owner 5. Customize insurance calculation formulas |
| 🗺️ Map Affected Properties | code | Matches alerts to properties and outputs affected properties | 📊 Get Property Database | 🤖 Damage Prediction Agent | ## Setup Steps  1. Configure alert fetch nodes with weather/seismic/flood   2. Connect property database credentials   3. Add OpenAI API key for GPT-4 damage assessments 4. Set up Gmail/SMTP credentials for owner 5. Customize insurance calculation formulas |
| 🧠 OpenAI GPT-4 | lmChatOpenAi | LLM backing the damage prediction agent | — (AI link) | 🤖 Damage Prediction Agent (ai_languageModel) | ## Prerequisites  Weather/seismic/flood alert API access, property database (SQL/Sheets/Airtable)  ## Use Cases  Insurance companies automating claims preparation, property management firms protecting rental portfolios  ## Customization  Modify alert source APIs, adjust damage assessment prompts  ## Benefits  Reduces emergency response time by 90%, eliminates manual alert monitoring |
| 🤖 Damage Prediction Agent | agent | Produces AI damage assessment (not configured in JSON) | 🗺️ Map Affected Properties | 💰 Calculate Insurance Claims | ## Prerequisites  Weather/seismic/flood alert API access, property database (SQL/Sheets/Airtable)  ## Use Cases  Insurance companies automating claims preparation, property management firms protecting rental portfolios  ## Customization  Modify alert source APIs, adjust damage assessment prompts  ## Benefits  Reduces emergency response time by 90%, eliminates manual alert monitoring |
| 💰 Calculate Insurance Claims | code | Computes repair cost, claim amount, maintenance window | 🤖 Damage Prediction Agent | 📅 Schedule Maintenance Teams | ## Maintenance Scheduling  **What:** Automatically creates maintenance tasks for affected properties. **Why:** Rapid task creation reduces response time and minimizes secondary damage. |
| 📅 Schedule Maintenance Teams | googleCalendar | Creates calendar events for maintenance inspections | 💰 Calculate Insurance Claims | 📝 Report Generation Agent | ## Maintenance Scheduling  **What:** Automatically creates maintenance tasks for affected properties. **Why:** Rapid task creation reduces response time and minimizes secondary damage. |
| 🧠 OpenAI GPT-4 for Reports | lmChatOpenAi | LLM backing the report generation agent | — (AI link) | 📝 Report Generation Agent (ai_languageModel) | ## Report Generation (AI)  **What:** Generates structured incident and impact reports using AI. **Why:** Clear, standardized reports ensure stakeholders receive accurate, decision-ready information. |
| 📝 Report Generation Agent | agent | Generates report fields used in emails (not configured in JSON) | 📅 Schedule Maintenance Teams | ✉️ Send Report to Property Owners; ✉️ Send Report to Insurers | ## Report Generation (AI)  **What:** Generates structured incident and impact reports using AI. **Why:** Clear, standardized reports ensure stakeholders receive accurate, decision-ready information. |
| ✉️ Send Report to Property Owners | gmail | Emails property owners with assessment and next steps | 📝 Report Generation Agent | — | ## Stakeholder Notifications  **What:** Sends reports to property owners and insurers via email. **Why:** Timely communication builds trust and enables faster recovery actions. |
| ✉️ Send Report to Insurers | gmail | Emails insurer with claim information | 📝 Report Generation Agent | — | ## Stakeholder Notifications  **What:** Sends reports to property owners and insurers via email. **Why:** Timely communication builds trust and enables faster recovery actions. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it “Autonomous Disaster Response and Property Damage Prediction System”.

2. **Add Trigger**
   - Add **Schedule Trigger** node named **⏰ Check for Disaster Alerts**.
   - Set interval to **Every 1 hour**.

3. **Add configuration node**
   - Add **Set** node named **⚙️ Workflow Configuration**.
   - Add fields (recommended) in the Set node:
     - `monitoredLocations` (e.g., `37.7749,-122.4194|34.0522,-118.2437`)
     - `openWeatherApiKey`
     - `usgsEarthquakeFeed` (e.g., a USGS RSS feed URL)
     - `noaaFloodFeed` (NOAA flood RSS URL)
     - `propertySheetId`
     - `propertySheetName`
     - `insurerEmail`
   - Connect: **⏰ Check for Disaster Alerts → ⚙️ Workflow Configuration**.

4. **Add alert fetch nodes**
   - Add **HTTP Request** node named **🌤️ Fetch Weather Alerts**:
     - URL: `https://api.openweathermap.org/data/2.5/onecall`
     - Query params:
       - `lat` = first lat from `monitoredLocations`
       - `lon` = first lon from `monitoredLocations`
       - `exclude=minutely,hourly,daily`
       - `appid={{openWeatherApiKey}}`
     - Response: JSON; enable “Never Error” if you want soft-fail behavior.
   - Add **RSS Feed Read** node named **🌍 Fetch Seismic Alerts (RSS)** with URL `={{ $json.usgsEarthquakeFeed }}` from config.
   - Add **RSS Feed Read** node named **💧 Fetch Flood Alerts (RSS)** with URL `={{ $json.noaaFloodFeed }}` from config.
   - Connect: **⚙️ Workflow Configuration →** each fetch node.

5. **Merge alert streams**
   - Add **Merge** node named **🔀 Combine All Alerts**, mode **Combine**.
   - Connect:
     - **🌤️ Fetch Weather Alerts → 🔀 Combine All Alerts (Input 1)**
     - **🌍 Fetch Seismic Alerts (RSS) → 🔀 Combine All Alerts (Input 2)**
     - Also connect **💧 Fetch Flood Alerts (RSS)** to the merge (recommended), because the provided JSON currently does not.

6. **Normalize and filter alerts**
   - Add **Code** node named **🔍 Parse and Filter Active Alerts**.
   - Paste/adapt the JS logic to match your upstream payloads.
   - Critical: ensure the merged payload provides arrays under keys you actually use (you may need to map OpenWeather `alerts` into `weatherAlerts`, and RSS items into `seismicAlerts`/`floodAlerts` before filtering).
   - Connect: **🔀 Combine All Alerts → 🔍 Parse and Filter Active Alerts**.

7. **Connect property database (Google Sheets)**
   - Add **Google Sheets** node named **📊 Get Property Database**.
   - Set credentials (Google OAuth2 or Service Account).
   - Set Document ID = config value, Sheet name = config value.
   - Decide the correct operation:
     - If you truly need to *read* all properties, use a **Read / Get Many** operation (recommended).
     - The JSON uses **Append or Update** with `property_id` as matching column, which is primarily for writes.
   - Connect: **🔍 Parse and Filter Active Alerts → 📊 Get Property Database**.

8. **Map affected properties**
   - Add **Code** node named **🗺️ Map Affected Properties**.
   - Implement property-vs-alert matching (distance/area).
   - Ensure you are iterating over **all incoming alert items**, not only `$input.first()`, if you want complete coverage.
   - Connect: **📊 Get Property Database → 🗺️ Map Affected Properties**.

9. **Set up AI for damage prediction**
   - Add **OpenAI Chat Model** node (LangChain) named **🧠 OpenAI GPT-4**.
     - Configure OpenAI credentials.
     - Choose model (e.g., GPT‑4 class model available in your account).
   - Add **AI Agent** node named **🤖 Damage Prediction Agent**.
     - Attach **🧠 OpenAI GPT-4** as the Agent’s Language Model connection.
     - Configure the agent prompt to output structured fields if you need them downstream.
   - Connect: **🗺️ Map Affected Properties → 🤖 Damage Prediction Agent**.

10. **Calculate claims and maintenance window**
    - Add **Code** node named **💰 Calculate Insurance Claims** with the provided claim formula logic (or your own).
    - Connect: **🤖 Damage Prediction Agent → 💰 Calculate Insurance Claims**.

11. **Schedule maintenance in Google Calendar**
    - Add **Google Calendar** node named **📅 Schedule Maintenance Teams**.
    - Configure Google Calendar credentials.
    - Select a Calendar.
    - Map:
      - Start: `{{$json.maintenance_start_time}}`
      - End: `{{$json.maintenance_end_time}}`
    - Connect: **💰 Calculate Insurance Claims → 📅 Schedule Maintenance Teams**.

12. **Set up AI for report generation**
    - Add **OpenAI Chat Model** node named **🧠 OpenAI GPT-4 for Reports** (separate model node is optional but matches the JSON).
    - Add **AI Agent** node named **📝 Report Generation Agent**.
      - Attach **🧠 OpenAI GPT-4 for Reports** as Language Model.
      - Configure prompt to output fields required by email templates, at minimum:
        - `recommended_actions`
        - `damage_assessment_details`
      - Ensure the agent preserves inputs like owner email/name/address and computed claim fields.
    - Connect: **📅 Schedule Maintenance Teams → 📝 Report Generation Agent**.

13. **Send Gmail notifications**
    - Add **Gmail** node named **✉️ Send Report to Property Owners**:
      - Credentials: Gmail OAuth2 with send scope.
      - To: `{{$json.property_owner_email}}`
      - Subject & HTML body as per your template.
    - Add **Gmail** node named **✉️ Send Report to Insurers**:
      - To: `={{ $('⚙️ Workflow Configuration').item.json.insurerEmail }}`
      - Subject/body referencing claim + AI details.
    - Connect: **📝 Report Generation Agent →** both Gmail nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How It Works**: Automated monitoring of weather/seismic/flood alerts, cross-references a property database, calculates insurance exposure, generates damage assessments using GPT‑4, schedules maintenance teams, and emails owners/insurers. | Sticky note: “📥 Alert Ingestion & Normalization” |
| **Setup Steps**: Configure alert fetch nodes; connect property DB credentials; add OpenAI API key; set up Gmail/SMTP; customize insurance formulas. | Sticky note: “🗺️ Property Impact Identification & Mapping” |
| **Prerequisites / Use Cases / Customization / Benefits**: Needs alert API access + property DB; useful for insurers and property managers; modify sources and prompts; reduces response time. | Sticky note: “🤖 AI Damage & Claim Estimation” |
| **Maintenance Scheduling**: Auto-creates maintenance tasks to reduce response time and secondary damage. | Sticky note: “Maintenance Scheduling” |
| **Report Generation (AI)**: Produces standardized, decision-ready reports for stakeholders. | Sticky note: “Report Generation (AI)” |
| **Stakeholder Notifications**: Email reports to owners and insurers for timely communication. | Sticky note: “Stakeholder Notifications” |