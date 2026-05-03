Create weather-aware morning alarms with Spotify, Gemini AI and OpenMeteo

https://n8nworkflows.xyz/workflows/create-weather-aware-morning-alarms-with-spotify--gemini-ai-and-openmeteo-11153


# Create weather-aware morning alarms with Spotify, Gemini AI and OpenMeteo

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs every morning (intended for ~7AM), checks current local weather (Open-Meteo), verifies a specific Spotify playback device is available, starts playing a **random Led Zeppelin top track**, asks **Google Gemini** to generate a short DJ-style greeting, logs the event to **Google Sheets**, and emails the greeting + album art via **Gmail**. If the Spotify device is not found, it sends an error email instead.

**Target use cases:**
- Smart “music alarm” that adapts messaging to the morning context (song + weather).
- Daily logging of alarms (song, artist, weather, temperature) for history/analytics.
- Example of coordinating Spotify + HTTP API + Gemini + Sheets + Email in n8n.

### 1.1 Logical Blocks
1. **Context & Setup**: Trigger, configuration, fetch Spotify devices, fetch weather.
2. **Device Verification**: Identify the target Spotify device and branch on success/failure.
3. **Music Selection & Playback**: Fetch artist top tracks, pick one randomly, start playback on the device.
4. **AI Persona + Logging + Notification**: Gemini generates DJ line; workflow logs to Sheets; sends formatted email to user.
5. **Failure Path**: If device not found, send an error email.

---

## 2. Block-by-Block Analysis

### Block 1 — Context & Setup
**Overview:**  
Initializes the run, defines location/device/sheet configuration, pulls current weather from Open-Meteo, and retrieves Spotify devices so the workflow can target the correct speaker.

**Nodes Involved:**
- Sticky Note Main
- Sticky Note Setup
- Schedule Trigger (7AM)
- Workflow Configuration
- Get Spotify Devices
- Get Weather (OpenMeteo)

#### Sticky Note Main
- **Type / Role:** Sticky Note (documentation)
- **Configuration choices:** Describes purpose, setup steps, requirements (Spotify Premium, Gemini API key, Sheets/Gmail).
- **Connections:** None
- **Edge cases:** None (non-executing)

#### Sticky Note Setup
- **Type / Role:** Sticky Note (documentation)
- **Configuration choices:** Labels block intent: initialize, fetch weather, verify Spotify device.
- **Connections:** None
- **Edge cases:** None (non-executing)

#### Schedule Trigger (7AM)
- **Type / Role:** `Schedule Trigger` (entry point)
- **Configuration choices:** Runs every **24 hours** (`hoursInterval: 24`).  
  *Note:* Despite the node name “(7AM)”, the JSON shown does not explicitly set a specific hour; it runs based on activation time + interval unless additional schedule settings exist in the UI.
- **Outputs:** → Workflow Configuration
- **Potential failures / edge cases:**
  - Workflow timezone mismatch (n8n instance timezone vs desired local 7AM).
  - Interval scheduling drift if relying purely on 24h intervals.

#### Workflow Configuration
- **Type / Role:** `Set` node (central configuration)
- **Configuration choices:** Sets and keeps other fields:
  - `targetDeviceName`: `"2F"` (Spotify device name to target)
  - `latitude`: `"35.6895"`
  - `longitude`: `"139.6917"`
  - `sheetId`: `"1WOrmZA7kOo9wqTTAv5wyjZ6yY_QbZfhLDe7zS_JdojE"`
- **Key expressions/variables:** Values are static strings here, used downstream via `$('Workflow Configuration').first().json.<field>`.
- **Outputs:**
  - → Get Spotify Devices
  - → Get Weather (OpenMeteo)
- **Edge cases / failures:**
  - Wrong device name causes “device not found” branch.
  - Latitude/longitude stored as strings (Open-Meteo accepts them, but beware if later numeric ops are added).
  - Invalid sheetId leads to Google Sheets auth/not-found errors later.

#### Get Spotify Devices
- **Type / Role:** `Spotify` node (`getDevices`) to list available playback devices
- **Credentials:** Spotify OAuth2 (requires Spotify Premium for playback control)
- **Outputs:** → Find Target Device
- **Edge cases / failures:**
  - OAuth token expired/invalid.
  - No active devices (Spotify may return an empty list).
  - Spotify API rate limits or temporary 5xx.

#### Get Weather (OpenMeteo)
- **Type / Role:** `HTTP Request` to Open-Meteo current weather endpoint
- **Configuration choices:**
  - URL is built via expression using config:
    - `https://api.open-meteo.com/v1/forecast?latitude=<lat>&longitude=<lon>&current_weather=true`
- **Outputs:** → Find Target Device  
  (This means weather data is made available in the execution context; it does not directly influence branching.)
- **Edge cases / failures:**
  - Network/DNS errors from n8n host.
  - Non-200 responses (rate limiting, downtime).
  - If Open-Meteo returns unexpected schema, later expressions referencing `current_weather.*` may fail.

---

### Block 2 — Device Verification
**Overview:**  
Searches Spotify’s returned device list for a device matching the configured name, then branches depending on whether the target device exists.

**Nodes Involved:**
- Find Target Device
- Device Found?

#### Find Target Device
- **Type / Role:** `Code` node (JavaScript) to locate device ID by name
- **Configuration choices (interpreted):**
  - Reads `targetDeviceName` from Workflow Configuration; defaults to `'2F'` if missing.
  - Inspects `$input.all()` expecting Spotify devices structure:
    - Primary expectation: `items[0].json.devices` is an array.
    - Fallback: if `items[0].json` itself is an array, searches that.
  - Outputs:
    - `target_device_id` (string or null)
    - `device_found` (boolean)
    - `target_name` (string)
- **Key variables:**
  - `$('Workflow Configuration').first()` for `targetDeviceName`
- **Inputs:**
  - Receives main input from **Get Spotify Devices**.
  - Additionally, **Get Weather (OpenMeteo)** also connects into this node; multiple inbound connections can change the actual shape/order of `$input` depending on how n8n merges inputs.
- **Outputs:** → Device Found?
- **Edge cases / failures:**
  - **Input merging ambiguity:** With two incoming connections (Spotify devices + weather), `$input.all()` may include items not shaped like Spotify response, depending on execution timing/merge behavior. The code assumes the first item has the devices list; if weather arrives “first”, device detection may fail.
  - Device names are case-sensitive (`d.name === targetName`).
  - Spotify may return devices but without `id` in rare cases; then playback fails later.

#### Device Found?
- **Type / Role:** `IF` node (branching)
- **Condition:** `$json.device_found` is true
- **True output:** → Get Artist Tracks  
- **False output:** → Send Error Email
- **Edge cases / failures:**
  - If `device_found` is undefined due to upstream issues, it will go to false branch.

---

### Block 3 — Music Selection & Playback
**Overview:**  
Fetches Led Zeppelin’s top tracks for a given market, randomly chooses one, and starts Spotify playback on the discovered device.

**Nodes Involved:**
- Sticky Note Music
- Get Artist Tracks
- Pick Random Track
- Start Playback

#### Sticky Note Music
- **Type / Role:** Sticky Note (documentation)
- **Content intent:** Explains top tracks retrieval, random selection, playback start.
- **Connections:** None

#### Get Artist Tracks
- **Type / Role:** `Spotify` node (`artist.getTopTracks`)
- **Configuration choices:**
  - Artist ID: `36QJpDe2go2KgaRleHCDTp` (Led Zeppelin)
  - Country/market: `JP` (affects available tracks and ordering)
- **Credentials:** Spotify OAuth2
- **Outputs:** → Pick Random Track
- **Edge cases / failures:**
  - Market `JP` may restrict some tracks.
  - Spotify API errors/rate limit.
  - Empty result set (rare but possible).

#### Pick Random Track
- **Type / Role:** `Code` node (JavaScript) to randomize selection and shape data for downstream nodes
- **Logic:**
  - Reads all incoming items (tracks) and selects a random one.
  - Builds a single output item containing:
    - `track_uri`, `track_name`, `artist_name`, `album_image`
    - `device_id` from `$('Find Target Device').first().json.target_device_id`
- **Outputs:** → Start Playback
- **Edge cases / failures:**
  - If input has 0 items, returns `[]` and stops downstream execution.
  - Assumes `track.artists[0]` exists and `track.album.images[0].url` exists; may throw if missing.
  - If `device_id` is null, playback likely fails.

#### Start Playback
- **Type / Role:** `Spotify` node (`start` playback)
- **Configuration choices:** Uses the prepared track/device context from previous nodes (implicitly; exact parameter mapping is not shown in the JSON snippet, but operation is `start`).
- **Credentials:** Spotify OAuth2
- **Outputs:** → Message a model
- **Edge cases / failures:**
  - Spotify Premium required.
  - Device must be active/online; otherwise Spotify returns “No active device found” or similar.
  - If the node is not explicitly configured to use `device_id` and `track_uri` fields, it may start playback incorrectly or fail (implementation detail depends on node UI configuration).

---

### Block 4 — AI Persona + Logging + Notification
**Overview:**  
Gemini generates a DJ greeting inspired by the playing track, then the workflow logs song + weather into Google Sheets and emails the user a formatted message including album art.

**Nodes Involved:**
- Sticky Note AI
- Message a model
- Log to Google Sheets
- Send DJ Email

#### Sticky Note AI
- **Type / Role:** Sticky Note (documentation)
- **Content intent:** Explains DJ intro, logging, summary email.
- **Connections:** None

#### Message a model
- **Type / Role:** `@n8n/n8n-nodes-langchain.googleGemini` (Gemini chat completion)
- **Configuration choices:**
  - Model: `models/gemini-2.5-flash`
  - Prompt instructs: “You are a rock & roll DJ… Write a short, high-energy, one-sentence morning greeting… English only.”
  - References:
    - Track name: `$('Pick Random Track').first().json.track_name`
    - Artist name: `$('Pick Random Track').first().json.artist_name`
- **Credentials:** Google Palm / Gemini API credential (`googlePalmApi`)
- **Outputs:** → Log to Google Sheets
- **Edge cases / failures:**
  - Auth/quota errors on Gemini.
  - Model ID availability can vary by region/account.
  - Output field assumption: downstream uses `$('Message a model').first().json.text`—if the node returns a different schema (e.g., `content`), email/logging will break.

#### Log to Google Sheets
- **Type / Role:** `Google Sheets` node (append/update log row)
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document ID: from config `sheetId`
  - Sheet name: `History`
  - Columns mapped explicitly:
    - `date`: `$now.format('yyyy-MM-dd')`
    - `time`: `$now.format('HH:mm')`
    - `song`, `artist`: from Pick Random Track
    - `weather`: `$('Get Weather (OpenMeteo)').first().json.current_weather.weathercode`
    - `temperature`: `$('Get Weather (OpenMeteo)').first().json.current_weather.temperature`
- **Credentials:** Google Sheets OAuth2
- **Outputs:** → Send DJ Email
- **Edge cases / failures:**
  - Sheet/tab “History” missing.
  - Headers must match exactly if using mapping.
  - `appendOrUpdate` may behave unexpectedly if a matching key column isn’t configured in the UI (risk of unintended updates vs append-only).
  - Weather node failure → `current_weather` undefined → expression error.

#### Send DJ Email
- **Type / Role:** `Gmail` node (send formatted HTML email)
- **Configuration choices:**
  - To: `user@example.com`
  - Subject: `🎸 Now Playing: <track_name>`
  - HTML message includes:
    - Gemini output: `$('Message a model').first().json.text`
    - Temperature: from Open-Meteo
    - Album image URL: from Pick Random Track (`<img src="...">`)
- **Credentials:** Gmail OAuth2
- **Outputs:** None (end)
- **Edge cases / failures:**
  - Gmail OAuth expired/insufficient scopes.
  - Some email clients block external images by default.
  - If Gemini output field isn’t `text`, the email will render blank or error.

---

### Block 5 — Failure Path (Device Not Found)
**Overview:**  
If the target Spotify device cannot be located, send an error email to inform the user to check Spotify/device availability.

**Nodes Involved:**
- Send Error Email

#### Send Error Email
- **Type / Role:** `Gmail` node (send failure notification)
- **Configuration choices:**
  - To: `user@example.com`
  - Subject: `🚨 Alarm Failed: Spotify Device Not Found`
  - Body: “Could not find target device. Please check Spotify.”
- **Credentials:** Gmail OAuth2
- **Input:** From Device Found? (false branch)
- **Edge cases / failures:**
  - Gmail credential issues.
  - This does not include diagnostic details (e.g., device list), which can slow troubleshooting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Setup | Sticky Note | Documentation for setup/context block | — | — | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Sticky Note Music | Sticky Note | Documentation for music selection block | — | — | ## 2. Music Selection<br>Retrieves top tracks from Led Zeppelin, selects one at random, and starts playback. |
| Sticky Note AI | Sticky Note | Documentation for AI/log/email block | — | — | ## 3. AI Persona & Log<br>Gemini creates a DJ intro linking weather to the song. Logs history and sends a summary email. |
| Sticky Note Main | Sticky Note | High-level overview + setup requirements | — | — | # Wake Up with Led Zeppelin: Smart Weather DJ 🎸☔️☀️<br><br>## Overview<br>**Turn your alarm into a smart morning briefing.**<br>This workflow checks the local weather, picks a Led Zeppelin track, and uses **Google Gemini** to generate a DJ persona that comments on **both the song and the weather**.<br><br>## How it works<br>1. **Context:** Checks Spotify device & fetches weather (OpenMeteo).<br>2. **Music:** Plays a random Led Zeppelin track.<br>3. **AI DJ:** Gemini generates a greeting linking the song's vibe to the weather.<br>4. **Notify:** Logs to Sheets and sends an email with album art.<br><br>## Setup Steps<br>1. **Credentials:** Connect Spotify, Gemini, Sheets, Gmail.<br>2. **Config:** Open **"Workflow Configuration"** to set location and device name.<br>3. **Sheets:** Create headers: `date`, `time`, `weather`, `temperature`, `song`, `artist`.<br><br>## Requirements<br>- Spotify Premium<br>- Google Gemini API Key<br>- Google Sheets & Gmail |
| Schedule Trigger (7AM) | Schedule Trigger | Daily trigger | — | Workflow Configuration | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Workflow Configuration | Set | Central config (device name, lat/long, sheetId) | Schedule Trigger (7AM) | Get Spotify Devices; Get Weather (OpenMeteo) | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Get Spotify Devices | Spotify | List available Spotify devices | Workflow Configuration | Find Target Device | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Get Weather (OpenMeteo) | HTTP Request | Fetch current weather by lat/long | Workflow Configuration | Find Target Device | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Find Target Device | Code | Find target Spotify device ID by name | Get Spotify Devices; Get Weather (OpenMeteo) | Device Found? | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Device Found? | IF | Branch on whether device exists | Find Target Device | Get Artist Tracks (true); Send Error Email (false) | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |
| Get Artist Tracks | Spotify | Fetch Led Zeppelin top tracks | Device Found? (true) | Pick Random Track | ## 2. Music Selection<br>Retrieves top tracks from Led Zeppelin, selects one at random, and starts playback. |
| Pick Random Track | Code | Randomly pick a track; prepare playback/email fields | Get Artist Tracks | Start Playback | ## 2. Music Selection<br>Retrieves top tracks from Led Zeppelin, selects one at random, and starts playback. |
| Start Playback | Spotify | Start Spotify playback | Pick Random Track | Message a model | ## 2. Music Selection<br>Retrieves top tracks from Led Zeppelin, selects one at random, and starts playback. |
| Message a model | Google Gemini (LangChain) | Generate DJ greeting text | Start Playback | Log to Google Sheets | ## 3. AI Persona & Log<br>Gemini creates a DJ intro linking weather to the song. Logs history and sends a summary email. |
| Log to Google Sheets | Google Sheets | Append/update alarm history | Message a model | Send DJ Email | ## 3. AI Persona & Log<br>Gemini creates a DJ intro linking weather to the song. Logs history and sends a summary email. |
| Send DJ Email | Gmail | Send morning email with greeting + weather + album art | Log to Google Sheets | — | ## 3. AI Persona & Log<br>Gemini creates a DJ intro linking weather to the song. Logs history and sends a summary email. |
| Send Error Email | Gmail | Notify user device not found | Device Found? (false) | — | ## 1. Context & Setup<br>Initializes the workflow, fetches local weather data, and verifies the Spotify speaker is online. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: *Wake Up with Led Zeppelin: Smart Weather DJ*.

2. **Add Trigger**
   - Add **Schedule Trigger**.
   - Configure it to run daily at your intended time (set timezone appropriately).  
     *If you use “Every 24 hours”, it will run every 24h from activation time rather than “exactly 7AM”.*

3. **Add configuration node**
   - Add a **Set** node named **Workflow Configuration**.
   - Add fields:
     - `targetDeviceName` (string) → e.g. `2F` (must match Spotify device name exactly)
     - `latitude` (string) → e.g. `35.6895`
     - `longitude` (string) → e.g. `139.6917`
     - `sheetId` (string) → your Google Sheet document ID
   - Enable **Include Other Fields** (keep defaults).

4. **Add Spotify device fetch**
   - Add **Spotify** node named **Get Spotify Devices**.
   - Operation: **Get Devices**.
   - Configure **Spotify OAuth2** credentials (Spotify Premium account recommended/required for playback control).

5. **Add weather fetch**
   - Add **HTTP Request** node named **Get Weather (OpenMeteo)**.
   - Method: GET
   - URL (expression):
     - `https://api.open-meteo.com/v1/forecast?latitude={{latitude}}&longitude={{longitude}}&current_weather=true`
   - In n8n expression form, reference the Set node fields:
     - `{{ $('Workflow Configuration').first().json.latitude }}` and `{{ $('Workflow Configuration').first().json.longitude }}`

6. **Add device-selection code**
   - Add **Code** node named **Find Target Device**.
   - Paste logic equivalent to:
     - Read target device name from Workflow Configuration
     - Search Spotify devices list for matching `name`
     - Output `target_device_id`, `device_found`, `target_name`
   - Connect:
     - Workflow Configuration → Get Spotify Devices
     - Workflow Configuration → Get Weather (OpenMeteo)
     - Get Spotify Devices → Find Target Device
     - Get Weather (OpenMeteo) → Find Target Device  
     *Note:* This creates two inbound connections into Find Target Device; consider instead merging with a Merge node if you want deterministic input ordering.

7. **Add IF branch**
   - Add **IF** node named **Device Found?**
   - Condition: `{{ $json.device_found }}` is true
   - Connect: Find Target Device → Device Found?

8. **On success path: fetch Led Zeppelin top tracks**
   - Add **Spotify** node named **Get Artist Tracks**
   - Resource: **Artist**
   - Operation: **Get Top Tracks**
   - Artist ID: `36QJpDe2go2KgaRleHCDTp`
   - Market/Country: `JP` (or your preferred market)
   - Connect: Device Found? (true) → Get Artist Tracks

9. **Pick a random track**
   - Add **Code** node named **Pick Random Track**
   - Logic:
     - Randomly pick one track item
     - Output: `track_uri`, `track_name`, `artist_name`, `album_image`
     - Also include `device_id` from Find Target Device output
   - Connect: Get Artist Tracks → Pick Random Track

10. **Start Spotify playback**
   - Add **Spotify** node named **Start Playback**
   - Operation: **Start / Start Playback**
   - Configure it to use:
     - Device ID: `{{ $json.device_id }}` (from Pick Random Track output)
     - Track/URI: `{{ $json.track_uri }}` (from Pick Random Track output)
   - Connect: Pick Random Track → Start Playback

11. **Generate DJ greeting (Gemini)**
   - Add **Google Gemini** node (LangChain) named **Message a model**
   - Model: `models/gemini-2.5-flash` (or another available model)
   - Message content should reference:
     - Track name and artist name from Pick Random Track
   - Configure **Gemini API** credentials (Google Palm/Gemini key as required by your n8n node).
   - Connect: Start Playback → Message a model

12. **Log to Google Sheets**
   - Add **Google Sheets** node named **Log to Google Sheets**
   - Operation: **Append or Update**
   - Document ID: `{{ $('Workflow Configuration').first().json.sheetId }}`
   - Sheet name: `History`
   - Map columns:
     - `date`: `{{ $now.format('yyyy-MM-dd') }}`
     - `time`: `{{ $now.format('HH:mm') }}`
     - `song`: `{{ $('Pick Random Track').first().json.track_name }}`
     - `artist`: `{{ $('Pick Random Track').first().json.artist_name }}`
     - `weather`: `{{ $('Get Weather (OpenMeteo)').first().json.current_weather.weathercode }}`
     - `temperature`: `{{ $('Get Weather (OpenMeteo)').first().json.current_weather.temperature }}`
   - Create the Google Sheet tab **History** with headers: `date`, `time`, `weather`, `temperature`, `song`, `artist`.
   - Connect: Message a model → Log to Google Sheets

13. **Send the DJ email**
   - Add **Gmail** node named **Send DJ Email**
   - To: your email address
   - Subject: `🎸 Now Playing: {{ $('Pick Random Track').first().json.track_name }}`
   - Body (HTML) including:
     - Gemini output (ensure correct field, e.g. `{{ $('Message a model').first().json.text }}`)
     - Temperature
     - Album image URL in an `<img>` tag
   - Configure **Gmail OAuth2** credentials (scopes to send email).
   - Connect: Log to Google Sheets → Send DJ Email

14. **Failure path: send error email**
   - Add **Gmail** node named **Send Error Email**
   - To: your email address
   - Subject: `🚨 Alarm Failed: Spotify Device Not Found`
   - Body: device not found message
   - Connect: Device Found? (false) → Send Error Email

15. **(Optional) Add sticky notes**
   - Add sticky notes matching the three blocks (Setup, Music, AI) and the main overview note.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Spotify Premium is required for remote playback control. | Applies to Spotify “Start Playback” use case. |
| Ensure your Google Sheet has a “History” tab with headers: `date`, `time`, `weather`, `temperature`, `song`, `artist`. | Required for the Google Sheets mapping. |
| The “Schedule Trigger (7AM)” name may not match its actual schedule unless the trigger is configured for a specific time and timezone. | Verify n8n timezone + cron/daily time setting. |
| Potential reliability issue: Find Target Device has two inbound connections (Spotify devices + weather). Consider using a Merge node to control data flow deterministically. | Prevents mis-reading `$input` if weather arrives first. |
| Open-Meteo is public and does not require authentication, but network failures or schema changes can break expressions. | Consider adding error handling or fallback values. |