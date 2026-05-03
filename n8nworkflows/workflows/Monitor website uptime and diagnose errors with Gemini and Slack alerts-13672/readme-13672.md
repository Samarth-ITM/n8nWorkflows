Monitor website uptime and diagnose errors with Gemini and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-website-uptime-and-diagnose-errors-with-gemini-and-slack-alerts-13672


# Monitor website uptime and diagnose errors with Gemini and Slack alerts

# Workflow Reference: Website Uptime Monitor & AI Diagnosis

This document provides a technical breakdown of the n8n workflow designed to monitor website availability, diagnose errors using Google Gemini AI, and manage notifications via Slack, Google Sheets, and Gmail.

---

### 1. Workflow Overview

The workflow automates the process of checking a website's health at regular intervals. It distinguishes between healthy states, slow performance, and critical downtime. When a failure is detected, it leverages Large Language Models (LLM) to provide a technical diagnosis, ensuring that alerts are actionable rather than just status codes.

**Functional Logic Blocks:**
*   **1.1 Health Check (Input & Evaluation):** Triggers the process, fetches the website data, and applies logic to determine the status (Up, Down, Degraded, or Slow).
*   **1.2 AI Diagnosis:** If the site is down or degraded, an AI model analyzes the technical error to suggest root causes.
*   **1.3 Logging & Notifications:** Records every check in a database (Google Sheets) and pushes real-time alerts to Slack or summary reports to Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Health Check
**Overview:** This block performs the raw HTTP request and categorizes the result based on response codes and latency.

*   **Nodes Involved:** `Check Every 5 Minutes`, `Check Website`, `Evaluate Response`.
*   **Node Details:**
    *   **Check Every 5 Minutes (Schedule Trigger):** Fires every 5 minutes.
    *   **Check Website (HTTP Request):** Performs a GET request to the target URL. Configuration includes `Full Response` enabled to capture headers and status codes, and `Continue On Fail` to ensure the workflow doesn't stop if the site is down.
    *   **Evaluate Response (Code):** A JavaScript snippet that calculates the state.
        *   *Threshold:* 3000ms is considered "slow".
        *   *Logic:* Sets `needsAlert` to true if the status is `down` (5xx errors or connection failures) or `degraded` (4xx errors).
        *   *Output:* A unified object containing `url`, `statusCode`, `responseTime`, `status`, and `issue`.

#### 2.2 AI Diagnosis
**Overview:** Triggered only when a problem is detected, this block uses AI to interpret technical error messages for human operators.

*   **Nodes Involved:** `Site Down?`, `Diagnose Error`, `Gemini Chat Model`.
*   **Node Details:**
    *   **Site Down? (If):** Filters incoming data. If `needsAlert` is true, it routes to the AI; otherwise, it skips straight to logging.
    *   **Diagnose Error (Chain LLM):** A LangChain-powered node that sends a prompt to Gemini. It passes the URL, Status Code, and Error Message. It instructs the AI to return a specific JSON structure (diagnosis, severity, likely cause).
    *   **Gemini Chat Model (Google Gemini):** Provides the intelligence for the diagnosis. Uses `gemini-1.5-flash` with a low temperature (0.1) for consistent, factual technical analysis.

#### 2.3 Logging and Notifications
**Overview:** Ensures data persistence and informs stakeholders of the site's status.

*   **Nodes Involved:** `Log to Uptime Sheet`, `Alert Slack`, `Send Daily Report`.
*   **Node Details:**
    *   **Alert Slack (Slack):** Sends a formatted message to a channel. It includes the AI's diagnosis text and the specific error detected by the system.
    *   **Log to Uptime Sheet (Google Sheets):** Appends a row to a spreadsheet for long-term uptime tracking.
    *   **Send Daily Report (Gmail):** Sends an email containing the current check's details. (Note: In a production environment, this node would typically follow an aggregator or a different schedule to send only once per day).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Check Every 5 Minutes** | Schedule Trigger | Workflow Entry | (None) | Check Website | Hits the target URL and evaluates response status and timing. |
| **Check Website** | HTTP Request | Data Acquisition | Check Every 5 Minutes | Evaluate Response | Hits the target URL and evaluates response status and timing. |
| **Evaluate Response** | Code | Logic Engine | Check Website | Site Down? | Hits the target URL and evaluates response status and timing. |
| **Site Down?** | If | Branching | Evaluate Response | Diagnose Error, Log to Uptime Sheet | AI error diagnosis / Gemini analyzes the error pattern. |
| **Diagnose Error** | Chain LLM | AI Processor | Site Down? | Alert Slack | Gemini analyzes the error pattern and suggests probable root causes. |
| **Gemini Chat Model** | Google Gemini | AI Model | (None) | Diagnose Error | Gemini analyzes the error pattern and suggests probable root causes. |
| **Alert Slack** | Slack | Critical Alerting | Diagnose Error | Log to Uptime Sheet | Logs every check to Sheets. Sends Slack alerts. |
| **Log to Uptime Sheet** | Google Sheets | Data Persistence | Site Down?, Alert Slack | Send Daily Report | Logs every check to Sheets. Sends Slack alerts. |
| **Send Daily Report** | Gmail | Reporting | Log to Uptime Sheet | (None) | Sends daily Gmail reports. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Add a **Schedule Trigger** node set to run every 5 minutes.
2.  **Request Setup:** Add an **HTTP Request** node. Set URL to your target site. Change "On Error" to `Continue Regular Output`. Enable "Full Response" in the options.
3.  **Logic Setup:** Add a **Code Node**. Use JavaScript to map `item.json.statusCode`. Create a variable `needsAlert` that is true if the status code is $\ge 400$ or if an error exists.
4.  **Branching:** Add an **If Node** to check if `needsAlert` is true.
5.  **AI Integration (True Path):**
    *   Add a **Basic LLM Chain** node connected to the "True" output.
    *   Attach a **Google Gemini Chat Model** node to the LLM Chain. Use model `gemini-1.5-flash`.
    *   Configure the prompt to accept the error details and output a diagnosis.
6.  **Slack Alerting:** Add a **Slack Node** after the LLM Chain to post the diagnosis.
7.  **Data Logging:** 
    *   Add a **Google Sheets Node** (Append Operation).
    *   Connect both the "False" path of the If node and the output of the Slack node to this node.
8.  **Email Notification:** Add a **Gmail Node** to send a summary of the current status.
9.  **Credentials:** Ensure OAuth2 credentials are set for Google (Sheets/Gmail) and Slack, and an API Key is provided for Gemini.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Steps** | 1. Set URL in HTTP node. 2. Connect Gemini API. 3. Create "Uptime Log" Sheet. 4. Connect Slack/Gmail. |
| **Customization** | Adjust response time threshold (default 3000ms) inside the "Evaluate Response" code node. |
| **Requirements** | Requires Google Gemini API key (Free tier available). |