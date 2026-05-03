Automated Indonesian Weather Alerts with BMKG Data & Telegram Notifications

https://n8nworkflows.xyz/workflows/automated-indonesian-weather-alerts-with-bmkg-data---telegram-notifications-9675


# Automated Indonesian Weather Alerts with BMKG Data & Telegram Notifications

---
### 1. Workflow Overview

This workflow automates weather monitoring for Indonesian regions using BMKG (Badan Meteorologi, Klimatologi, dan Geofisika) public API data, formats weather reports, and sends alerts to a Telegram chat. It is designed for recurring weather updates and notifications, with error handling to report issues via Telegram.

**Use Cases:**  
- Automated weather monitoring and alerting for Indonesian locations.  
- Delivering summarized weather forecasts and warnings via Telegram messaging.  
- Providing actionable weather insights such as temperature extremes and rain probabilities.

**Logical Blocks:**  
- **1.1 Trigger & Data Retrieval**: Scheduled and manual triggers start the workflow, fetching BMKG weather data via HTTP request.  
- **1.2 Data Processing**: Parses and processes raw BMKG data into structured weather information including current conditions, 24-hour forecast, averages, dominant weather, and warnings.  
- **1.3 Conditional Flow & Formatting**: Checks for successful data retrieval, formats the weather data into a Telegram-friendly markdown message, or prepares an error message if data is missing or invalid.  
- **1.4 Notifications via Telegram**: Sends formatted weather reports or error alerts to a configured Telegram chat.  
- **1.5 Error Handling**: Logs errors and sends alerts when weather data retrieval or processing fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Retrieval

**Overview:**  
This block initiates the workflow execution either on a fixed schedule (every 6 hours) or manually, then requests BMKG weather forecast data via an HTTP GET request.

**Nodes Involved:**  
- Schedule Trigger  
- Manual Test  
- Get BMKG Weather Data

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically starts workflow every 6 hours.  
  - Configuration: Interval set to 6 hours (modifiable).  
  - Inputs: None  
  - Outputs: Connects to "Get BMKG Weather Data" node.  
  - Edge Cases: Workflow inactive or scheduling misconfiguration could prevent execution.

- **Manual Test**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing purposes.  
  - Configuration: No parameters.  
  - Inputs: None  
  - Outputs: Connects to "Get BMKG Weather Data".  
  - Edge Cases: Manual trigger requires user intervention.

- **Get BMKG Weather Data**  
  - Type: HTTP Request  
  - Role: Fetches raw weather data JSON from BMKG public API endpoint.  
  - Configuration: URL set to "https://api.bmkg.go.id/publik/prakiraan-cuaca". No query parameters set by default but optional adm4 region code can be added.  
  - Inputs: From trigger nodes.  
  - Outputs: Passes raw JSON to "Process Weather Data".  
  - Edge Cases: Network errors, API downtime, invalid responses, rate limiting.  
  - Version: HTTP Request node v4.1

#### 2.2 Data Processing

**Overview:**  
Processes raw BMKG data to extract current weather, a 24-hour forecast, calculates average temperature and humidity, identifies dominant weather condition, and generates warnings based on thresholds.

**Nodes Involved:**  
- Process Weather Data

**Node Details:**

- **Process Weather Data**  
  - Type: Code (JavaScript)  
  - Role: Parses and structures BMKG JSON data into a comprehensive object including location, current weather, forecast array, statistical summary, and warnings.  
  - Configuration: Custom JS code performing validation, extraction, aggregation, and conditional warning creation.  
  - Inputs: Raw JSON from "Get BMKG Weather Data".  
  - Outputs: JSON object with processed weather data to "Check Success".  
  - Key Variables: `weatherData`, `processedData`, `warnings`, averages calculations, dominant condition logic.  
  - Edge Cases: Missing or malformed BMKG data, empty arrays, unexpected data formats. Returns error JSON if no valid weather data.  
  - Version: Code node v2

#### 2.3 Conditional Flow & Formatting

**Overview:**  
Checks if the weather data processing was successful. If yes, formats a detailed Telegram message; if not, generates an error message for alerting.

**Nodes Involved:**  
- Check Success  
- Format Telegram Message  
- Error Handler

**Node Details:**

- **Check Success**  
  - Type: If  
  - Role: Branch workflow based on `success` boolean in processed data JSON.  
  - Configuration: Condition: `$json.success === true`  
  - Inputs: From "Process Weather Data"  
  - Outputs: True branch to "Format Telegram Message", False branch to "Error Handler"  
  - Edge Cases: Unexpected `success` values or missing keys.

- **Format Telegram Message**  
  - Type: Code (JavaScript)  
  - Role: Formats the weather report into a markdown Telegram message string including location, current weather, 24-hour summary, warnings, and next 6 hours forecast.  
  - Configuration: Custom JS generates message text, uses localization for time formatting with timezone consideration. Includes fallback for error messages.  
  - Inputs: From "Check Success" (true branch).  
  - Outputs: JSON with `telegram_message`, `parse_mode`, and embedded `weather_data` for potential future use.  
  - Edge Cases: Missing data fields, formatting errors.  
  - Version: Code node v2

- **Error Handler**  
  - Type: Code (JavaScript)  
  - Role: Logs the error details and creates a Telegram markdown message describing the failure with timestamp.  
  - Configuration: Custom JS extracting error info from input JSON and formatting alert message.  
  - Inputs: From "Check Success" (false branch).  
  - Outputs: JSON with `telegram_message`, `parse_mode`, and `error_logged` flag.  
  - Edge Cases: Missing error details in input JSON.  
  - Version: Code node v2

#### 2.4 Notifications via Telegram

**Overview:**  
Sends the formatted weather report or error alert messages to a specified Telegram chat using Telegram Bot API credentials.

**Nodes Involved:**  
- Send Weather Report  
- Send Error Alert

**Node Details:**

- **Send Weather Report**  
  - Type: Telegram  
  - Role: Sends weather report messages to Telegram chat.  
  - Configuration: Uses message text from `telegram_message` output of previous node; chat ID is a parameter placeholder `{{TELEGRAM_CHAT_ID}}`; parse mode set to Markdown for formatting.  
  - Inputs: From "Format Telegram Message".  
  - Outputs: None (final node).  
  - Credentials: Telegram API credential selected.  
  - Edge Cases: Invalid chat ID, Telegram API errors, message size limits.  
  - Version: Telegram node v1.1

- **Send Error Alert**  
  - Type: Telegram  
  - Role: Sends error alert messages to Telegram chat similarly to Send Weather Report.  
  - Configuration: Same as above, message text from "Error Handler" node.  
  - Inputs: From "Error Handler".  
  - Outputs: None (final node).  
  - Credentials: Same Telegram API credential.  
  - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                      | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                                      |
|-------------------------|--------------------|------------------------------------|---------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Scheduled workflow start            | —                         | Get BMKG Weather Data        | Default: Every 6 hours; can be modified. See Sticky Note3 for scheduling details.                               |
| Manual Test             | Manual Trigger      | Manual workflow start               | —                         | Get BMKG Weather Data        | Enables manual testing.                                                                                          |
| Get BMKG Weather Data   | HTTP Request       | Retrieve BMKG weather JSON          | Schedule Trigger, Manual Test | Process Weather Data          | Optional: add 'adm4' query param for region code. See Sticky Note4.                                             |
| Process Weather Data    | Code               | Parse and aggregate weather data   | Get BMKG Weather Data       | Check Success                | Returns success flag & detailed weather object or error info.                                                  |
| Check Success           | If                 | Branch on data processing success  | Process Weather Data        | Format Telegram Message (true), Error Handler (false) |                                                                                                                 |
| Format Telegram Message | Code               | Generate Telegram message text     | Check Success (true)        | Send Weather Report          | Formats markdown message with weather summary and warnings.                                                    |
| Error Handler           | Code               | Generate error alert message        | Check Success (false)       | Send Error Alert             | Logs error and formats error message for Telegram.                                                             |
| Send Weather Report     | Telegram            | Send weather report via Telegram    | Format Telegram Message     | —                           | Replace {{TELEGRAM_CHAT_ID}} with actual Chat ID. See Sticky Note4. Credential required.                        |
| Send Error Alert        | Telegram            | Send error alert via Telegram       | Error Handler              | —                           | Same Chat ID and credential as above.                                                                           |
| Sticky Note             | Sticky Note         | Learning resources                  | —                         | —                           | Contains links to n8n docs, BMKG API docs, Telegram Bot API info.                                               |
| Sticky Note1            | Sticky Note         | Workflow title display              | —                         | —                           | "BMKG WEATHER MONITORING & TELEGRAM ALERT SYSTEM" title.                                                      |
| Sticky Note2            | Sticky Note         | Telegram bot creation instructions  | —                         | —                           | Explains how to create bot and get Chat ID for Telegram.                                                       |
| Sticky Note3            | Sticky Note         | Instructions for enabling and scheduling | —                         | —                           | Details on activating workflow and schedule trigger settings.                                                  |
| Sticky Note4            | Sticky Note         | Telegram chat ID and credential configuration | —                         | —                           | Guidance on replacing Chat ID placeholders and linking Telegram credentials.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours (default). Optionally adjust timing as needed.  
   - No credentials needed.

2. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters.

3. **Connect Schedule Trigger and Manual Trigger outputs to the next node.**

4. **Create HTTP Request node "Get BMKG Weather Data"**  
   - URL: `https://api.bmkg.go.id/publik/prakiraan-cuaca`  
   - Method: GET (default)  
   - No authentication required.  
   - Optional: Add query parameter `adm4` with BMKG region code if targeting specific location.  
   - Connect inputs from both triggers.

5. **Create Code node "Process Weather Data"**  
   - Paste the provided JavaScript code to parse and process BMKG JSON.  
   - Inputs: connect from "Get BMKG Weather Data".  
   - Outputs: processed weather data JSON with success flag, location info, current weather, forecast, summary, and warnings.

6. **Create If node "Check Success"**  
   - Condition: Boolean check where `$json.success` equals `true`.  
   - Inputs: connect from "Process Weather Data".  
   - True branch connects to "Format Telegram Message".  
   - False branch connects to "Error Handler".

7. **Create Code node "Format Telegram Message"**  
   - Paste the provided JavaScript code for formatting Telegram markdown message.  
   - Inputs: connect from "Check Success" true output.  
   - Outputs: JSON with `telegram_message` and `parse_mode`.

8. **Create Code node "Error Handler"**  
   - Paste provided JavaScript code logging error and preparing error message.  
   - Inputs: connect from "Check Success" false output.  
   - Outputs: JSON with `telegram_message` and `parse_mode`.

9. **Create Telegram node "Send Weather Report"**  
   - Text: `={{ $json.telegram_message }}`  
   - Chat ID: Replace placeholder `{{TELEGRAM_CHAT_ID}}` with actual Telegram chat ID.  
   - Additional Fields: set `parse_mode` to `Markdown`.  
   - Credentials: Select Telegram API credential with bot token.  
   - Inputs: connect from "Format Telegram Message".

10. **Create Telegram node "Send Error Alert"**  
    - Same settings as above for text, chat ID, parse mode, and credentials.  
    - Inputs: connect from "Error Handler".

11. **Activate the workflow**  
    - Toggle workflow status to active for scheduled runs.  
    - Manual trigger can be used for immediate testing.

12. **Additional setup notes:**  
    - Obtain Telegram bot token by creating a bot with @BotFather on Telegram.  
    - Retrieve chat ID by sending message to bot and querying Telegram API getUpdates.  
    - Store credentials securely in n8n for Telegram node use.  
    - Optionally customize schedule trigger interval or BMKG region code.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                  | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| 🎓 **n8n Documentation:** Official docs, node references, and workflow examples provide comprehensive guidance on n8n usage and integration.                                                    | https://docs.n8n.io, https://n8n.io/workflows/       |
| **BMKG API Information:** Official API endpoint documentation, region codes, and JSON data structure details to understand data sources and content.                                         | https://api.bmkg.go.id                              |
| **Telegram Bot API:** Instructions on creating bots, retrieving tokens, chat IDs, and Telegram API references for bot integration and message sending.                                        | https://core.telegram.org/bots                      |
| **Telegram Bot Creation Instructions:** Use @BotFather to create bot and obtain tokens; get Chat ID by sending message to bot and querying `getUpdates` endpoint.                               | See Sticky Note2 in workflow                         |
| **Workflow Activation & Scheduling:** Toggle workflow active switch to enable automatic runs; schedule trigger defaults to every 6 hours but is adjustable.                                   | See Sticky Note3 in workflow                         |
| **Configuration Tips:** Replace `{{TELEGRAM_CHAT_ID}}` placeholders with actual chat IDs in Telegram nodes; link Telegram credentials properly to authorize API calls.                           | See Sticky Note4 in workflow                         |

---

**Disclaimer:**  
The content above is derived solely from an automated n8n workflow. The workflow respects all current content policies and contains no illegal or protected elements. All data processed are public and lawful.

---