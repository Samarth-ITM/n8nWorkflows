Get Real-time International Space Station Visibility Alerts with N2YO and Telegram

https://n8nworkflows.xyz/workflows/get-real-time-international-space-station-visibility-alerts-with-n2yo-and-telegram-9320


# Get Real-time International Space Station Visibility Alerts with N2YO and Telegram

### 1. Workflow Overview

This workflow monitors the International Space Station’s (ISS) real-time visibility status using the N2YO API and sends alerts via Telegram when the ISS is visible from a certain location. It is designed for users interested in receiving timely notifications about ISS flyovers, enabling them to observe or track the station in the sky.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Execution:** Periodically triggers the workflow to check ISS visibility.
- **1.2 Data Retrieval:** Fetches real-time ISS visibility data from the N2YO API.
- **1.3 Data Processing:** Converts raw API data into human-readable information.
- **1.4 Decision Making:** Determines whether the ISS is currently visible or not.
- **1.5 Notification Sending:** Sends a Telegram message alerting the user if the ISS is visible.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Execution

**Overview:**  
This block initiates the workflow on a defined schedule, ensuring that ISS visibility checks happen automatically at regular intervals.

**Nodes Involved:**  
- Schedule Trigger

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow periodically based on a configured time interval (e.g., every few minutes or hours).  
  - Configuration: Default schedule parameters (not explicitly shown), likely configured to run at a frequency suitable for near real-time monitoring.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers the HTTP Request node  
  - Potential Failures: Misconfigured schedule could cause too frequent or infrequent triggers; no authentication needed.  
  - Version: 1.2

---

#### 1.2 Data Retrieval

**Overview:**  
This block performs an HTTP request to the N2YO API to obtain the current visibility data of the ISS for a specific location.

**Nodes Involved:**  
- HTTP Request

**Node Details:**  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls the external N2YO API to fetch real-time ISS visibility data.  
  - Configuration:  
    - Request Method: GET (assumed)  
    - URL and query parameters: Configured to N2YO API endpoint for ISS visibility (exact URL and parameters not visible in JSON, but typically includes latitude, longitude, altitude, and API key).  
  - Inputs: Triggered by Schedule Trigger  
  - Outputs: Passes raw API response data to the Readable node  
  - Potential Failures:  
    - Network issues or timeouts  
    - API key invalid or missing  
    - API rate limits reached  
    - Malformed response or unexpected data structure  
  - Version: 4.2

---

#### 1.3 Data Processing

**Overview:**  
This block processes the raw JSON response from the N2YO API to extract meaningful, human-readable information about the ISS visibility status.

**Nodes Involved:**  
- Readable

**Node Details:**  

- **Readable**  
  - Type: Code (JavaScript)  
  - Role: Parses and transforms the raw API data into a user-friendly format.  
  - Configuration: Contains JavaScript code that likely:  
    - Parses the JSON response  
    - Extracts relevant fields such as visibility status, pass start time, duration, etc.  
    - Formats these fields into readable text messages  
  - Inputs: Receives raw API response from HTTP Request node  
  - Outputs: Passes formatted data to the If node for visibility evaluation  
  - Potential Failures:  
    - Parsing errors if API response changes  
    - Runtime errors in code expressions  
  - Version: 2

---

#### 1.4 Decision Making

**Overview:**  
This block evaluates the processed visibility data to decide whether to send a notification. It checks if the ISS is currently visible or about to be visible.

**Nodes Involved:**  
- If

**Node Details:**  

- **If**  
  - Type: If  
  - Role: Conditional branching based on ISS visibility status extracted in the previous block.  
  - Configuration: Expression checks a key variable (e.g., boolean or status string) indicating ISS visibility.  
  - Inputs: Receives formatted data from Readable node  
  - Outputs:  
    - True branch: Connected to Send a text message node (ISS visible)  
    - False branch: No further action  
  - Potential Failures:  
    - Expression failures if data is missing or malformed  
  - Version: 2.2

---

#### 1.5 Notification Sending

**Overview:**  
This block sends a Telegram message to the user alerting them that the ISS is currently visible.

**Nodes Involved:**  
- Send a text message

**Node Details:**  

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends a text notification via Telegram bot to user(s).  
  - Configuration:  
    - Bot token and chat ID configured via credentials (OAuth2 or Bot Token)  
    - Message text constructed from visibility data (probably passed from Readable or If node)  
  - Inputs: Triggered only if If node confirms ISS visibility (true branch)  
  - Outputs: None (terminal node)  
  - Potential Failures:  
    - Telegram API authentication errors  
    - Invalid chat ID or revoked bot permissions  
    - Message rate limits imposed by Telegram  
  - Version: 1.2

---

### 3. Summary Table

| Node Name          | Node Type         | Functional Role             | Input Node(s)     | Output Node(s)         | Sticky Note                      |
|--------------------|-------------------|----------------------------|-------------------|------------------------|---------------------------------|
| Schedule Trigger    | Schedule Trigger  | Initiate workflow on timer | None              | HTTP Request           |                                 |
| HTTP Request       | HTTP Request      | Fetch ISS visibility data  | Schedule Trigger  | Readable               |                                 |
| Readable           | Code              | Process and format data    | HTTP Request      | If                     |                                 |
| If                 | If                | Decide if ISS is visible   | Readable          | Send a text message     |                                 |
| Send a text message | Telegram          | Send Telegram notification | If (true branch)  | None                   |                                 |
| Sticky Note        | Sticky Note       | (Empty)                    | None              | None                   |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set the frequency (e.g., every 5 minutes) to periodically check ISS visibility.

2. **Add an HTTP Request node:**  
   - Connect it to the Schedule Trigger node.  
   - Configure it as a GET request to the N2YO ISS visibility API endpoint.  
   - Include required query parameters such as:  
     - Latitude, longitude, altitude of the observation point.  
     - API key for N2YO service (set up via credentials).  
   - Ensure the response format is JSON.

3. **Add a Code node:**  
   - Connect it to the HTTP Request node.  
   - Insert JavaScript code to parse the JSON response:  
     - Extract visibility status and relevant details (e.g., pass time, duration).  
     - Format this information into a text message or flag indicating visibility.  
   - Handle possible null or error values gracefully.

4. **Add an If node:**  
   - Connect it to the Code node.  
   - Configure a condition expression to check if the ISS is visible based on the processed data (e.g., visibility flag == true).

5. **Add a Telegram node:**  
   - Connect the If node’s “true” output to this node.  
   - Configure Telegram credentials (Bot token or OAuth2).  
   - Set the target chat ID for notifications.  
   - Set the message text using the output from the Code node (e.g., “The ISS is currently visible!” with additional info).

6. **Optionally add Sticky Notes:**  
   - For documentation or reminders within the workflow editor.

7. **Test the workflow:**  
   - Run manually or wait for the schedule trigger to verify correct API calls and Telegram message delivery.  
   - Monitor for errors such as API rate limits, token expiration, or malformed data.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                           |
|----------------------------------------------------------------------------------------------|------------------------------------------|
| N2YO API documentation: https://www.n2yo.com/api/                                          | Reference for API endpoints and parameters |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                           | Guide for bot setup and message sending  |
| Ensure API keys and Telegram bot tokens are securely stored and not exposed publicly         | Security best practice                     |
| Consider adding error handling nodes or alerts for API failures or message send failures     | Workflow robustness improvement           |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow constructed with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.