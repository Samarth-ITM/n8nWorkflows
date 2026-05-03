Send Daily Weather Reports to Email with OpenWeatherMap and Gmail

https://n8nworkflows.xyz/workflows/send-daily-weather-reports-to-email-with-openweathermap-and-gmail-6465


# Send Daily Weather Reports to Email with OpenWeatherMap and Gmail

### 1. Workflow Overview

This workflow automates the process of sending a daily weather report email using the OpenWeatherMap API and Gmail. It is ideal for users who want to receive up-to-date weather information for a specified city delivered directly to their inbox each morning.

**Logical blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specific time (8 AM by default).
- **1.2 Weather Data Retrieval:** Fetches current weather data for a preset city from OpenWeatherMap.
- **1.3 Weather Report Formatting:** Processes and formats the raw weather data into a human-readable report.
- **1.4 Email Dispatch:** Sends the formatted weather report to a designated email address via Gmail.
- **1.5 Documentation and Setup Notes:** Provides inline notes for users about the workflow’s purpose and setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the entire workflow automatically every day at 8 AM.

- **Nodes Involved:**  
  - Daily Schedule (8 AM)

- **Node Details:**  
  - **Node Name:** Daily Schedule (8 AM)  
  - **Type:** Cron Trigger  
  - **Role:** Initiates the workflow based on a scheduled time event.  
  - **Configuration:** Default cron-based trigger set to run daily at 08:00. No explicit parameters modified beyond default time.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connects to "Fetch Weather Data" node.  
  - **Version Requirements:** n8n version supporting cron trigger (widely supported).  
  - **Potential Failures:** None typical unless the n8n instance is down or misconfigured time zone settings affect triggering.  
  - **Sub-workflow:** None.

#### 1.2 Weather Data Retrieval

- **Overview:**  
  Makes an HTTP GET request to the OpenWeatherMap API to fetch current weather data for a specified city.

- **Nodes Involved:**  
  - Fetch Weather Data

- **Node Details:**  
  - **Node Name:** Fetch Weather Data  
  - **Type:** HTTP Request  
  - **Role:** Retrieves live weather data from an external API.  
  - **Configuration:**  
    - HTTP Method: GET (default)  
    - URL Template: `https://api.openweathermap.org/data/2.5/weather?q={{ encodeURIComponent('London') }}&units=metric&appid=YOUR_OPENWEATHERMAP_API_KEY`  
    - The city is hardcoded as "London" but can be changed.  
    - API Key placeholder `YOUR_OPENWEATHERMAP_API_KEY` must be replaced with a valid OpenWeatherMap API key.  
    - Units set to metric for temperature in Celsius.  
  - **Expressions:** Uses `encodeURIComponent` to safely encode the city name.  
  - **Inputs:** Triggered by "Daily Schedule (8 AM)".  
  - **Outputs:** Passes JSON weather data to "Format Weather Report".  
  - **Version Requirements:** HTTP Request node version 3 used; ensure n8n supports this version or adapt accordingly.  
  - **Potential Failures:**  
    - Invalid or missing API key (401 Unauthorized).  
    - Network errors (timeouts, DNS issues).  
    - API rate limiting by OpenWeatherMap.  
    - City name errors (404 Not Found).  
  - **Sub-workflow:** None.

#### 1.3 Weather Report Formatting

- **Overview:**  
  Processes the raw JSON weather data and creates a friendly, formatted textual weather report.

- **Nodes Involved:**  
  - Format Weather Report

- **Node Details:**  
  - **Node Name:** Format Weather Report  
  - **Type:** Code (JavaScript)  
  - **Role:** Extracts relevant fields from API response and generates a textual summary.  
  - **Configuration:**  
    - JavaScript code accesses the first input item’s JSON, extracting fields like city name, weather description, temperature, feels like, humidity, and wind speed.  
    - Capitalizes first letter of weather description.  
    - Constructs a multi-line string report with greetings and weather details.  
    - Returns an object containing `report` (string) and `city` (string) keys.  
  - **Expressions / Variables:** Uses `$input.first().json` to access input data.  
  - **Inputs:** Receives JSON from "Fetch Weather Data".  
  - **Outputs:** Passes formatted report and city name to "Send Email Report".  
  - **Version Requirements:** Code node version 2 in use; ensure compatibility with JavaScript environment.  
  - **Potential Failures:**  
    - Data structure changes in API response causing undefined errors.  
    - Null or missing fields in weather data.  
    - JavaScript runtime errors if unexpected data types.  
  - **Sub-workflow:** None.

#### 1.4 Email Dispatch

- **Overview:**  
  Sends the formatted weather report via Gmail to a specified email address.

- **Nodes Involved:**  
  - Send Email Report

- **Node Details:**  
  - **Node Name:** Send Email Report  
  - **Type:** Gmail  
  - **Role:** Sends an email message with the weather report.  
  - **Configuration:**  
    - Recipient email set to `your_email@example.com` (placeholder to be replaced).  
    - Subject line dynamically constructed as `Daily Weather Report for {{ $json.city }}`.  
    - Email body set to the formatted report from the previous node.  
    - Email type set as plain text.  
  - **Credentials:** Uses OAuth2 credentials named "Gmail account 2" (configured separately in n8n).  
  - **Inputs:** Receives report and city from "Format Weather Report".  
  - **Outputs:** None (end node).  
  - **Version Requirements:** Gmail node version 2.1 used; confirm OAuth2 compatibility.  
  - **Potential Failures:**  
    - Authentication errors if credentials are revoked or invalid.  
    - Recipient address misconfiguration (bounce).  
    - SMTP or Gmail API quota limits.  
  - **Sub-workflow:** None.

#### 1.5 Documentation and Setup Notes

- **Overview:**  
  Provides users with information about the workflow’s purpose, author, and instructions for initial setup.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Node Name:** Sticky Note  
    - **Type:** Sticky Note  
    - **Role:** Describes the workflow, author, and use cases.  
    - **Content:**  
      ```
      ## Daily Weather Report to Email

      **Author: David Olusola**

      Description: This workflow runs daily, fetches current weather data for a specified city using the OpenWeatherMap API, and sends a formatted weather report to your email.

      Use Case: Daily digests, automated reports, personalized notifications.
      ```
  - **Node Name:** Sticky Note1  
    - **Type:** Sticky Note  
    - **Role:** Lists setup instructions and prerequisites.  
    - **Content:**  
      ```
      ## Set up steps
      Before Running :

      OpenWeatherMap API Key: Go to OpenWeatherMap and sign up for a free API key. Replace YOUR_OPENWEATHERMAP_API_KEY in the "Fetch Weather Data" node's URL.

      Adjust City: Change London in the "Fetch Weather Data" node's URL to your desired city.

      Set up Gmail Credentials: In n8n, create a Gmail OAuth2 credential.

      Recipient Email: Replace your_email@example.com in the "Send Email Report" node with your actual email address.

      Schedule: The "Daily Schedule (8 AM)" node is set to run every day at 8 AM. You can adjust the hour and minute parameters as needed.
      ```

---

### 3. Summary Table

| Node Name             | Node Type       | Functional Role               | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                       |
|-----------------------|-----------------|------------------------------|-----------------------|----------------------|-------------------------------------------------------------------------------------------------|
| Daily Schedule (8 AM)  | Cron Trigger    | Scheduled daily trigger       | —                     | Fetch Weather Data    |                                                                                                |
| Fetch Weather Data     | HTTP Request    | Retrieve weather data from API| Daily Schedule (8 AM)  | Format Weather Report |                                                                                                |
| Format Weather Report  | Code            | Format raw data to text report| Fetch Weather Data     | Send Email Report     |                                                                                                |
| Send Email Report      | Gmail           | Send email with weather report| Format Weather Report  | —                    |                                                                                                |
| Sticky Note           | Sticky Note     | Workflow description          | —                     | —                    | ## Daily Weather Report to Email\n\n**Author: David Olusola**\n\nDescription: ...               |
| Sticky Note1          | Sticky Note     | Setup instructions            | —                     | —                    | ## Set up steps\nBefore Running :\n\nOpenWeatherMap API Key: ...                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger Node:**  
   - Add a **Cron** node named `Daily Schedule (8 AM)`.  
   - Configure it to trigger daily at 08:00 (hour = 8, minute = 0).  
   - No credentials required.

2. **Create the Weather Data Retrieval Node:**  
   - Add an **HTTP Request** node named `Fetch Weather Data`.  
   - Set HTTP Method to GET.  
   - Set URL to:  
     ```
     https://api.openweathermap.org/data/2.5/weather?q={{ encodeURIComponent('London') }}&units=metric&appid=YOUR_OPENWEATHERMAP_API_KEY
     ```  
     Replace `'London'` with desired city name.  
     Replace `YOUR_OPENWEATHERMAP_API_KEY` with your actual OpenWeatherMap API key.  
   - No authentication needed for the HTTP request node itself.  
   - Connect `Daily Schedule (8 AM)` node’s output to this node’s input.

3. **Create the Weather Report Formatting Node:**  
   - Add a **Code** node named `Format Weather Report`.  
   - Use JavaScript mode.  
   - Paste the following code (adjust if needed):  
     ```javascript
     const weatherData = $input.first().json;

     const city = weatherData.name;
     const description = weatherData.weather[0].description;
     const temp = weatherData.main.temp;
     const feelsLike = weatherData.main.feels_like;
     const humidity = weatherData.main.humidity;
     const windSpeed = weatherData.wind.speed;

     const report = `
     Good morning!

     Here's your daily weather report for ${city}:

     Conditions: ${description.charAt(0).toUpperCase() + description.slice(1)}
     Temperature: ${temp}°C (Feels like: ${feelsLike}°C)
     Humidity: ${humidity}%
     Wind Speed: ${windSpeed} m/s

     Have a great day!
     `;

     return [{ json: { report: report, city: city } }];
     ```  
   - Connect `Fetch Weather Data` node’s output to this node’s input.

4. **Create the Email Sending Node:**  
   - Add a **Gmail** node named `Send Email Report`.  
   - Configure Credentials: Create or select an existing **Gmail OAuth2** credential in n8n. This requires setting up OAuth2 with Google Cloud Console and authorizing n8n.  
   - Set **Send To** to your target email address (replace `your_email@example.com`).  
   - Set **Subject** to:  
     ```
     Daily Weather Report for {{ $json.city }}
     ```  
   - Set **Message** to:  
     ```
     {{ $json.report }}
     ```  
   - Set **Email Type** to `text`.  
   - Connect `Format Weather Report` node’s output to this node’s input.

5. **Add Documentation Sticky Notes:**  
   - Add a **Sticky Note** node named `Sticky Note` with content describing the workflow purpose, author, and use cases.  
   - Add another **Sticky Note** node named `Sticky Note1` with setup instructions about API keys, credentials, and email addresses.  
   - Position these notes visibly for users.

6. **Connect the nodes sequentially:**  
   `Daily Schedule (8 AM)` → `Fetch Weather Data` → `Format Weather Report` → `Send Email Report`

7. **Test the workflow:**  
   - Run manually to verify API key and email sending.  
   - Monitor for errors such as invalid API keys or email auth problems.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Replace `YOUR_OPENWEATHERMAP_API_KEY` with a valid API key from https://openweathermap.org/api                   | OpenWeatherMap API documentation                            |
| Gmail OAuth2 credential requires Google Cloud Console setup with OAuth consent and scopes for Gmail API           | https://developers.google.com/gmail/api/quickstart/js      |
| Adjust the city parameter in the HTTP Request URL to receive weather information for any supported location      | OpenWeatherMap city list or API docs                        |
| The workflow runs daily at 8 AM by default; you can modify the cron node settings to change the schedule          | n8n Cron Trigger documentation                              |
| Email sending depends on valid OAuth2 credentials and may require re-authentication if tokens expire              | n8n Gmail node and OAuth2 documentation                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.