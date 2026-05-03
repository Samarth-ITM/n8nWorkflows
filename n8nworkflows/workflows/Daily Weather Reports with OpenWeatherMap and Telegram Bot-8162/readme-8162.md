Daily Weather Reports with OpenWeatherMap and Telegram Bot

https://n8nworkflows.xyz/workflows/daily-weather-reports-with-openweathermap-and-telegram-bot-8162


# Daily Weather Reports with OpenWeatherMap and Telegram Bot

### 1. Workflow Overview

This workflow automates the process of sending daily weather reports to a Telegram chat using data from OpenWeatherMap. It is designed for users who want scheduled, formatted weather updates delivered directly via Telegram. The workflow performs the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow on a defined daily schedule.
- **1.2 Weather Data Retrieval:** Fetches current weather data from OpenWeatherMap API for a specified city.
- **1.3 Data Formatting:** Processes and formats raw weather data into a human-readable message with emojis and time zone adjustments.
- **1.4 Telegram Notification:** Sends the formatted weather report as a text message to a designated Telegram chat.

Supporting notes provide setup instructions, customization tips, and quick start guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** This block triggers the workflow execution automatically at scheduled times.
- **Nodes Involved:** 
  - Schedule Trigger
  - Schedule Setup (sticky note)
- **Node Details:**

  - **Schedule Trigger**
    - Type: Cron Trigger node
    - Role: Initiates the workflow based on a cron schedule.
    - Configuration: Default cron expression is empty but intended to be set (examples provided in sticky note).
    - Inputs: None (trigger)
    - Outputs: Connects to "Get Weather" node.
    - Edge Cases: Misconfigured cron can cause no triggers or incorrect timing.
    - Notes: User should set cron as per desired frequency, e.g., daily at 8:00 AM (`0 8 * * *`).
  
  - **Schedule Setup** (sticky note)
    - Role: Provides instructions on configuring the cron schedule.
    - Content: Examples for daily, every 6 hours, or twice daily triggers.
    - No technical role beyond documentation.

#### 2.2 Weather Data Retrieval

- **Overview:** Retrieves current weather data for a specified city from OpenWeatherMap API.
- **Nodes Involved:** 
  - Get Weather
  - Weather API (sticky note)
- **Node Details:**

  - **Get Weather**
    - Type: OpenWeatherMap node
    - Role: Fetches weather data using city name and language parameters.
    - Configuration:
      - City: "Kielce,PL" by default (modifiable to any city, e.g., "YourCity,CountryCode").
      - Language: English (`en`)
      - Requires OpenWeatherMap API credentials.
    - Inputs: Trigger from Schedule Trigger node.
    - Outputs: Outputs raw weather JSON to "Format Weather" node.
    - Edge Cases:
      - API key missing or invalid: authentication failure.
      - City not found or misspelled: API returns error or empty data.
      - API rate limits can cause request failures.
  
  - **Weather API** (sticky note)
    - Role: Documents required setup and data retrieved.
    - Content: Notes on credentials, city/language customization, and data fields fetched (temperature, humidity, wind, etc.).

#### 2.3 Data Formatting

- **Overview:** Converts raw weather JSON into a readable, emoji-enhanced text message, including local time formatting for sunrise/sunset.
- **Nodes Involved:** 
  - Format Weather
  - Message Formatting (sticky note)
- **Node Details:**

  - **Format Weather**
    - Type: Function node
    - Role: Processes and formats weather data into a human-friendly string message.
    - Configuration:
      - Contains JavaScript code extracting fields like temperature, humidity, wind speed/direction, cloudiness, precipitation, sunrise and sunset times.
      - Applies timezone offset for sunrise/sunset.
      - Formats message with emojis and line breaks.
    - Inputs: Raw weather JSON from "Get Weather".
    - Outputs: JSON object with key `message` containing the formatted string.
    - Edge Cases:
      - Missing fields (e.g., rain data might be absent) handled with default zero.
      - Timezone offset processing errors if data incomplete.
      - Expression or code errors may break message formatting.
  
  - **Message Formatting** (sticky note)
    - Role: Explains the purpose of formatting and how to customize the message template.
    - Content: Emphasizes readability, emojis, and timezone correctness.

#### 2.4 Telegram Notification

- **Overview:** Sends the formatted weather message to a Telegram chat using a bot.
- **Nodes Involved:** 
  - Send a text message
  - Telegram Delivery (sticky note)
- **Node Details:**

  - **Send a text message**
    - Type: Telegram node (Telegram Bot API)
    - Role: Sends text messages to a specified chat using bot credentials.
    - Configuration:
      - Text: Uses expression to inject the formatted message `={{ $json.message }}`.
      - Chat ID: Placeholder `"XXXXXXX"` to be replaced by user’s chat ID.
      - Credentials: Requires Telegram Bot token configured in node credentials.
    - Inputs: Receives formatted message from "Format Weather".
    - Outputs: None (final node).
    - Edge Cases:
      - Invalid bot token or chat ID causes authentication or delivery failure.
      - Network errors or Telegram API downtime.
      - Message size limits (generally large enough for this use case).
  
  - **Telegram Delivery** (sticky note)
    - Role: Setup instructions for bot creation, token retrieval, and chat ID discovery.
    - Content: Step-by-step guidance to create the Telegram bot and configure chat ID.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role               | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                       |
|---------------------|-------------------------|------------------------------|----------------------|----------------------|-------------------------------------------------------------------------------------------------|
| Workflow Overview   | Sticky Note             | Documentation overview       | None                 | None                 | ## 🤖 Automated Weather Bot ... Setup needed: OpenWeatherMap API key, Telegram bot token, chat ID |
| Schedule Trigger    | Cron Trigger            | Initiates scheduled workflow | None                 | Get Weather          | ⏰ **SCHEDULE TRIGGER** Set when to send weather reports (e.g., 8:00 AM daily)                    |
| Schedule Setup      | Sticky Note             | Cron setup instructions      | None                 | None                 | ⏰ **SCHEDULE TRIGGER** Set when to send weather reports (e.g., 8:00 AM daily)                    |
| Get Weather         | OpenWeatherMap          | Fetches weather data         | Schedule Trigger     | Format Weather       | 🌤️ **WEATHER DATA** Required: OpenWeatherMap credentials, city and language customization        |
| Weather API         | Sticky Note             | Weather API setup notes      | None                 | None                 | 🌤️ **WEATHER DATA** Required: OpenWeatherMap credentials, city and language customization        |
| Format Weather      | Function                | Formats weather data to text | Get Weather          | Send a text message  | 📝 **FORMAT MESSAGE** Converts raw data to readable text with emojis and timezone adjustments    |
| Message Formatting  | Sticky Note             | Message formatting notes     | None                 | None                 | 📝 **FORMAT MESSAGE** Converts raw data to readable text with emojis and timezone adjustments    |
| Send a text message | Telegram                | Sends message to Telegram    | Format Weather       | None                 | 📱 **SEND TO TELEGRAM** Setup bot via @BotFather, get token/chat ID, replace chat ID placeholder  |
| Telegram Delivery   | Sticky Note             | Telegram setup instructions  | None                 | None                 | 📱 **SEND TO TELEGRAM** Setup bot via @BotFather, get token/chat ID, replace chat ID placeholder  |
| Quick Start Guide   | Sticky Note             | Overall setup summary        | None                 | None                 | 🚀 **QUICK START:** Steps for API keys, bot creation, chat ID, testing, activation                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Cron Trigger node:**
   - Name: `Schedule Trigger`
   - Set the schedule for daily execution, e.g., Cron expression: `0 8 * * *` (8:00 AM daily)
   - No credentials needed.

3. **Add an OpenWeatherMap node:**
   - Name: `Get Weather`
   - Credentials: Add OpenWeatherMap API credentials (API key).
   - Parameters:
     - City Name: `Kielce,PL` (or your desired city in format `City,CountryCode`).
     - Language: `en` (or desired language code).
   - Connect `Schedule Trigger` output to `Get Weather` input.

4. **Add a Function node:**
   - Name: `Format Weather`
   - Copy and paste the following JavaScript code to format the weather data:
     ```javascript
     const weather = items[0].json;

     function formatTime(timestamp, timezoneOffset) {
       const date = new Date((timestamp + timezoneOffset) * 1000);
       return date.toISOString().substr(11, 5);
     }

     const city = weather.name;
     const temp = weather.main.temp.toFixed(1);
     const tempMin = weather.main.temp_min.toFixed(1);
     const tempMax = weather.main.temp_max.toFixed(1);
     const feelsLike = weather.main.feels_like.toFixed(1);
     const description = weather.weather[0].description;
     const rain = weather.rain ? weather.rain["1h"] : 0;
     const windSpeed = weather.wind.speed.toFixed(1);
     const windDeg = weather.wind.deg;
     const clouds = weather.clouds.all;
     const sunrise = formatTime(weather.sys.sunrise, weather.timezone);
     const sunset = formatTime(weather.sys.sunset, weather.timezone);

     const message = `Weather in ${city}:\n🌡 Temperature: ${temp}°C (feels like: ${feelsLike}°C)\n📉 Min: ${tempMin}°C, 📈 Max: ${tempMax}°C\n🌧 Precipitation: ${description}, ${rain} mm in the last hour\n💨 Wind: ${windSpeed} m/s, direction ${windDeg}°\n☁️ Cloudiness: ${clouds}%\n🌅 Sunrise: ${sunrise}\n🌇 Sunset: ${sunset}`;

     return [{ json: { message } }];
     ```
   - Connect `Get Weather` output to `Format Weather` input.

5. **Add a Telegram node:**
   - Name: `Send a text message`
   - Credentials: Set up Telegram Bot credentials (token from @BotFather).
   - Parameters:
     - Chat ID: Replace `"XXXXXXX"` with your Telegram chat ID.
     - Text: Use expression `={{ $json.message }}` to send the formatted message.
   - Connect `Format Weather` output to `Send a text message` input.

6. **Add sticky notes for documentation (optional but recommended):**
   - `Workflow Overview`: Summary of workflow purpose and setup.
   - `Schedule Setup`: Instructions for cron expression customization.
   - `Weather API`: Notes on OpenWeatherMap API configuration.
   - `Message Formatting`: Explanation of formatting logic.
   - `Telegram Delivery`: Setup instructions for Telegram bot and chat ID.
   - `Quick Start Guide`: Step-by-step setup summary for users.

7. **Test the workflow:**
   - Run manually or wait for scheduled trigger.
   - Verify Telegram message delivery with correct weather data.

8. **Activate the workflow:**
   - Toggle active switch ON to enable automatic scheduled execution.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| API key required for OpenWeatherMap: free signup at https://openweathermap.org/api               | OpenWeatherMap API documentation                               |
| Telegram bot creation via @BotFather: https://core.telegram.org/bots#6-botfather                  | Telegram Bot API official guide                                |
| How to get your Telegram Chat ID: send message to your bot, then visit https://api.telegram.org/ | Telegram API chat ID retrieval                                 |
| Message formatting uses local timezone offsets from API data for sunrise/sunset display          | Ensures accurate time display per location                    |
| Cron expressions can be customized to change report frequency (e.g., every 6 hours, twice daily) | Cron format guide: https://crontab.guru/                       |
| Emojis used to enhance message readability and visual appeal                                    | Customizable in Function node code                             |

---

This completes the detailed documentation and reconstruction guide for the "Daily Weather Reports with OpenWeatherMap and Telegram Bot" workflow.