Daily & On-demand Weather Reports with OpenWeatherMap to Telegram

https://n8nworkflows.xyz/workflows/daily---on-demand-weather-reports-with-openweathermap-to-telegram-5838


# Daily & On-demand Weather Reports with OpenWeatherMap to Telegram

### 1. Workflow Overview

This workflow, titled **Daily & On-demand Weather Reports with OpenWeatherMap to Telegram**, automates the retrieval and delivery of weather information for specified locations. It supports two main use cases:

- **Daily Scheduled Weather Report:** Automatically fetches and sends the current weather every day at 8:00 AM IST.
- **On-demand Weather Report via Form Submission:** Allows users to request weather updates for any city and country by submitting a form.

The workflow consists of four main logical blocks:

- **1.1 Input Reception:** Triggers that start the workflow either on a daily schedule or via form submission.
- **1.2 Weather Data Retrieval:** Calls the OpenWeatherMap API to get current weather data based on city and country inputs.
- **1.3 Message Formatting:** Processes raw weather data into a human-readable message including temperature, humidity, wind, pressure, sunrise/sunset times, and date formatted in IST (Indian Standard Time).
- **1.4 Output Delivery:** Sends the formatted weather message to a designated Telegram chat via a Telegram Bot.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:** This block triggers the workflow either daily at 8:00 AM IST or upon a user submitting a weather request form.
- **Nodes Involved:** `Schedule Trigger`, `On form submission`

##### Node: Schedule Trigger
- **Type:** Schedule Trigger
- **Role:** Initiates the workflow automatically every day at 8:00 AM IST.
- **Configuration:** 
  - Trigger set to run once daily at hour 8 (08:00 AM).
  - Timezone assumed to be server default; formatting to IST happens downstream.
- **Inputs:** None (trigger node).
- **Outputs:** Connected to `Get Weather Data`.
- **Edge Cases:** Server timezone drift can affect exact trigger time; ensure server time aligns with IST or adjust accordingly.
- **Version:** 1.2

##### Node: On form submission
- **Type:** Form Trigger
- **Role:** Starts workflow on user form submission with inputs for city and country.
- **Configuration:** 
  - Form titled “Daily Weather Bot”.
  - Fields: “City Name”, “Country Name”.
- **Inputs:** HTTP webhook-based (form input).
- **Outputs:** Connected to `Get Weather Data`.
- **Edge Cases:** Invalid or missing city/country input may cause API errors downstream.
- **Version:** 2.2

---

#### Block 1.2: Weather Data Retrieval

- **Overview:** Retrieves current weather data from OpenWeatherMap API using city and country parameters.
- **Nodes Involved:** `Get Weather Data`

##### Node: Get Weather Data
- **Type:** HTTP Request
- **Role:** Requests current weather info from OpenWeatherMap API.
- **Configuration:** 
  - URL dynamically constructed using inputs:  
    `https://api.openweathermap.org/data/2.5/weather?q={{ $json['City Name'] }},{{ $json['Country Name'] }}&APPID=key&units=metric`
  - `APPID=key` is a placeholder for the OpenWeatherMap API key (must be replaced with actual key).
  - Metric units specified for temperature and wind speed.
- **Inputs:** Receives city and country data from either trigger node.
- **Outputs:** Provides JSON weather data to `Format Weather Message`.
- **Edge Cases:** 
  - API key missing or invalid → authentication failure.
  - City/country not found → API returns error or empty data.
  - Network/timeouts.
- **Version:** 4.2

---

#### Block 1.3: Message Formatting

- **Overview:** Converts raw API weather data into a formatted string message with localized date and time.
- **Nodes Involved:** `Format Weather Message`

##### Node: Format Weather Message
- **Type:** Set
- **Role:** Uses JavaScript expression to build a detailed weather report message.
- **Configuration:** 
  - Constructs message with fields:  
    - Date (weekday, day, month, year) in IST timezone.  
    - Weather condition description, temperature (°C), humidity (%), wind speed (m/s), pressure (hPa).  
    - Sunrise and sunset times converted from UNIX timestamp to IST time strings.
  - Uses JavaScript IIFE (Immediately Invoked Function Expression) within expression editor for dynamic data formatting.
- **Inputs:** Weather JSON from `Get Weather Data`.
- **Outputs:** Passes formatted message string to `Send Telegram Message`.
- **Edge Cases:** 
  - Missing or malformed JSON fields cause expression evaluation errors.
  - Timezone conversion requires correct data types.
- **Version:** 3.4

---

#### Block 1.4: Output Delivery

- **Overview:** Sends the formatted weather message to a Telegram chat via Telegram Bot API.
- **Nodes Involved:** `Send Telegram Message`

##### Node: Send Telegram Message
- **Type:** Telegram
- **Role:** Sends message text to a Telegram chat.
- **Configuration:** 
  - Message text is injected from the `message` field from previous node.
  - `chatId` set to `telegramChatId` (must be replaced with actual chat ID).
  - Attribution disabled to keep message clean.
  - Uses configured Telegram API credentials (OAuth or Bot Token).
- **Inputs:** Receives formatted message string.
- **Outputs:** None (endpoint).
- **Edge Cases:** 
  - Invalid chat ID or credentials cause message send failure.
  - Telegram API rate limits or downtime.
- **Version:** 1.2

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role           | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                     |
|----------------------|-------------------|--------------------------|-----------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger  | Daily workflow trigger   | -                     | Get Weather Data         | ## Schedule Trigger (08:00 AM daily) ⮕ HTTP Request (OpenWeatherMap) ⮕ Set (Format message) ⮕ Telegram |
| On form submission   | Form Trigger      | User input trigger       | -                     | Get Weather Data         |                                                                                                |
| Get Weather Data     | HTTP Request      | Fetch weather data       | Schedule Trigger, On form submission | Format Weather Message |                                                                                                |
| Format Weather Message | Set             | Format weather text      | Get Weather Data       | Send Telegram Message    |                                                                                                |
| Send Telegram Message | Telegram          | Send message to Telegram | Format Weather Message | -                        | ## Telegram Output -Bot Setting                                                                |
| Sticky Note3         | Sticky Note       | Visual guidance          | -                     | -                        | ## Schedule Trigger (08:00 AM daily) ⮕ HTTP Request (OpenWeatherMap) ⮕ Set (Format message with weather + atmosphere + IST) ⮕ Telegram |
| Sticky Note2         | Sticky Note       | Visual guidance          | -                     | -                        | ## Telegram Output -Bot Setting                                                                |
| Sticky Note1         | Sticky Note       | Visual guidance          | -                     | -                        | ## api calling openweathermap                                                                   |
| Sticky Note          | Sticky Note       | Visual guidance          | -                     | -                        | ## Daily Schudule Trigger Mode: Every Day Time: Hour: 08 Minute: 00 This means the workflow will run once per day at 08:00 AM. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named "Daily Weather Bot".

2. **Add a Schedule Trigger node:**
   - Set the interval to "Every Day".
   - Set trigger hour to 8 (08:00 AM).
   - Leave minutes at 0.
   - This node triggers the daily weather fetch.

3. **Add a Form Trigger node:**
   - Title the form "Daily Weather Bot".
   - Add two input fields named "City Name" and "Country Name".
   - This node triggers on user-submitted city and country.

4. **Add an HTTP Request node named "Get Weather Data":**
   - Set the HTTP Method to GET.
   - Set URL to:  
     `https://api.openweathermap.org/data/2.5/weather?q={{ $json["City Name"] }},{{ $json["Country Name"] }}&APPID=YOUR_OPENWEATHERMAP_API_KEY&units=metric`  
     Replace `YOUR_OPENWEATHERMAP_API_KEY` with your actual OpenWeatherMap API key.
   - This node receives input from both triggers.

5. **Connect both the Schedule Trigger and Form Trigger nodes to the "Get Weather Data" node.**

6. **Add a Set node named "Format Weather Message":**
   - Add a new field `message` of type String.
   - Use the following expression for the value (adapted for n8n expression editor):

```javascript
=(() => {
  const desc = $json.weather[0].description;
  const temp = $json.main.temp;
  const hum = $json.main.humidity;
  const pres = $json.main.pressure;
  const wind = $json.wind.speed;
  const city = $json.name;
  const country = $json.sys.country;

  const sunrise = new Date($json.sys.sunrise * 1000).toLocaleTimeString('en-IN', { timeZone: 'Asia/Kolkata' });
  const sunset = new Date($json.sys.sunset * 1000).toLocaleTimeString('en-IN', { timeZone: 'Asia/Kolkata' });

  const now = new Date();
  const dateStr = now.toLocaleDateString('en-IN', {
    weekday: 'long', year: 'numeric', month: 'long', day: 'numeric',
    timeZone: 'Asia/Kolkata'
  });

  return `📅 ${dateStr}
🌤 Weather in ${city}, ${country}:
Condition: ${desc}
Temperature: ${temp}°C
💧 Humidity: ${hum}%
🌬 Wind Speed: ${wind} m/s
🔼 Pressure: ${pres} hPa
🌅 Sunrise: ${sunrise}
🌇 Sunset: ${sunset}`;
})()
```

7. **Connect "Get Weather Data" to "Format Weather Message".**

8. **Add a Telegram node named "Send Telegram Message":**
   - Set `chatId` to your Telegram chat ID where messages will be sent.
   - Set `text` to `{{ $json.message }}` to send the formatted message.
   - Disable attribution (`appendAttribution` set to false).
   - Configure Telegram API credentials by creating a Telegram Bot token credential in n8n.
   
9. **Connect "Format Weather Message" to "Send Telegram Message".**

10. **Add Sticky Notes for documentation and clarity (optional but recommended):**
    - Note 1: Explains daily schedule trigger at 08:00 AM.
    - Note 2: Explains API call to OpenWeatherMap.
    - Note 3: Explains message formatting and Telegram output.

11. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses Indian Standard Time (Asia/Kolkata) for all date and time formatting, including sunrise and sunset times. | Timezone handling best practice                  |
| Replace placeholders `APPID=key` and `telegramChatId` with your actual OpenWeatherMap API key and Telegram chat ID.          | API Key and chat ID required for operation       |
| Telegram credentials require creation of a Telegram Bot and OAuth2 token setup in n8n credentials section.                   | Telegram Bot API documentation: https://core.telegram.org/bots/api |
| Ensure the server running n8n has a stable internet connection for API calls and Telegram message delivery.                   | Network reliability impacts workflow success     |
| If city/country input is invalid, OpenWeatherMap returns an error, which is not explicitly handled in this workflow.          | Consider adding error handling or fallback nodes |

---

This documentation provides a complete understanding for replicating, modifying, or extending the Daily Weather Bot workflow using n8n. It highlights key configurations, potential failure points, and integration details with OpenWeatherMap and Telegram APIs.

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow export. It complies fully with content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.