UptimeRobot Alerts to Telegram with Visual Verification

https://n8nworkflows.xyz/workflows/uptimerobot-alerts-to-telegram-with-visual-verification-5780


# UptimeRobot Alerts to Telegram with Visual Verification

### 1. Workflow Overview

This workflow automates the processing of UptimeRobot alert emails received via Gmail and sends corresponding status notifications to a Telegram chat, optionally including a visual screenshot of the monitored website. It is designed for users who want real-time status updates with visual verification of monitored websites’ uptime directly in Telegram.

The workflow consists of the following logical blocks:

- **1.1 Input Reception (Email Trigger):** Detects incoming UptimeRobot alert emails in Gmail.
- **1.2 Configuration Setup:** Defines workflow-level parameters such as screenshot preferences and ScreenshotMachine API credentials.
- **1.3 Alert Parsing:** Extracts monitor URL and ID from the email content.
- **1.4 UptimeRobot API Query:** Retrieves detailed monitor status and logs using the extracted monitor ID.
- **1.5 Status Extraction:** Maps numeric status codes to human-readable strings and extracts last status change details.
- **1.6 Conditional Screenshot Generation:** Decides whether to generate a screenshot based on configuration.
- **1.7 Screenshot Generation:** Uses ScreenshotMachine API to capture a screenshot of the monitored URL.
- **1.8 Notification Dispatch:** Sends formatted status updates via Telegram, either as text or photo messages depending on screenshot availability.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception (Email Trigger)

- **Overview:** Listens for new emails from UptimeRobot alert sender in Gmail, polling every 5 minutes.
- **Nodes Involved:** `Gmail Trigger`
- **Node Details:**
  - Type: Gmail Trigger node — triggers workflow on incoming emails.
  - Configuration: Filters emails by sender `alert@uptimerobot.com`, polls every 5 minutes.
  - Input: Incoming Gmail email data.
  - Output: Email metadata and body content.
  - Edge Cases: Possible failures include Gmail OAuth token expiration, network issues, misconfigured sender filter leading to missed alerts.
  - Sticky Note Reference: Detailed explanation on setup, filtering, and polling.

#### Block 1.2: Configuration Setup

- **Overview:** Sets parameters controlling screenshot generation and ScreenshotMachine API details.
- **Nodes Involved:** `Conf`
- **Node Details:**
  - Type: Set node — assigns fixed configuration variables.
  - Configuration:
    - `take_screenshot`: boolean flag to enable/disable screenshots.
    - `screenshotmachine_secret`: API secret key for ScreenshotMachine.
    - `screenshotmachine_device`: device type for screenshot (`desktop`).
    - `screenshotmachine_dimension`: screenshot resolution (`1366xfull`).
    - `screenshotmachine_format`: image format (`png`).
  - Input: Triggered by Gmail Trigger output.
  - Output: JSON object with configuration fields.
  - Edge Cases: Misconfiguration leads to incorrect or failed screenshot requests.
  - Sticky Note Reference: Explains meaning and use of each config parameter.

#### Block 1.3: Alert Parsing

- **Overview:** Extracts monitor dashboard URL and 9-digit monitor ID from email text using regex.
- **Nodes Involved:** `Extract ID and URL`
- **Node Details:**
  - Type: Code node (JavaScript) — parses email text.
  - Configuration:
    - Uses regex `/\[(https:\/\/dashboard\.uptimerobot\.com\/monitors\/(\d{9})\?[^\]]*)\]/` to extract monitor URL and ID inside square brackets.
    - Adds `monitorUrl` and `monitorId` to JSON.
  - Input: Output from `Conf` node (which follows Gmail Trigger).
  - Output: JSON enriched with extracted URL and ID.
  - Edge Cases: Regex mismatch if email format changes, missing URL or ID results in downstream failures.
  - Sticky Note Reference: Contains regex explanation and troubleshooting tips.

#### Block 1.4: UptimeRobot API Query

- **Overview:** Queries UptimeRobot API to get detailed monitor information including logs using extracted monitor ID.
- **Nodes Involved:** `Get many monitors`
- **Node Details:**
  - Type: UptimeRobot node — fetches monitor details.
  - Configuration:
    - Filters monitor by ID extracted previously.
    - Requests logs and alert contact info.
  - Input: From `Extract ID and URL`.
  - Output: Monitor details JSON including logs and status codes.
  - Edge Cases: API key issues, permission errors, invalid monitor ID, API downtime.
  - Sticky Note Reference: Setup tips for UptimeRobot credentials and alerts.

#### Block 1.5: Status Extraction

- **Overview:** Converts numeric status codes to readable text, extracts last status change timestamp and reason.
- **Nodes Involved:** `Extract Status Details`
- **Node Details:**
  - Type: Code node (JavaScript) — interprets API response.
  - Configuration:
    - Maps status codes (0,1,2,8,9) to descriptive strings.
    - Extracts last status change datetime and reason from logs.
    - Defaults to "unknown" if missing.
  - Input: From `Get many monitors`.
  - Output: JSON with fields `monitorStatus`, `lastStatusChange`, `lastStatusChangeReason`.
  - On error: Continues workflow (does not fail).
  - Edge Cases: Missing or malformed logs, unexpected status codes.
  
#### Block 1.6: Conditional Screenshot Generation

- **Overview:** Checks if screenshot generation is enabled and branches accordingly.
- **Nodes Involved:** `If Screenshot Required`
- **Node Details:**
  - Type: If node — evaluates boolean expression.
  - Configuration:
    - Condition: `take_screenshot` configuration variable must be true.
  - Input: From `Extract Status Details`.
  - Outputs:
    - True branch: proceeds to screenshot generation.
    - False branch: skips screenshot, sends text message only.
  - Edge Cases: Misconfiguration causing wrong branch taken.

#### Block 1.7: Screenshot Generation

- **Overview:** Generates a SHA1 hash for authentication and requests a screenshot from ScreenshotMachine API.
- **Nodes Involved:** `Screenshotmachine-secret`, `HTTP Request`
- **Node Details:**
  - `Screenshotmachine-secret`:
    - Type: Crypto node — computes SHA1 hash of URL + secret.
    - Input: URL from `Extract Status Details` and secret from `Conf`.
    - Output: Hash string stored in `hash`.
    - Edge Cases: Incorrect secret causes invalid hash.
  - `HTTP Request`:
    - Type: HTTP Request node — calls ScreenshotMachine API.
    - Configuration:
      - Query parameters include URL, dimension, device, format, and hash.
      - Auth via generic HTTP query auth with customer key.
    - Input: From `Screenshotmachine-secret`.
    - Output: Binary image data of screenshot.
    - On error: Continues workflow.
    - Edge Cases: API limits exceeded, network errors, invalid auth.
  - Sticky Note Reference: Setup instructions for ScreenshotMachine credentials and plan limits.

#### Block 1.8: Notification Dispatch

- **Overview:** Sends Telegram notifications with formatted status updates, optionally including screenshot image.
- **Nodes Involved:** `Send a photo message`, `Send a text message`
- **Node Details:**
  - `Send a photo message`:
    - Type: Telegram node — sends photo message with caption.
    - Configuration:
      - Sends binary image from `HTTP Request`.
      - Caption includes monitor name, status emoji, last changed timestamp, reason, and dashboard link.
      - Uses Telegram bot credentials.
    - Input: From `HTTP Request`.
    - Output: Telegram message confirmation.
    - Edge Cases: Telegram API rate limits, invalid bot token, chat ID errors.
  - `Send a text message`:
    - Type: Telegram node — sends plain text message.
    - Configuration:
      - Text message template similar to photo caption.
      - Uses Telegram bot credentials.
      - Chat ID configured.
    - Input:
      - From `If Screenshot Required` false branch (no screenshot).
      - Also from `HTTP Request` error handling fallback.
    - Edge Cases: Same as photo message.
  - Sticky Note Reference: Telegram bot setup steps, chat ID retrieval links, and message format tips.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                          | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                                          |
|--------------------------|--------------------|----------------------------------------|--------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger            | Gmail Trigger      | Trigger workflow on new UptimeRobot emails | (Start)                  | Conf                          | #1 Gmail Trigger: Setup, polling, sender filter details                                                             |
| Conf                     | Set                | Workflow configuration parameters       | Gmail Trigger            | Extract ID and URL             | #2 Workflow settings: screenshot and ScreenshotMachine parameters                                                   |
| Extract ID and URL       | Code               | Extract monitor URL and ID from email   | Conf                     | Get many monitors             | # Extract ID and URL: Regex explanation and troubleshooting                                                         |
| Get many monitors        | UptimeRobot        | Query monitor details from API           | Extract ID and URL       | Extract Status Details         | #3 UptimeRobot settings: API key and alert integration tips                                                         |
| Extract Status Details   | Code               | Map status codes and extract last status change info | Get many monitors        | If Screenshot Required         |                                                                                                                      |
| If Screenshot Required   | If                 | Branch based on screenshot config flag  | Extract Status Details   | Screenshotmachine-secret (true), Send a text message (false) |                                                                                                                      |
| Screenshotmachine-secret | Crypto             | Generate hash for ScreenshotMachine auth | If Screenshot Required (true) | HTTP Request                  | #4 ScreenshotMachine setup: credential config and plan limits                                                       |
| HTTP Request             | HTTP Request       | Call ScreenshotMachine API to get screenshot | Screenshotmachine-secret | Send a photo message, Send a text message (on error) |                                                                                                                      |
| Send a photo message     | Telegram            | Send status update with screenshot      | HTTP Request             | (End)                         | #5 Telegram setup: bot creation, chat ID, send text and photo message info                                          |
| Send a text message      | Telegram            | Send status update as text only          | If Screenshot Required (false), HTTP Request (error) | (End)                         |                                                                                                                      |
| Sticky Note (multiple)   | Sticky Note         | Documentation and instructions           | N/A                      | N/A                           | Various notes covering Gmail Trigger, Config, UptimeRobot, ScreenshotMachine, Telegram setup, with links and tips   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**
   - Set credentials for Gmail OAuth2.
   - Set filter sender to `alert@uptimerobot.com`.
   - Configure polling interval to every 5 minutes.
   
2. **Create Set node named "Conf":**
   - Add boolean `take_screenshot`, default `true`.
   - Add string `screenshotmachine_secret`, set to your ScreenshotMachine secret key.
   - Add string `screenshotmachine_device`, default `desktop`.
   - Add string `screenshotmachine_dimension`, default `1366xfull`.
   - Add string `screenshotmachine_format`, default `png`.
   - Connect output of Gmail Trigger to this node.

3. **Create Code node "Extract ID and URL":**
   - Use JavaScript to parse incoming email text.
   - Extract `monitorUrl` and `monitorId` using regex matching UptimeRobot monitor URLs.
   - Input: Connect from "Conf".
   - Output: Pass extracted data downstream.

4. **Create UptimeRobot node "Get many monitors":**
   - Set resource to "monitor".
   - Set filters to include logs, alert contacts, and monitor IDs matching extracted `monitorId`.
   - Use UptimeRobot credentials with API key and read permissions.
   - Connect input from "Extract ID and URL".

5. **Create Code node "Extract Status Details":**
   - Map numeric status codes to descriptive strings.
   - Extract last status change timestamp and reason from logs.
   - Input: Connect from "Get many monitors".
   - Set onError to continue execution.

6. **Create If node "If Screenshot Required":**
   - Condition: Check if `take_screenshot` from "Conf" is true.
   - Connect input from "Extract Status Details".

7. **Create Crypto node "Screenshotmachine-secret":**
   - Compute SHA1 hash of concatenated URL + secret.
   - Input: Connect from true branch of "If Screenshot Required".
   - Output: Pass hash to next node.

8. **Create HTTP Request node:**
   - Set URL to `https://api.screenshotmachine.com`.
   - Configure query parameters: url, dimension, device, format, and hash.
   - Use Generic HTTP Query Auth with ScreenshotMachine customer key credential.
   - Input: Connect from "Screenshotmachine-secret".
   - Set onError to continue.

9. **Create Telegram node "Send a photo message":**
   - Operation: sendPhoto.
   - Chat ID: your Telegram chat ID.
   - File: Use binary data from HTTP Request node.
   - Caption: Use expressions to include monitor name, status with emoji, last change time, reason, and monitor dashboard link.
   - Use Telegram API credentials with bot token.
   - Input: Connect from HTTP Request main output.

10. **Create Telegram node "Send a text message":**
    - Operation: sendMessage (text only).
    - Chat ID: your Telegram chat ID.
    - Text: Format similar to photo caption but plain text.
    - Use same Telegram credentials.
    - Input: Connect from false branch of "If Screenshot Required" and error branch of HTTP Request.

11. **Connect all nodes as per above and deploy.**

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Gmail Trigger node filters emails from `alert@uptimerobot.com`, polling every 5 minutes.                        | #1 Gmail Trigger Sticky Note                                                                           |
| Configuration variables centralize control of screenshot usage and API credentials for ScreenshotMachine.       | #2 Workflow Settings Sticky Note                                                                       |
| UptimeRobot node requires API key with Monitor Read permission. Set alert emails in UptimeRobot dashboard.      | #3 UptimeRobot Settings Sticky Note, https://docs.n8n.io/integrations/builtin/credentials/uptimerobot/ |
| ScreenshotMachine credentials require query auth with customer key; free plan has limits on screenshots/month. | #4 ScreenshotMachine Sticky Note, https://www.screenshotmachine.com/                                  |
| Telegram bot setup requires creating bot via @BotFather and obtaining chat ID via @getidsbot.                   | #5 Telegram Setup Sticky Note, https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/ |

---

This completes the detailed reference for the "UptimeRobot Alerts to Telegram with Visual Verification" n8n workflow, enabling understanding, reproduction, and troubleshooting.