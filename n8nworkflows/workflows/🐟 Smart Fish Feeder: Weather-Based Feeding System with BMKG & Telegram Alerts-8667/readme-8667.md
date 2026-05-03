🐟 Smart Fish Feeder: Weather-Based Feeding System with BMKG & Telegram Alerts

https://n8nworkflows.xyz/workflows/---smart-fish-feeder--weather-based-feeding-system-with-bmkg---telegram-alerts-8667


# 🐟 Smart Fish Feeder: Weather-Based Feeding System with BMKG & Telegram Alerts

### 1. Workflow Overview

This workflow, titled **🐟 Smart Fish Feeder: BMKG Weather-Based Automated Feeding System**, automates fish feeding based on weather forecasts from the Indonesian Meteorological, Climatological, and Geophysical Agency (BMKG). It leverages weather data to adjust feeding quantities, aiming to optimize fish health and feeding efficiency according to rain probability. Notifications and control commands are sent via Telegram and an ESP8266 microcontroller, respectively.

The workflow is structured into the following logical blocks:

- **1.1 Input Scheduling:** Triggers the workflow automatically twice daily (05:30 and 16:30 WIB).
- **1.2 Configuration Setup:** Defines static and dynamic parameters including location, BMKG API details, Telegram bot credentials, and ESP8266 webhook URL.
- **1.3 BMKG Weather Data Handling:** Builds the API request URL, fetches weather forecast, and parses the data to evaluate rain probabilities.
- **1.4 Decision Logic:** Compares rain probability against a threshold to decide on feed reduction or normal feeding.
- **1.5 Device Control:** Sends feeding commands to the ESP8266 fish feeder hardware.
- **1.6 Reporting & Logging:** Sends a detailed feeding report via Telegram and logs the activity for monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Scheduling

- **Overview:** Initiates the workflow twice daily at fixed times to provide timely feeding decisions.
- **Nodes Involved:** `Cron: 05:30 & 16:30 WIB`

##### Node: Cron: 05:30 & 16:30 WIB
- **Type & Role:** Schedule Trigger; triggers workflow execution.
- **Configuration:** Schedule set to trigger twice daily at 05:30 and 16:30 WIB (Asia/Jakarta timezone implied though not explicitly set in node parameters).
- **Expressions:** None.
- **Connections:** Outputs to `Config` node.
- **Failure Modes:** Cron misconfiguration or timezone mismatch may cause incorrect triggering times.
- **Version:** 1.2 (Schedule Trigger node recent version).

---

#### 2.2 Configuration Setup

- **Overview:** Prepares essential configuration variables used throughout the workflow, including API keys, location info, thresholds, and endpoint URLs.
- **Nodes Involved:** `Config`

##### Node: Config
- **Type & Role:** Set node; assigns workflow-wide parameters.
- **Configuration Highlights:**
  - `locationName`: "Main Pond"
  - Geographical coordinates (`lat`, `lon`): "-6.2000", "106.8166"
  - BMKG API URL template with placeholder for `adm4` area code.
  - API keys and tokens placeholders for BMKG, Telegram, and ESP8266 webhook.
  - `adm4`: BMKG administrative code "31.71.03.1001"
  - Thresholds: `thresholdProb` (60%), `reducePercent` (-20%)
- **Expressions:** Uses placeholders for sensitive credentials.
- **Connections:** Outputs to `Build Forecast URL`.
- **Failures:** Missing or invalid credentials or incorrect area code can cause API failures.
- **Version:** 3.4 (Set node).

---

#### 2.3 BMKG Weather Data Handling

- **Overview:** Builds a request URL for BMKG, fetches weather forecast, and parses the forecast to calculate rain probabilities and feeding adjustments.
- **Nodes Involved:** 
  - `Build Forecast URL`
  - `HTTP: Forecast BMKG`
  - `Parse & Score Weather (6-12h)`

##### Node: Build Forecast URL
- **Type & Role:** Code node; dynamically builds the BMKG API URL.
- **Configuration:**
  - Reads `adm4` and URL template from `Config`.
  - Replaces `{{ADM4}}` placeholder with actual value.
  - Returns constructed URL and metadata (timestamp, location).
  - Includes error handling with fallback URL.
- **Key Expressions:** Uses `$input.item(0).json` and template string replacement.
- **Connections:** Outputs URL to `HTTP: Forecast BMKG`.
- **Failures:** Missing `adm4` or URL template leads to fallback URL and error message in output.
- **Version:** 2 (Code node).

##### Node: HTTP: Forecast BMKG
- **Type & Role:** HTTP Request; fetches real-time weather forecast from BMKG API.
- **Configuration:**
  - Uses URL from previous node.
  - Timeout set to 30 seconds.
  - Configured to never error on HTTP failure (to allow graceful handling downstream).
- **Connections:** Outputs JSON response to `Parse & Score Weather (6-12h)`.
- **Failures:** Network timeout, API downtime, or malformed response.
- **Version:** 4.2 (HTTP Request node).

##### Node: Parse & Score Weather (6-12h)
- **Type & Role:** Code node; parses BMKG forecast data, calculates rain probabilities, and determines feed reduction ratio.
- **Configuration:**
  - Supports multiple BMKG response formats.
  - Processes weather data for next 12 hours (four 3-hour periods).
  - Calculates:
    - Maximum rain probability over first 6 hours.
    - Average rain probability over 12 hours.
    - Final rain probability = max of average and max 6h.
  - Compares final rain probability against threshold from `Config`.
  - Determines feed reduction percentage.
  - Builds detailed weather conditions array for reporting.
  - Provides error handling with default zero values.
- **Expressions:** Uses `$`, `$json`, and references `Config` node.
- **Connections:** Outputs metrics to `IF: High Rain Probability`.
- **Failures:** Unexpected API response structure, missing fields, or parsing errors.
- **Version:** 2 (Code node).

---

#### 2.4 Decision Logic

- **Overview:** Evaluates rain probability to decide whether to reduce feeding or maintain normal feeding.
- **Nodes Involved:** 
  - `IF: High Rain Probability`
  - `Set: Reduce Feed 20%`
  - `Set: Normal Feed 0%`
  - `Merge Branches`

##### Node: IF: High Rain Probability
- **Type & Role:** If node; compares rain probability to threshold.
- **Configuration:** Condition `rain_prob >= thresholdProb`.
- **Expressions:** Uses `$json.rain_prob` and `Config.thresholdProb`.
- **Connections:** 
  - True branch → `Set: Reduce Feed 20%`
  - False branch → `Set: Normal Feed 0%`
- **Failures:** Expression evaluation errors if inputs missing.
- **Version:** 2.

##### Node: Set: Reduce Feed 20%
- **Type & Role:** Set node; sets parameters for reduced feeding.
- **Configuration:**
  - Note message with warning and instructions.
  - Feed ratio from parsed weather node (`feed_ratio`).
  - Action type: "reduce_feed".
  - ESP8266 command: "FEED_REDUCE_20".
- **Connections:** Outputs to `Merge Branches` (branch 0).
- **Failures:** None expected.
- **Version:** 3.4.

##### Node: Set: Normal Feed 0%
- **Type & Role:** Set node; sets parameters for normal feeding.
- **Configuration:**
  - Note message indicating safe weather.
  - Feed ratio 0 (no reduction).
  - Action type: "normal_feed".
  - ESP8266 command: "FEED_NORMAL".
- **Connections:** Outputs to `Merge Branches` (branch 1).
- **Failures:** None expected.
- **Version:** 3.4.

##### Node: Merge Branches
- **Type & Role:** Merge node; merges two branches from IF node.
- **Configuration:** Mode "chooseBranch" to continue with whichever branch is active.
- **Connections:** Outputs merged data to `ESP8266 Fish Feeder Control`.
- **Failures:** Branch mismatch or missing inputs can cause errors.
- **Version:** 2.1.

---

#### 2.5 Device Control

- **Overview:** Sends feeding commands with parameters to the ESP8266 fish feeder hardware via webhook.
- **Nodes Involved:** `ESP8266 Fish Feeder Control`

##### Node: ESP8266 Fish Feeder Control
- **Type & Role:** HTTP Request node; posts control commands to ESP8266 webhook URL.
- **Configuration:**
  - URL from `Config.esp8266WebhookUrl`.
  - HTTP POST with JSON body including:
    - `command` (e.g., "FEED_REDUCE_20", "FEED_NORMAL")
    - `feed_ratio`, `rain_prob`, `timestamp`, `location`.
  - Headers: Content-Type application/json, User-Agent "n8n-bmkg-feeder/1.0"
  - Timeout 10 seconds.
- **Connections:** Outputs to `Telegram: Send Report`.
- **Failures:** Network errors, webhook unreachable, timeout, or ESP8266 firmware errors.
- **Version:** 4.2.

---

#### 2.6 Reporting & Logging

- **Overview:** Sends feeding status report via Telegram and logs activity for monitoring.
- **Nodes Involved:** 
  - `Telegram: Send Report`
  - `Activity Logger`

##### Node: Telegram: Send Report
- **Type & Role:** Telegram node; delivers formatted feeding report message.
- **Configuration:**
  - Markdown-formatted message summarizing:
    - Location and feeding schedule time.
    - BMKG rain probabilities (12h, 6h max, 12h avg).
    - Feed ratio decision and ESP8266 command status.
    - Additional notes from feeding decision.
    - Next scheduled feeding time calculation.
  - Chat ID from `Config`.
  - Uses Telegram API credentials.
- **Connections:** Outputs to `Activity Logger`.
- **Failures:** Telegram API authorization errors, invalid chat ID, network failures.
- **Version:** 1.2.

##### Node: Activity Logger
- **Type & Role:** Code node; compiles detailed log entry for each execution.
- **Configuration:**
  - Collects data from weather parsing, feeding decision, ESP8266 response.
  - Logs timestamp, location, rain probabilities, feed ratio, action type, ESP8266 status, weather summary, BMKG analysis time.
- **Connections:** End node, no outputs.
- **Failures:** None expected unless upstream data missing.
- **Version:** 2.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                 | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                          |
|-------------------------------|---------------------|--------------------------------|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Cron: 05:30 & 16:30 WIB       | Schedule Trigger    | Trigger workflow twice daily   |                             | Config                        | See installation and setup instructions in sticky note for workflow setup and troubleshooting.                       |
| Config                        | Set                 | Define static/dynamic configs  | Cron: 05:30 & 16:30 WIB     | Build Forecast URL            | See installation and setup instructions in sticky note for API keys, Telegram, and ESP8266 configurations.            |
| Build Forecast URL            | Code                 | Build BMKG API URL             | Config                      | HTTP: Forecast BMKG           |                                                                                                                      |
| HTTP: Forecast BMKG           | HTTP Request         | Fetch BMKG weather data        | Build Forecast URL          | Parse & Score Weather (6-12h) |                                                                                                                      |
| Parse & Score Weather (6-12h) | Code                 | Parse and score rain prob      | HTTP: Forecast BMKG         | IF: High Rain Probability     |                                                                                                                      |
| IF: High Rain Probability     | If                   | Decide on feed reduction       | Parse & Score Weather (6-12h) | Set: Reduce Feed 20% / Set: Normal Feed 0% |                                                                                                                      |
| Set: Reduce Feed 20%          | Set                  | Set parameters for reduced feed| IF: High Rain Probability (true) | Merge Branches              |                                                                                                                      |
| Set: Normal Feed 0%           | Set                  | Set parameters for normal feed | IF: High Rain Probability (false) | Merge Branches              |                                                                                                                      |
| Merge Branches               | Merge                 | Merge decision branches        | Set: Reduce Feed 20%, Set: Normal Feed 0% | ESP8266 Fish Feeder Control |                                                                                                                      |
| ESP8266 Fish Feeder Control   | HTTP Request         | Send feeding command to ESP8266| Merge Branches              | Telegram: Send Report          |                                                                                                                      |
| Telegram: Send Report         | Telegram              | Send report to Telegram chat   | ESP8266 Fish Feeder Control | Activity Logger               |                                                                                                                      |
| Activity Logger              | Code                  | Log feeding activity           | Telegram: Send Report        |                               |                                                                                                                      |
| Sticky Note                  | Sticky Note           | Provides detailed installation, configuration, and troubleshooting guide |                              |                               | Full installation and setup instructions, troubleshooting, Arduino code snippet, and links for ESP8266 firmware.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node**
   - Type: Schedule Trigger
   - Name: "Cron: 05:30 & 16:30 WIB"
   - Set schedule to trigger twice daily at 05:30 and 16:30.
   - Timezone: Asia/Jakarta (set in n8n environment or node if supported).
   - Connect output to next node.

2. **Create a Set Node for Configuration**
   - Type: Set
   - Name: "Config"
   - Assign values:
     - `locationName`: "Main Pond"
     - `lat`: "-6.2000"
     - `lon`: "106.8166"
     - `bmkgUrlTemplate`: "https://api.bmkg.go.id/publik/prakiraan-weather?adm4={{ADM4}}"
     - `bmkgApiKey`: (leave placeholder or actual key if available)
     - `telegramBotToken`: (Telegram bot token placeholder)
     - `telegramChatId`: (Telegram chat ID placeholder)
     - `esp8266WebhookUrl`: (ESP8266 webhook URL placeholder, e.g., "http://192.168.1.100/webhook")
     - `adm4`: "31.71.03.1001"
     - `thresholdProb`: "60"
     - `reducePercent`: "-20"
   - Connect output to next node.

3. **Create a Code Node to Build Forecast URL**
   - Type: Code
   - Name: "Build Forecast URL"
   - Paste JavaScript code to:
     - Replace `{{ADM4}}` in URL template with actual `adm4`.
     - Return the constructed URL, timestamp, and location.
     - Include error handling fallback.
   - Connect output to HTTP Request node.

4. **Create HTTP Request Node to Fetch BMKG Forecast**
   - Type: HTTP Request
   - Name: "HTTP: Forecast BMKG"
   - Method: GET
   - URL: Use expression to get URL from previous node output (`{{$json.url}}`)
   - Timeout: 30000 ms
   - Response format: JSON
   - Set option to never error on HTTP errors (important for graceful failure).
   - Connect output to next node.

5. **Create Code Node to Parse & Score Weather**
   - Type: Code
   - Name: "Parse & Score Weather (6-12h)"
   - Paste JavaScript code that:
     - Parses BMKG response.
     - Calculates rain probabilities max 6h, average 12h, and final rain probability.
     - Compares with threshold to determine feed ratio.
     - Prepares detailed weather conditions array.
     - Handles errors gracefully.
   - Connect output to IF node.

6. **Create IF Node for Rain Probability Decision**
   - Type: If
   - Name: "IF: High Rain Probability"
   - Condition: `$json.rain_prob >= Config.thresholdProb` (use expression)
   - True output to "Set: Reduce Feed 20%"
   - False output to "Set: Normal Feed 0%"

7. **Create Set Node for Feed Reduction**
   - Type: Set
   - Name: "Set: Reduce Feed 20%"
   - Assign:
     - `note`: Warning about high rain probability and recommended actions.
     - `feed_ratio`: feed ratio from previous code node.
     - `action_type`: "reduce_feed"
     - `esp8266_command`: "FEED_REDUCE_20"
   - Connect output to Merge node branch 0.

8. **Create Set Node for Normal Feed**
   - Type: Set
   - Name: "Set: Normal Feed 0%"
   - Assign:
     - `note`: Info about safe weather and normal feeding.
     - `feed_ratio`: 0
     - `action_type`: "normal_feed"
     - `esp8266_command`: "FEED_NORMAL"
   - Connect output to Merge node branch 1.

9. **Create Merge Node**
   - Type: Merge
   - Name: "Merge Branches"
   - Mode: Choose Branch
   - Inputs: From both Set nodes.
   - Connect output to ESP8266 HTTP Request node.

10. **Create HTTP Request Node for ESP8266 Control**
    - Type: HTTP Request
    - Name: "ESP8266 Fish Feeder Control"
    - Method: POST
    - URL: Use expression from `Config.esp8266WebhookUrl`.
    - Body (JSON) parameters:
      - `command`: from previous Set node (`$json.esp8266_command`).
      - `feed_ratio`: from previous Set node (`$json.feed_ratio`).
      - `rain_prob`: from Parse & Score Weather node.
      - `timestamp`: current ISO timestamp.
      - `location`: from Config.
    - Headers:
      - Content-Type: application/json
      - User-Agent: n8n-bmkg-feeder/1.0
    - Timeout: 10000 ms.
    - Connect output to Telegram node.

11. **Create Telegram Node to Send Report**
    - Type: Telegram
    - Name: "Telegram: Send Report"
    - Use Telegram API credentials (created by adding Telegram Bot token in n8n credentials).
    - Chat ID: from Config node.
    - Message text: Markdown formatted message including weather data, feed ratio, ESP8266 status, note, and next feeding time.
    - Connect output to Activity Logger.

12. **Create Code Node for Activity Logging**
    - Type: Code
    - Name: "Activity Logger"
    - Code to create a JSON log entry with timestamp, location, rain probabilities, feed ratio, action type, ESP8266 status, weather summary, and BMKG analysis time.
    - No further nodes connected.

13. **Add a Sticky Note**
    - Paste the full installation, configuration, and troubleshooting guide content.
    - Position anywhere visible for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Full installation and setup instructions for n8n, Telegram bot, BMKG API, ESP8266 webhook, and schedule. | Included as a large sticky note in the workflow for easy reference during deployment and troubleshooting.          |
| Arduino example code for ESP8266 webhook handling feeding commands.                                      | https://github.com/TegarDev9/fish-feed-esp8266.git (linked in sticky note)                                         |
| Monitoring and logs instructions for n8n execution history and Telegram notifications.                    | Detailed in sticky note for operational monitoring.                                                                |
| Troubleshooting tips for common errors including node not found, Telegram authentication, ESP8266 response, and BMKG API timeouts. | Included in sticky note to aid users during setup and runtime issues.                                              |
| Powered by BMKG API, ESP8266 hardware, and n8n automation platform.                                       | Branding note in Telegram message template to credit components of the system.                                     |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.