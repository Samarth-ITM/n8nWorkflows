Send daily weather forecast from Meteosource to Email (with Gmail)

https://n8nworkflows.xyz/workflows/send-daily-weather-forecast-from-meteosource-to-email--with-gmail--7815


# Send daily weather forecast from Meteosource to Email (with Gmail)

### 1. Workflow Overview

This workflow automates the daily sending of a weather forecast email using data from the Meteosource API and Gmail. It is designed to run every day at 07:00, retrieve the weather forecast for a predefined location, and then send an email summarizing the weather. The email subject contains today's weather summary, while the body includes tomorrow's forecast. This setup provides a quick weather check early in the day.

**Logical blocks:**

- **1.1 Schedule Trigger:** Initiates the workflow daily at a specified time.  
- **1.2 Configuration Setup:** Defines the location (`place_id`) for the weather forecast and the recipient email address.  
- **1.3 Weather Data Retrieval:** Calls the Meteosource API to fetch detailed daily weather data for the configured location.  
- **1.4 Email Sending:** Sends a plain text email via Gmail with the weather summary, formatted with today's weather in the subject and tomorrow's weather in the body.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Triggers the workflow execution automatically at 07:00 every day.

- **Nodes Involved:**  
  - Schedule - every day 07:00

- **Node Details:**  

  - **Schedule - every day 07:00**  
    - Type: Schedule Trigger  
    - Configuration: Runs daily, triggering at hour 07:00 UTC by default (can be adjusted).  
    - Expressions/variables: None used here; fixed schedule.  
    - Input: None (trigger node).  
    - Output: Triggers the next node in sequence.  
    - Version: 1.2  
    - Edge cases/failures: Time zone considerations (configured as UTC); if n8n instance is offline or paused, trigger won't fire.  
    - Sub-workflow: None.

#### 2.2 Configuration Setup

- **Overview:**  
  Sets up essential variables for the workflow: the location identifier for Meteosource and the recipient email address.

- **Nodes Involved:**  
  - Config - place_id and recipient

- **Node Details:**  

  - **Config - place_id and recipient**  
    - Type: Set node  
    - Configuration: Two static string parameters:  
      - `place_id`: "london" (can be changed to any valid Meteosource location ID or name)  
      - `send_to_email`: "example@example.com" (replace with the intended recipient email)  
    - Expressions/variables: None; static assignment.  
    - Input: Receives trigger from the Schedule node.  
    - Output: Passes the JSON including `place_id` and `send_to_email` to the next node.  
    - Version: 3.4  
    - Edge cases/failures: Misconfigured or invalid place_id may cause API errors downstream; invalid recipient email leads to email sending failure.  
    - Sub-workflow: None.

#### 2.3 Weather Data Retrieval

- **Overview:**  
  Calls the Meteosource API to retrieve all weather data sections for the configured location, in metric units and English language.

- **Nodes Involved:**  
  - Weather API - Meteosource

- **Node Details:**  

  - **Weather API - Meteosource**  
    - Type: HTTP Request node  
    - Configuration:  
      - Method: GET (default)  
      - URL: `https://www.meteosource.com/api/v1/free/point`  
      - Query parameters:  
        - `place_id` dynamically set from previous node data (`={{ $json.place_id }}`)  
        - `sections`: "all" (retrieves all weather data sections)  
        - `timezone`: "UTC"  
        - `language`: "en"  
        - `units`: "metric"  
      - Authentication: HTTP Query Auth with Meteosource API credentials  
    - Expressions/variables: Uses expression to inject `place_id` from JSON input.  
    - Input: Receives JSON with `place_id` from Config node.  
    - Output: Emits JSON response with detailed weather data, including daily forecasts.  
    - Version: 4.2  
    - Edge cases/failures:  
      - API authentication failure (invalid or expired credentials)  
      - Network timeout or connectivity issues  
      - Invalid `place_id` causing API errors or empty responses  
      - Rate limiting by Meteosource API  
    - Sub-workflow: None.

#### 2.4 Email Sending

- **Overview:**  
  Sends a plain text email via Gmail with the weather forecast summary: today's weather in the subject and tomorrow's weather in the email body.

- **Nodes Involved:**  
  - Email - send weather summary

- **Node Details:**  

  - **Email - send weather summary**  
    - Type: Gmail node  
    - Configuration:  
      - Recipient email: Dynamically set using expression `={{ $('Config - place_id and recipient').item.json.send_to_email }}`  
      - Subject: "Weather today: {{ $json.daily.data[0].summary }}" — uses the first day's summary from the API response.  
      - Message body:  
        ```
        Tomorrow:  {{ $json.daily.data[1].summary }}

        Powered by Meteosource
        ```  
        — uses the second day's summary from the API response.  
      - Email Type: Plain text  
      - Options: Attribution disabled  
    - Credentials: Gmail OAuth2 credentials named "Jim Halpert"  
    - Expressions/variables: Uses expressions to reference data from the Meteosource API response and the Config node.  
    - Input: Receives weather JSON from Meteosource node.  
    - Output: Email is sent; no further nodes connected.  
    - Version: 2.1  
    - Edge cases/failures:  
      - Invalid Gmail OAuth2 credentials or expired token  
      - Recipient email invalid or blocked  
      - Expression failures if API data structure changes or is empty  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                  | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                             |
|-----------------------------|--------------------|---------------------------------|-----------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow description         | Sticky Note        | Documentation / overview        | None                        | None                         | ## Workflow overview  Daily weather email. Runs at 07:00. Pulls Meteosource forecast for the configured place. Sends a short Gmail message with today in the subject and tomorrow in the body. Quick check before the day starts.  Setup  - **Config - place_id and recipient**  - `place_id` - your Meteosource location (example: london or a specific ID)  - `send_to_email` - where to send the email  - **Credentials**  - Meteosource http query auth for the API call  - Gmail OAuth2 for sending  - **Schedule**  - Default is 07:00. Adjust the hour in the schedule node.  How it flows  1) Schedule fires  2) Config provides `place_id` and `send_to_email`  3) Meteosource API returns daily data  4) Gmail node sends:  - Subject: today’s summary  - Body: tomorrow’s summary  Tip: keep it simple first. Then, if needed, add HTML email, more fields, or multiple recipients. |
| Schedule - every day 07:00   | Schedule Trigger   | Start workflow daily at 07:00   | None                        | Config - place_id and recipient |                                                                                                       |
| Config - place_id and recipient | Set               | Define location and recipient   | Schedule - every day 07:00  | Weather API - Meteosource     |                                                                                                       |
| Weather API - Meteosource    | HTTP Request       | Retrieve weather data from API  | Config - place_id and recipient | Email - send weather summary |                                                                                                       |
| Email - send weather summary | Gmail              | Send weather forecast email     | Weather API - Meteosource   | None                         |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 07:00 (UTC default). Adjust trigger hour as needed.

2. **Add a Set node for configuration:**  
   - Name: `Config - place_id and recipient`  
   - Add two fields of type string:  
     - `place_id` with the desired Meteosource location ID or name (e.g., "london")  
     - `send_to_email` with the email address to receive the forecast (e.g., "example@example.com")  
   - Connect Schedule Trigger output to this Set node input.

3. **Add an HTTP Request node to call Meteosource API:**  
   - Name: `Weather API - Meteosource`  
   - Method: GET  
   - URL: `https://www.meteosource.com/api/v1/free/point`  
   - Query parameters:  
     - `place_id` = `={{ $json.place_id }}` (expression referring to the Set node output)  
     - `sections` = `all`  
     - `timezone` = `UTC`  
     - `language` = `en`  
     - `units` = `metric`  
   - Authentication: Set to HTTP Query Auth with Meteosource API credentials (configure credentials with your API key).  
   - Connect output of the Set node to this HTTP Request node.

4. **Add a Gmail node to send the email:**  
   - Name: `Email - send weather summary`  
   - Credentials: Set Gmail OAuth2 credentials (e.g., "Jim Halpert") with proper OAuth2 setup.  
   - To: `={{ $('Config - place_id and recipient').item.json.send_to_email }}` (expression to get recipient email)  
   - Subject: `=Weather today:  {{ $json.daily.data[0].summary }}`  
   - Message:  
     ```
     Tomorrow:  {{ $json.daily.data[1].summary }}

     Powered by Meteosource
     ```  
   - Email Type: Plain text  
   - Options: Disable append attribution  
   - Connect output of HTTP Request node to this Gmail node.

5. **Confirm and activate the workflow.**  
   - Test by running manually or waiting for scheduled trigger.  
   - Verify email reception and correct weather data in subject and body.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses the free Meteosource API endpoint requiring HTTP Query authentication with an API key. Ensure you have an active Meteosource account and API key configured in n8n credentials.                                                               | Meteosource API documentation: https://www.meteosource.com/api                                   |
| Gmail node requires OAuth2 credential setup with Gmail account allowing sending emails via API. Make sure to configure OAuth2 credentials properly in n8n.                                                                                                      | Gmail OAuth2 n8n docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/   |
| The workflow uses simple plain text emails; to enhance, consider adding HTML formatting or multiple recipients as needed.                                                                                                                                       |                                                                                                |
| Timezone is fixed to UTC in the API call and scheduling; adjust if your local time zone differs by modifying the Schedule node or API parameters.                                                                                                               |                                                                                                |
| The workflow is designed to be minimal and easy to customize; start simple, then add advanced features like richer email content or error handling as needed.                                                                                                   |                                                                                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.