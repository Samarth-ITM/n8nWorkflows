Send daily weather updates to a phone number via Plivo

https://n8nworkflows.xyz/workflows/send-daily-weather-updates-to-a-phone-number-via-plivo-1005


# Send daily weather updates to a phone number via Plivo

### 1. Workflow Overview

This workflow automates the process of sending daily weather updates via SMS to a specified phone number using Plivo. It is designed to run once every day at 9 AM and fetches the current weather for a predefined city (Berlin by default) from OpenWeatherMap. The weather temperature data is then formatted into a message and sent as an SMS through the Plivo service.

The workflow’s logic can be grouped into three main functional blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified time.
- **1.2 Weather Data Retrieval:** Fetches current weather details for the chosen city.
- **1.3 SMS Dispatch:** Sends the weather update message via Plivo SMS service.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow on a daily schedule at 9 AM.

- **Nodes Involved:**  
  - Cron

- **Node Details:**  
  - **Node Name:** Cron  
    - **Type & Role:** Cron node, serves as the scheduled trigger for workflow execution.  
    - **Configuration:** Set to trigger once daily at 9 AM. The hour is set to 9, with no minute specified (defaults to 0).  
    - **Expressions/Variables:** None.  
    - **Input/Output:** No input; outputs trigger signal to OpenWeatherMap node.  
    - **Version Requirements:** Compatible with n8n version 1 and above.  
    - **Failure Modes:** Misconfigured time settings or system clock issues could delay or prevent trigger.  
    - **Sub-workflow:** None.

#### 2.2 Weather Data Retrieval

- **Overview:**  
  This block fetches current weather data for a specific city (default: Berlin) using OpenWeatherMap API.

- **Nodes Involved:**  
  - OpenWeatherMap

- **Node Details:**  
  - **Node Name:** OpenWeatherMap  
    - **Type & Role:** OpenWeatherMap node, retrieves current weather information.  
    - **Configuration:** City name set to "berlin" (case-insensitive). To customize, replace "berlin" with desired city name.  
    - **Credentials:** Uses OpenWeatherMap API credentials (referred to as "owm").  
    - **Expressions/Variables:** None in parameters; city is static.  
    - **Input/Output:** Receives trigger from Cron; outputs JSON data containing weather details, notably temperature under `main.temp`.  
    - **Version Requirements:** Compatible with n8n version 1 and above.  
    - **Failure Modes:**  
      - Invalid API credentials causing auth errors.  
      - Incorrect city name returning errors or empty data.  
      - API rate limits or network issues.  
    - **Sub-workflow:** None.

#### 2.3 SMS Dispatch

- **Overview:**  
  This block sends an SMS containing the current temperature to a phone number using Plivo.

- **Nodes Involved:**  
  - Plivo

- **Node Details:**  
  - **Node Name:** Plivo  
    - **Type & Role:** Plivo node, responsible for sending SMS messages.  
    - **Configuration:**  
      - Message text dynamically includes temperature extracted from prior node’s output:  
        `Hey! The temperature outside is {{$node["OpenWeatherMap"].json["main"]["temp"]}}°C.`  
      - Phone number and other required parameters are set via Plivo credentials and node settings (not explicitly shown but assumed configured).  
    - **Credentials:** Uses Plivo API credentials (named "Plivo API Credentials").  
    - **Expressions/Variables:** Uses expression to fetch temperature from OpenWeatherMap node output.  
    - **Input/Output:** Input from OpenWeatherMap node; outputs SMS send status (not used further in this workflow).  
    - **Version Requirements:** Compatible with n8n version 1 and above.  
    - **Failure Modes:**  
      - Authentication failures due to invalid Plivo credentials.  
      - Network issues causing SMS send failures.  
      - Message formatting errors if temperature data is missing or malformed.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name        | Node Type                  | Functional Role           | Input Node(s)     | Output Node(s)  | Sticky Note                                                                                  |
|------------------|----------------------------|---------------------------|-------------------|-----------------|----------------------------------------------------------------------------------------------|
| Cron             | Cron                       | Scheduled trigger         | -                 | OpenWeatherMap  | The Cron node will trigger the workflow daily at 9 AM.                                      |
| OpenWeatherMap   | OpenWeatherMap             | Fetch weather data        | Cron              | Plivo           | This node will return data about the current weather in Berlin. You can change the city name.|
| Plivo            | Plivo                      | Send SMS with weather     | OpenWeatherMap    | -               | This node will send an SMS with the weather update, which was sent by the previous node.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Cron Node:**  
   - Add a new node of type "Cron".  
   - Set it to trigger daily at 9 AM by configuring the time to hour = 9, minute = 0.  
   - No credentials needed.

2. **Create the OpenWeatherMap Node:**  
   - Add a new node of type "OpenWeatherMap".  
   - Configure the city name parameter to "berlin" (or your preferred city).  
   - Set up credentials for OpenWeatherMap API: create or select existing credentials named (e.g.) "owm".  
   - Connect the output of the Cron node to the input of this node.

3. **Create the Plivo Node:**  
   - Add a new node of type "Plivo".  
   - Configure the message parameter with the expression:  
     `Hey! The temperature outside is {{$node["OpenWeatherMap"].json["main"]["temp"]}}°C.`  
   - Configure the recipient phone number (this will be set in the node parameters, typically in the “To” or “Destination” number field).  
   - Set up Plivo API credentials (named, for example, "Plivo API Credentials") with valid authentication details.  
   - Connect the output of the OpenWeatherMap node to the input of this node.

4. **Verify Connections:**  
   - Ensure the data flows: Cron → OpenWeatherMap → Plivo.

5. **Test the Workflow:**  
   - Manually execute or wait for scheduled trigger to verify SMS delivery.

6. **Optional Customizations:**  
   - Change the city name in the OpenWeatherMap node to your preferred location.  
   - Modify the SMS message content in the Plivo node as needed.  
   - Add error handling nodes if desired (e.g., Notify on failure).

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| To customize the weather location, update the city name parameter in the OpenWeatherMap node accordingly.    | Workflow description                                      |
| For Plivo setup details and API credentials, refer to Plivo official docs: https://www.plivo.com/docs/        | Plivo API documentation                                   |
| OpenWeatherMap API key registration and usage details: https://openweathermap.org/api                         | OpenWeatherMap API documentation                           |
| The example message uses Celsius; adjust units in OpenWeatherMap node if required for Fahrenheit or others.   | OpenWeatherMap node parameter settings                    |
| The workflow screenshot is referenced but not included here.                                                  | Provided in the original workflow metadata                 |

---

This completes the detailed reference for the "Send daily weather updates to a phone number via Plivo" n8n workflow.