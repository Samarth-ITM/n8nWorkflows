Health Check Websites with Google Sheets & Telegram Alerts

https://n8nworkflows.xyz/workflows/health-check-websites-with-google-sheets---telegram-alerts-3352


# Health Check Websites with Google Sheets & Telegram Alerts

### 1. Workflow Overview

This workflow, titled **Health Check Websites with Google Sheets & Telegram Alerts**, automates the monitoring of website availability by periodically checking a list of URLs stored in a Google Sheet. It is designed to run on a schedule, fetch URLs, perform HTTP requests to verify their health, and send notifications via Telegram if any URL check fails.

**Target Use Cases:**  
- Monitoring uptime and responsiveness of internal or external web services  
- Real-time alerting for service outages or failures  
- Easily adaptable to other notification platforms like Slack, Discord, Email, or Microsoft Teams

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow at regular intervals  
- **1.2 URL Retrieval:** Reads the list of URLs from a Google Sheet  
- **1.3 URL Health Check:** Sends HTTP requests to each URL and handles errors  
- **1.4 Notification Dispatch:** Sends alerts via Telegram when a URL check fails  
- **1.5 No Operation (Control Flow):** Separates success and failure paths after HTTP requests  

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow at a fixed interval (every X minutes), ensuring periodic health checks.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger every minute (interval field set to minutes with value 1)  
    - Input: None (starting point)  
    - Output: Triggers the next node "Fetch Urls"  
    - Version: 1.2  
    - Edge Cases: If the n8n instance is down or paused, scheduled triggers will not fire; ensure n8n is running continuously.  
    - Notes: Interval can be customized to any number of minutes as needed.

#### 1.2 URL Retrieval

- **Overview:**  
  Retrieves the list of URLs to check from a specified Google Sheet document and sheet tab.

- **Nodes Involved:**  
  - Fetch Urls

- **Node Details:**  
  - **Fetch Urls**  
    - Type: Google Sheets  
    - Configuration:  
      - Document ID: Points to a specific Google Sheet (ID: `17-tY9_wn-D2FV627Sx3-Z3abqFYvz794edej7es5J6w`)  
      - Sheet Name: Uses the first sheet tab (`gid=0`)  
      - Reads all rows, expecting a column named `URLS` containing URLs to check  
    - Input: Triggered by Schedule Trigger  
    - Output: Passes each row (each URL) to the next node "Check URL"  
    - Credentials: Uses OAuth2 credentials for Google Sheets access  
    - Version: 4.5  
    - Edge Cases:  
      - Authentication errors if OAuth2 token expires or is revoked  
      - Empty or missing `URLS` column leads to no URLs being checked  
      - Large sheets may cause delays or rate limiting by Google API

#### 1.3 URL Health Check

- **Overview:**  
  Sends HTTP GET requests to each URL retrieved to verify their availability. Continues workflow execution even if requests fail.

- **Nodes Involved:**  
  - Check URL  
  - No Operation, do nothing

- **Node Details:**  
  - **Check URL**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: Dynamically set to the value of the `URLS` field from the Google Sheet row (`={{ $json.URLS }}`)  
      - Method: Defaults to GET  
      - Options: Default (no special headers or authentication)  
      - On Error: Set to "continueErrorOutput" to allow workflow to continue on HTTP errors  
    - Input: Receives URL data from "Fetch Urls"  
    - Output: Two outputs:  
      - Main output (index 0): Successful HTTP responses  
      - Error output (index 1): Failed HTTP requests (e.g., non-2xx status codes, timeouts)  
    - Version: 4.2  
    - Edge Cases:  
      - Network timeouts or DNS failures  
      - HTTP errors like 404, 500, or connection refused  
      - Malformed URLs causing request failures  
      - Expression failures if `URLS` field is missing or invalid  
  - **No Operation, do nothing**  
    - Type: No Operation (NoOp)  
    - Role: Receives successful HTTP request outputs but performs no action, effectively ignoring successful checks  
    - Input: Connected to main output of "Check URL"  
    - Output: None  
    - Version: 1  
    - Edge Cases: None (passive node)

#### 1.4 Notification Dispatch

- **Overview:**  
  Sends a Telegram message alerting about failed URL checks, including the URL and error code.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**  
  - **Telegram**  
    - Type: Telegram node  
    - Configuration:  
      - Chat ID: `1548053076` (Telegram chat to send alerts)  
      - Text: Template message including URL and error code:  
        ```
        Health Check :  {{ $json.URLS }}

        {{ $json.error.code }}
        ```  
      - Additional Fields: Attribution disabled (no "sent via n8n" message)  
    - Input: Connected to error output of "Check URL" (failed HTTP requests)  
    - Output: None (end of workflow)  
    - Credentials: Uses Telegram API credentials (OAuth token for bot)  
    - Version: 1.2  
    - Edge Cases:  
      - Telegram API rate limits or downtime  
      - Invalid chat ID or revoked bot permissions  
      - Message formatting errors if variables are missing

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                  | Input Node(s)       | Output Node(s)               | Sticky Note                                                                                   |
|-------------------------|---------------------|---------------------------------|---------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Initiates workflow periodically | None                | Fetch Urls                  |                                                                                               |
| Fetch Urls              | Google Sheets       | Reads URLs from Google Sheet     | Schedule Trigger    | Check URL                   |                                                                                               |
| Check URL               | HTTP Request        | Checks URL health via HTTP GET   | Fetch Urls          | No Operation, do nothing (success), Telegram (failure) |                                                                                               |
| No Operation, do nothing| No Operation (NoOp) | Handles successful HTTP checks   | Check URL (main)    | None                        |                                                                                               |
| Telegram                | Telegram            | Sends alert messages on failure  | Check URL (error)   | None                        | Step 2: To use telegram, simply define chatid. You can replace with any notification service. |
| Sticky Note             | Sticky Note         | Instructional note               | None                | None                        | Step 1: Create a new google sheet with column A listing all URLs to check.                    |
| Sticky Note1            | Sticky Note         | Instructional note               | None                | None                        | Step 2: To use telegram, simply define chatid. You can replace with any notification service. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 1 minute (or desired frequency)  
   - No credentials needed  
   - Connect output to next node

2. **Create Google Sheets Node ("Fetch Urls")**  
   - Type: Google Sheets  
   - Credentials: Configure OAuth2 credentials for Google Sheets access  
   - Parameters:  
     - Document ID: Enter your Google Sheet ID containing URLs  
     - Sheet Name: Use the first sheet tab or specify by gid (e.g., `gid=0`)  
     - Read mode: List all rows  
   - Connect input from Schedule Trigger  
   - Connect output to next node

3. **Create HTTP Request Node ("Check URL")**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Set to expression `={{ $json.URLS }}` to dynamically use URL from Google Sheet  
     - Method: GET (default)  
     - On Error: Set to "Continue on Error" to allow workflow to proceed even if request fails  
   - Connect input from Google Sheets node  
   - Connect two outputs:  
     - Main output (success) to No Operation node  
     - Error output (failure) to Telegram node

4. **Create No Operation Node ("No Operation, do nothing")**  
   - Type: No Operation (NoOp)  
   - Connect input from main output of HTTP Request node  
   - No outputs (end of success path)

5. **Create Telegram Node ("Telegram")**  
   - Type: Telegram  
   - Credentials: Configure Telegram API credentials (bot token)  
   - Parameters:  
     - Chat ID: Enter your Telegram chat ID (e.g., `1548053076`)  
     - Text: Use template:  
       ```
       Health Check :  {{ $json.URLS }}

       {{ $json.error.code }}
       ```  
     - Additional Fields: Disable attribution if desired  
   - Connect input from error output of HTTP Request node  
   - No outputs (end of failure path)

6. **Optional: Add Sticky Notes for Documentation**  
   - Add notes explaining setup steps, e.g., creating Google Sheet with URLs, configuring Telegram chat ID

7. **Activate Workflow**  
   - Save and activate the workflow to start scheduled health checks

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create a Google Sheet with column A listing all URLs to check, starting with a header in A1.    | See Sticky Note: Step 1                                                                             |
| To use Telegram, define your chat ID and bot credentials; can replace Telegram with Slack, Email, Discord, or Teams. | See Sticky Note1: Step 2                                                                            |
| This workflow template is ideal for uptime monitoring without complex infrastructure setup.      | Workflow description                                                                                |
| Customize schedule interval, notification message formatting, or add retries and logging as needed. | Workflow customization tips                                                                         |

---

This documentation provides a detailed, stepwise understanding of the **Health Check Websites with Google Sheets & Telegram Alerts** workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the workflow effectively.