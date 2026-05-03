Monitor Website Uptime with Gmail Alerts

https://n8nworkflows.xyz/workflows/monitor-website-uptime-with-gmail-alerts-5887


# Monitor Website Uptime with Gmail Alerts

### 1. Workflow Overview

This workflow monitors the uptime status of multiple websites at scheduled intervals and sends an email alert via Gmail if any website is detected as down. It is designed for website administrators or operators who want proactive notifications about website downtime to address issues before customers encounter them.

The workflow consists of the following logical blocks:

- **1.1 Scheduling and Configuration:** Periodically triggers the workflow and sets up the target websites and alert parameters.
- **1.2 Website URL Preparation:** Parses and prepares the list of websites to test.
- **1.3 First Uptime Test:** Performs the initial HTTP request checks for each website.
- **1.4 First Status Evaluation:** Determines if websites are up or down based on HTTP response codes.
- **1.5 Retry on Failure:** For websites initially detected as down, waits a configurable period and performs a second HTTP request test.
- **1.6 Final Status Evaluation:** Reassesses website status after the retry.
- **1.7 Alert Notification:** Sends a Gmail email alert if a website is confirmed down after the retry.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Configuration

- **Overview:**  
  This block triggers the workflow on an hourly schedule and defines the list of websites to monitor along with alert email and retry wait time configurations.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Config

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution every hour.  
    - Configuration: Set to trigger every 1 hour.  
    - Inputs: None (trigger node)  
    - Outputs: Sends execution signal to Config node  
    - Edge Cases: Missed executions if n8n server is down; no failover logic.

  - **Config**  
    - Type: Set  
    - Role: Stores configuration data including websites, retry wait time, and alert email.  
    - Configuration:  
      - `Websites` is a multi-line string with URLs separated by new lines.  
      - `wait_secs` is a number (default 5 seconds) representing wait before retry.  
      - `alert_email` is the recipient email for alerts.  
    - Inputs: From Schedule Trigger  
    - Outputs: Passes configuration data downstream  
    - Edge Cases: Invalid URLs or empty website list not handled explicitly.

---

#### 1.2 Website URL Preparation

- **Overview:**  
  Converts the multi-line string of websites into an array, then splits the array to process each website individually.

- **Nodes Involved:**  
  - Websites to Array  
  - Split Out

- **Node Details:**

  - **Websites to Array**  
    - Type: Set  
    - Role: Transforms the multiline `Websites` string into an array by splitting on new lines and filtering out empty lines.  
    - Key Expression:  
      ```javascript
      $json.Websites.split('\n').filter((url) => url)
      ```  
    - Inputs: From Config node  
    - Outputs: Passes array of URLs downstream  
    - Edge Cases: Extra whitespace or malformed URLs not sanitized here.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of websites into individual items for parallel processing.  
    - Parameters: Splits on field `Websites`  
    - Inputs: From Websites to Array  
    - Outputs: One output per website for parallel testing  
    - Edge Cases: Empty arrays produce no outputs; no error handling.

---

#### 1.3 First Uptime Test

- **Overview:**  
  Performs HTTP GET requests to each website to check if the site is reachable and returns a valid status code.

- **Nodes Involved:**  
  - Perform Site Test  
  - Merge Error & Success

- **Node Details:**

  - **Perform Site Test**  
    - Type: HTTP Request  
    - Role: Sends HTTP GET requests to each website URL.  
    - Configuration:  
      - URL set dynamically from each website item.  
      - Response set to full response (headers and status).  
      - `neverError` flag true to prevent node failure on HTTP errors.  
      - On error: continue with error output branch.  
    - Inputs: From Split Out  
    - Outputs: Two outputs — success and error (due to `continueErrorOutput`)  
    - Edge Cases: Network timeouts, DNS failures, non-200 status codes handled gracefully by continuing.

  - **Merge Error & Success**  
    - Type: Merge  
    - Role: Merges success and error outputs back into a single stream.  
    - Inputs: Both success and error outputs from Perform Site Test  
    - Outputs: Single unified stream downstream  
    - Edge Cases: None; ensures all test results continue processing.

---

#### 1.4 First Status Evaluation

- **Overview:**  
  Evaluates HTTP response status codes to determine if the website is up (status code ≤ 400) or down.

- **Nodes Involved:**  
  - Calculate Status  
  - Status Router

- **Node Details:**

  - **Calculate Status**  
    - Type: Set  
    - Role: Extracts key data from HTTP response for status evaluation.  
    - Assigns:  
      - `date` from HTTP response header `date`  
      - `website` URL from original request  
      - `IS_UP` boolean flag true if status code ≤ 400, else false  
    - Inputs: From Merge Error & Success  
    - Outputs: Passes enriched data downstream  
    - Edge Cases: Missing headers or malformed status codes could cause expression errors.

  - **Status Router**  
    - Type: Switch  
    - Role: Routes items based on `IS_UP` boolean.  
    - Outputs:  
      - `UP` for true (website up)  
      - `DOWN` for false (website down)  
    - Inputs: From Calculate Status  
    - Outputs:  
      - `UP` output discards item (no further processing here)  
      - `DOWN` output triggers retry block (Wait before Second Attempt)  
    - Edge Cases: Unexpected or missing `IS_UP` values may route incorrectly.

---

#### 1.5 Retry on Failure

- **Overview:**  
  For websites flagged as down, waits a configurable number of seconds and performs a second HTTP test to confirm downtime.

- **Nodes Involved:**  
  - Wait before Second Attempt  
  - Perform Site Test1  
  - Merge Error & Success1

- **Node Details:**

  - **Wait before Second Attempt**  
    - Type: Wait  
    - Role: Pauses workflow for `wait_secs` seconds before retry test.  
    - Configured dynamically from Config node parameter `wait_secs` (default 5 seconds).  
    - Inputs: From Status Router `DOWN` output  
    - Outputs: Passes to second HTTP test  
    - Edge Cases: Long waits delay alerting; short waits may cause false negatives.

  - **Perform Site Test1**  
    - Type: HTTP Request  
    - Role: Repeats the HTTP GET request to the website after waiting.  
    - Configuration is identical to Perform Site Test.  
    - Inputs: From Wait node  
    - Outputs: success and error outputs for merging  
    - Edge Cases: Same as first test.

  - **Merge Error & Success1**  
    - Type: Merge  
    - Role: Merges success and error outputs of retry HTTP request.  
    - Inputs: From Perform Site Test1 outputs  
    - Outputs: Passes unified stream downstream  
    - Edge Cases: None.

---

#### 1.6 Final Status Evaluation

- **Overview:**  
  After retry, reassesses the website status and routes for alerting if still down.

- **Nodes Involved:**  
  - Calculate Status1  
  - Status Router1

- **Node Details:**

  - **Calculate Status1**  
    - Type: Set  
    - Role: Same as Calculate Status, extracts date, website, and computes `IS_UP` for retry results.  
    - Inputs: From Merge Error & Success1  
    - Outputs: Passes enriched data downstream  
    - Edge Cases: Same as Calculate Status.

  - **Status Router1**  
    - Type: Switch  
    - Role: Routes retry results based on `IS_UP`.  
    - Outputs:  
      - `UP` output discards item (site recovered)  
      - `DOWN` output triggers email alert  
    - Inputs: From Calculate Status1  
    - Edge Cases: Incorrect routing if `IS_UP` missing or malformed.

---

#### 1.7 Alert Notification

- **Overview:**  
  Sends an email alert via Gmail OAuth2 if a website is confirmed down after retry.

- **Nodes Involved:**  
  - Send Email Alert

- **Node Details:**

  - **Send Email Alert**  
    - Type: Gmail  
    - Role: Sends a plain text email alert notifying that the website is down.  
    - Configuration:  
      - Recipient email dynamically from Config node parameter `alert_email`  
      - Subject: includes website URL and downtime notification  
      - Message body: includes date and website URL indicating downtime  
      - Sender name: "n8n uptime"  
      - OAuth2 Gmail credentials configured (named "Jim Halpert")  
    - Inputs: From Status Router1 `DOWN` output  
    - Outputs: Terminal; no further nodes  
    - Edge Cases: Authentication failures, quota limits, or invalid email address could cause email sending failures.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                       | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                        |
|-------------------------|--------------------|------------------------------------|------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger   | Triggers workflow hourly            | None                   | Config                   | ## 👇👇 Edit schedule                                                                                                               |
| Config                  | Set                | Holds website URLs, alert email, wait_secs | Schedule Trigger       | Websites to Array         | ## 👇👇 Add your websites here                                                                                                      |
| Websites to Array       | Set                | Converts websites string to array   | Config                  | Split Out                | ## 👇👇 Add your websites here                                                                                                      |
| Split Out               | Split Out          | Splits websites array into items    | Websites to Array        | Perform Site Test         |                                                                                                                                   |
| Perform Site Test       | HTTP Request       | Performs first HTTP test on sites   | Split Out                | Merge Error & Success     |                                                                                                                                   |
| Merge Error & Success   | Merge              | Merges success and error outputs    | Perform Site Test        | Calculate Status          |                                                                                                                                   |
| Calculate Status        | Set                | Evaluates first test results        | Merge Error & Success    | Status Router             |                                                                                                                                   |
| Status Router           | Switch             | Routes UP/DOWN from first test      | Calculate Status         | Wait before Second Attempt (DOWN), none (UP) |                                                                                                                                   |
| Wait before Second Attempt | Wait             | Waits before retrying                | Status Router (DOWN)     | Perform Site Test1        |                                                                                                                                   |
| Perform Site Test1      | HTTP Request       | Performs second HTTP test (retry)   | Wait before Second Attempt | Merge Error & Success1   |                                                                                                                                   |
| Merge Error & Success1  | Merge              | Merges retry test success and error | Perform Site Test1       | Calculate Status1         |                                                                                                                                   |
| Calculate Status1       | Set                | Evaluates retry test results         | Merge Error & Success1   | Status Router1            |                                                                                                                                   |
| Status Router1          | Switch             | Routes UP/DOWN from retry test       | Calculate Status1        | Send Email Alert (DOWN), none (UP) |                                                                                                                                   |
| Send Email Alert        | Gmail              | Sends email alert if site down       | Status Router1 (DOWN)    | None                     |                                                                                                                                   |
| Sticky Note             | Sticky Note        | Workflow description and motivation  | None                    | None                     | # Get Notified if website is down\n\nIf you manage a website, this is a must have for you!\n\nThis workflow will help you quickly identify if the website stopped working, before your customers do. |
| Sticky Note1            | Sticky Note        | Workflow description and motivation  | None                    | None                     | # Get Notified if website is down\n\nIf you manage a website, this is a must have for you!\n\nThis workflow will help you quickly identify if the website stopped working, before your customers do. |
| Sticky Note2            | Sticky Note        | Instructions for setup                | None                    | None                     | # Instructions\n\n1 - Add Gmail credentials (one-click google login)\n2 - Click on *Config* node and add your website URLs to *Websites* (you can add multiple!)\n3 - Change the alert email address + possibly the "wait_secs" for the double check mechanism. |
| Sticky Note3            | Sticky Note        | Instruction to add websites          | None                    | None                     | ## 👇👇 Add your websites here                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**  
   - Type: Schedule Trigger  
   - Set to run every 1 hour (interval: hours = 1)  
   - No input connections

2. **Create `Config` node:**  
   - Type: Set  
   - Connect input from Schedule Trigger  
   - Add fields:  
     - `Websites` (string, multiline): List your URLs separated by new lines, e.g.  
       ```
       https://smoothwork.ai
       https://bad.url
       https://smoothwork.ai/notfound
       ```  
     - `wait_secs` (number): default value 5  
     - `alert_email` (string): your alert recipient email  
   - No output connections yet

3. **Create `Websites to Array` node:**  
   - Type: Set  
   - Connect input from Config  
   - Add field `Websites` (type: array) with expression:  
     ```javascript
     {{$json.Websites.split('\n').filter(url => url)}}
     ```  
   - Output passes an array of URLs

4. **Create `Split Out` node:**  
   - Type: Split Out  
   - Connect input from Websites to Array  
   - Field to split out: `Websites`  
   - Outputs one item per website

5. **Create `Perform Site Test` node:**  
   - Type: HTTP Request  
   - Connect input from Split Out  
   - Set URL to expression: `{{$json.Websites}}`  
   - HTTP Method: GET  
   - Response: Full response with headers  
   - Set “Never Error” to true to prevent failures on HTTP error codes  
   - On error: continue with error output  
   - Outputs two branches: success and error

6. **Create `Merge Error & Success` node:**  
   - Type: Merge  
   - Connect both success and error outputs from Perform Site Test  
   - Merges back into one stream

7. **Create `Calculate Status` node:**  
   - Type: Set  
   - Connect input from Merge Error & Success  
   - Assign fields:  
     - `date`: `{{$json.headers.date}}`  
     - `website`: `{{$('Split Out').item.json.Websites}}`  
     - `IS_UP`: `{{$json.statusCode <= 400}}` (boolean)  
   - Outputs enriched data

8. **Create `Status Router` node:**  
   - Type: Switch  
   - Connect input from Calculate Status  
   - Add two outputs:  
     - `UP`: condition `{{$json.IS_UP === true}}`  
     - `DOWN`: condition `{{$json.IS_UP === false}}`  
   - `UP` output has no further nodes (end)  
   - `DOWN` output connects to Wait node

9. **Create `Wait before Second Attempt` node:**  
   - Type: Wait  
   - Connect input from Status Router `DOWN` output  
   - Set amount: expression `{{$('Config').item.json.wait_secs}}` seconds

10. **Create `Perform Site Test1` node:**  
    - Type: HTTP Request  
    - Connect input from Wait node  
    - Same configuration as Perform Site Test (dynamic URL, never error, full response)  
    - Outputs two branches: success and error

11. **Create `Merge Error & Success1` node:**  
    - Type: Merge  
    - Connect both outputs from Perform Site Test1  
    - Merge back into single stream

12. **Create `Calculate Status1` node:**  
    - Type: Set  
    - Connect input from Merge Error & Success1  
    - Assign fields same as Calculate Status:  
      - `date`: `{{$json.headers.date}}`  
      - `website`: `{{$('Split Out').item.json.Websites}}`  
      - `IS_UP`: `{{$json.statusCode <= 400}}` (boolean)

13. **Create `Status Router1` node:**  
    - Type: Switch  
    - Connect input from Calculate Status1  
    - Add two outputs:  
      - `UP`: condition `{{$json.IS_UP === true}}` (no further nodes)  
      - `DOWN`: condition `{{$json.IS_UP === false}}` connects to Send Email Alert

14. **Create `Send Email Alert` node:**  
    - Type: Gmail  
    - Connect input from Status Router1 `DOWN` output  
    - Set `Send To`: expression `{{$('Config').item.json.alert_email}}`  
    - Subject: `n8n uptime: {{$json.website}} is DOWN.`  
    - Message:  
      ```
      From: n8n uptime
      Date: {{$json.date}}

      {{$json.website}} is DOWN.
      ```  
    - Sender Name: `n8n uptime`  
    - Email Type: Text  
    - Credential: Use Gmail OAuth2 credentials (ensure OAuth2 is configured for Gmail API)

15. **Add Sticky Notes:**  
    - Add explanatory sticky notes on schedule, configuration, and workflow purpose as per original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow will help you quickly identify if the website stopped working, before your customers do. | Motivational note in sticky note                 |
| Instructions: 1 - Add Gmail credentials (OAuth2) 2 - Add your websites in Config node 3 - Set alert email and wait_secs | Setup instructions in sticky note                |
| Use OAuth2 Gmail credentials for sending alerts to avoid authentication errors or token expiration. | Important for reliable email alerts              |
| HTTP Request nodes are set with “Never Error” to prevent workflow failure on site errors.       | Ensures workflow continues even if site down    |
| The retry mechanism with a wait time reduces false alarms from transient errors.                 | Helps reliability of alerts                       |

---

**Disclaimer:** The provided text exclusively comes from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and includes no illegal, offensive, or protected elements. All data handled is legal and public.