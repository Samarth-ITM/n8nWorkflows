Send Daily Weather Forecasts from OpenWeatherMap to Telegram with Smart Formatting

https://n8nworkflows.xyz/workflows/send-daily-weather-forecasts-from-openweathermap-to-telegram-with-smart-formatting-4608


# Send Daily Weather Forecasts from OpenWeatherMap to Telegram with Smart Formatting

### 1. Workflow Overview

This workflow automates the daily retrieval and dissemination of weather forecasts for Strassen, Luxembourg, by integrating OpenWeatherMap’s 5-day forecast API with Telegram messaging. It runs every morning before typical work hours, processes detailed weather data into a human-friendly and emoji-enhanced HTML message, and sends it to a specified Telegram chat or channel.

**Target Use Cases:**  
- Daily weather updates for individuals or groups interested in Strassen’s local weather  
- Automated weather briefing with smart formatting and actionable recommendations  
- Integration of external weather APIs with messaging platforms for timely alerts

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a fixed time.  
- **1.2 Weather Data Retrieval:** Requests forecast data from OpenWeatherMap API.  
- **1.3 Data Processing & Formatting:** Transforms raw forecast data into a readable, emoji-rich message with weather analysis and recommendations.  
- **1.4 Telegram Notification:** Sends the formatted weather forecast message to Telegram.  
- **1.5 Documentation & Setup Notes:** Sticky notes providing configuration details, setup instructions, and best practices.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
Triggers the workflow every day at 7:50 AM to ensure recipients receive weather updates before work hours.

- **Nodes Involved:**  
  - Daily Morning Trigger

- **Node Details:**

| Node Name            | Details                                                                                 |
|----------------------|-----------------------------------------------------------------------------------------|
| Daily Morning Trigger | **Type:** Schedule Trigger node<br>**Role:** Starts the workflow daily at a fixed time<br>**Config:** Set to trigger at 07:50 every day<br>**Input:** None (trigger node)<br>**Output:** Triggers HTTP Request node<br>**Failures:** Rare, but may fail if n8n server time is misconfigured<br>**Version:** 1.2 |

---

#### 1.2 Weather Data Retrieval

- **Overview:**  
Fetches detailed 5-day weather forecasts (3-hour intervals) for Strassen, Luxembourg, from the OpenWeatherMap API, using metric units.

- **Nodes Involved:**  
  - OpenWeather API Request

- **Node Details:**

| Node Name             | Details                                                                                                                         |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------|
| OpenWeather API Request | **Type:** HTTP Request node<br>**Role:** Fetches forecast JSON from OpenWeatherMap<br>**Config:**<br>- URL includes city (`Strassen`) and metric units<br>- API key placeholder to be replaced by user<br>- Timeout: 30 seconds<br>- Retries: 3 attempts with 5-second delay<br>**Input:** Trigger from Schedule node<br>**Output:** JSON forecast data to Processor node<br>**Failures:** Possible API key errors, network issues, rate limiting<br>**Version:** 4.2 |

---

#### 1.3 Data Processing & Formatting

- **Overview:**  
Processes the raw weather data, extracting forecast details for key times of the day. It computes daily temperature ranges, rainfall totals, wind and humidity averages, and generates an emoji-rich, HTML-formatted message with smart recommendations.

- **Nodes Involved:**  
  - Weather Data Processor

- **Node Details:**

| Node Name             | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Weather Data Processor | **Type:** Code node (JavaScript)<br>**Role:** Parses API JSON, calculates daily weather stats, formats message with HTML and emojis<br>**Config Highlights:**<br>- Processes forecast items for today at 09:00, 12:00, 15:00, 18:00, 21:00<br>- Uses helper functions for:<br>  - Weather emojis by condition and cloudiness<br>  - Temperature-based emojis<br>  - Wind descriptions and speed conversion (m/s to km/h)<br>  - Humidity level descriptions<br>  - UV protection advice based on hour<br>- Aggregates daily high/low temps, rain, humidity, wind<br>- Builds structured message with sections:<br>  - Header with weather icon and location<br>  - Daily overview (range, rain, wind, humidity)<br>  - Hourly detailed forecast<br>  - Recommendations (umbrella, dress warmly, sun protection, etc.)<br>  - Footer with last update time<br>**Input:** JSON from HTTP Request node<br>**Output:** JSON object with `message` property for Telegram<br>**Failures:** JS errors if unexpected data structure, timezone issues, missing fields<br>**Version:** 2 |

---

#### 1.4 Telegram Notification

- **Overview:**  
Sends the processed weather forecast message to a configured Telegram chat or channel, using HTML parsing for rich formatting and disabling web preview for cleaner messages.

- **Nodes Involved:**  
  - Send Weather Update

- **Node Details:**

| Node Name           | Details                                                                                                                             |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Send Weather Update  | **Type:** Telegram node<br>**Role:** Delivers the HTML-formatted weather message<br>**Config:**<br>- `text`: Uses expression to access processed message from previous node<br>- `chatId`: Target Telegram chat/channel ID (must be updated)<br>- Additional options:<br>  - `parse_mode`: HTML<br>  - `disable_web_page_preview`: true<br>- Credentials: Telegram bot API token configured<br>**Input:** Output from Code node<br>**Output:** None (final node)<br>**Failures:** Authentication errors, invalid chat ID, network issues<br>**Version:** 1.2 |

---

#### 1.5 Documentation & Setup Notes (Sticky Notes)

- **Overview:**  
Provides user guidance within the workflow canvas for configuration, scheduling, API keys, Telegram setup, and a quick checklist for deployment.

- **Nodes Involved:**  
  - Schedule Setup  
  - API Configuration  
  - Processing Features  
  - Telegram Setup  
  - Setup Checklist  
  - Weather Bot Overview

- **Node Details:**

| Node Name           | Details                                                                                                                        |
|---------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Schedule Setup      | Sticky note explaining daily trigger time, timezone considerations (CET/CEST), and instructions to modify schedule timing.     |
| API Configuration   | Sticky note detailing OpenWeatherMap API key setup, location parameter, API features, and rate limits.                         |
| Processing Features | Sticky note describing the Code node’s advanced features, including smart emojis, detailed metrics, and message formatting.    |
| Telegram Setup      | Sticky note covering Telegram bot creation, obtaining chat ID, and message formatting options.                                 |
| Setup Checklist     | Quick reference checklist for essential steps to configure API key, Telegram bot, test execution, and optional customizations. |
| Weather Bot Overview| Summary and overview of the entire bot’s functionality, features, technical details, and configuration reminders.               |

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                                | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                         |
|-----------------------|---------------------|-----------------------------------------------|--------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Morning Trigger  | Schedule Trigger    | Starts workflow daily at 7:50 AM               | None                     | OpenWeather API Request    | Triggers every day at 7:50 AM to send weather forecast before work hours                                                                                                                                                                           |
| OpenWeather API Request| HTTP Request        | Retrieves 5-day forecast JSON from OpenWeather| Daily Morning Trigger    | Weather Data Processor     | Fetches 5-day forecast data with 3-hour intervals for Strassen, Luxembourg                                                                                                                                                                         |
| Weather Data Processor | Code                | Processes and formats weather data into message| OpenWeather API Request  | Send Weather Update        | Enhanced processor with comprehensive weather analysis, recommendations, and formatting                                                                                                                                                            |
| Send Weather Update    | Telegram            | Sends formatted weather forecast to Telegram  | Weather Data Processor   | None                      | Sends formatted weather forecast to Telegram with HTML formatting                                                                                                                                                                                  |
| Schedule Setup         | Sticky Note         | Explains scheduling configuration and tips    | None                     | None                      | ### ⏰ Schedule Configuration<br>Current Settings: 7:50 AM daily, timezone notes, how to change trigger time                                                                                                                                       |
| API Configuration      | Sticky Note         | Provides OpenWeatherMap API setup guidance     | None                     | None                      | ### 🌐 OpenWeatherMap API<br>API key setup, location param, API features, rate limits                                                                                                                                                               |
| Processing Features    | Sticky Note         | Details features of the Code node               | None                     | None                      | ### 💻 Code Node Features<br>Smart emojis, detailed weather metrics, formatting details                                                                                                                                                            |
| Telegram Setup         | Sticky Note         | Guides Telegram bot creation and message setup | None                     | None                      | ### 📱 Telegram Setup<br>Bot creation, chat ID retrieval, message format settings                                                                                                                                                                   |
| Setup Checklist        | Sticky Note         | Quick deployment checklist                      | None                     | None                      | ### 🚀 Quick Setup Checklist<br>API key, Telegram bot, test, optional customizations, monitoring                                                                                                                                                    |
| Weather Bot Overview   | Sticky Note         | High-level bot summary and instructions        | None                     | None                      | ## 🌤️ Daily Weather Forecast Bot<br>What it does, features, technical details, configuration instructions                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Settings: Trigger at 07:50 every day (set `triggerAtHour` to 7, `triggerAtMinute` to 50)  
   - Purpose: Start workflow daily before work hours  

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Connect input from Schedule Trigger node  
   - URL: `https://api.openweathermap.org/data/2.5/forecast?q=Strassen&appid=YOUR_API_KEY&units=metric`  
   - Replace `YOUR_API_KEY` with your actual OpenWeatherMap API key  
   - Set timeout to 30000 ms (30 seconds)  
   - Enable retry on failure with 3 tries and 5000 ms wait between tries  

3. **Create Code Node (JavaScript)**  
   - Type: Code (JavaScript)  
   - Connect input from HTTP Request node  
   - Paste the provided JavaScript code that:  
     - Extracts today’s forecast at 09:00, 12:00, 15:00, 18:00, 21:00  
     - Calculates daily high/low temps, total rain, average humidity and wind speed  
     - Uses helper functions to assign weather, temperature, wind, humidity emojis and descriptions  
     - Constructs an HTML message with bold and italic formatting and emojis  
     - Adds smart recommendations based on weather conditions  
     - Appends a footer with data source and update time in `Europe/Luxembourg` timezone  
   - Ensure the output is a single JSON with `message` key containing the formatted string  

4. **Create Telegram Node**  
   - Type: Telegram  
   - Connect input from Code node  
   - Set `text` parameter to expression: `={{$json["message"]}}`  
   - Provide `chatId` of the target Telegram group or channel (e.g., `-1002521174755`)  
   - In Additional Fields:  
     - Set `parse_mode` to `HTML` to enable rich text formatting  
     - Set `disable_web_page_preview` to `true` for clean message appearance  
   - Select or create Telegram credentials with your bot’s API token (obtained from @BotFather)  

5. **Optional: Add Sticky Notes for Documentation**  
   - Create Sticky Note nodes anywhere on the canvas to document:  
     - Schedule setup and timezone notes  
     - OpenWeatherMap API key and endpoint details  
     - Code node feature explanations  
     - Telegram bot creation and chat ID retrieval instructions  
     - Overall workflow overview and quick setup checklist  

6. **Testing and Activation**  
   - Manually execute the workflow to verify successful API fetch and message delivery to Telegram  
   - Correct any errors related to API key, chat ID, or network connectivity  
   - Once verified, activate the workflow for automatic daily runs  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| OpenWeatherMap API key required. Sign up at https://openweathermap.org/api to obtain a free API key. Replace in HTTP Request node URL.                                                                                                                                                         | OpenWeatherMap API                                                                                |
| Telegram bot creation via [@BotFather](https://t.me/BotFather). Use `/newbot` command to create a bot and receive API token.                                                                                                                                                                    | Telegram Bot Setup                                                                               |
| To find chat ID for sending messages, send a message to the bot and check updates at `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`. Extract the chat ID from the response.                                                                                                          | Telegram Chat ID retrieval                                                                       |
| Timezone awareness critical: Workflow assumes `Europe/Luxembourg` timezone for all date/time formatting and UV advice. Adjust if deploying elsewhere.                                                                                                                                           | Timezone configuration note                                                                     |
| API rate limits: Free tier allows 60 calls/minute and 1000 calls/day. Workflow uses 1 call per day, well within limits.                                                                                                                                                                         | OpenWeatherMap API rate limits                                                                  |
| Message formatting uses HTML with emojis for readability in Telegram. Web page preview disabled to keep appearance clean.                                                                                                                                                                       | Telegram message formatting                                                                     |
| The code node includes error handling primarily via retries in HTTP Request node. Additional try/catch could be added in Code node for robustness.                                                                                                                                              | Error handling strategy                                                                         |
| For customization: Change city in HTTP Request URL, modify forecast hours in code, or adjust schedule trigger time as needed.                                                                                                                                                                  | Customization guidance                                                                           |
| Workflow tested on n8n version compatible with HTTP Request 4.2 and Telegram node 1.2, Code node version 2.                                                                                                                                                                                    | Version compatibility                                                                           |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.