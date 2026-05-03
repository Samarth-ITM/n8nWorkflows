Website Downtime Monitoring with Smart Alerts via Telegram & Email

https://n8nworkflows.xyz/workflows/website-downtime-monitoring-with-smart-alerts-via-telegram---email-11763


# Website Downtime Monitoring with Smart Alerts via Telegram & Email

---

### 1. Workflow Overview

This workflow, titled **Website Downtime Monitoring with Smart Alerts via Telegram & Email**, is designed to periodically check the availability of a list of websites and send notifications via Telegram and email when any site is detected as down. It targets use cases such as uptime monitoring for web services, alerting IT teams or website owners promptly through multiple channels.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Configuration**: Defines the periodic trigger and website list configuration including alert contact info.
- **1.2 Input Preparation & Splitting**: Processes the configured websites into individual URLs for testing.
- **1.3 Website Availability Testing**: Performs HTTP requests to each website and collects response status data.
- **1.4 Status Calculation & Routing**: Determines if each website is up or down based on HTTP status codes and routes accordingly.
- **1.5 Notification Handling**: Sends alerts via Telegram immediately on downtime detection, and after a wait period sends email notifications.
- **1.6 Wait Mechanism**: Implements a delay before sending email alerts to avoid premature or duplicate notifications.
- **1.7 Auxiliary Notes & Credentials Guidance**: Provides sticky notes with instructions and credential setup information.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Configuration

- **Overview:** Periodically triggers the workflow and defines the list of websites to monitor along with alert contact details.
- **Nodes Involved:** Schedule Trigger, Config, Edit Fields

**Nodes:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow every minute (interval set to 1 minute).  
  - *Config:* Interval configured to trigger workflow every 1 minute.  
  - *Connections:* Outputs to Config node.  
  - *Failures:* Trigger failure rare; ensure system time is synchronized.

- **Config**  
  - *Type:* Set  
  - *Role:* Holds configuration variables including website URLs (multi-line string), Telegram chat ID, alert email, and wait time before sending emails.  
  - *Config:* Websites stored as newline-separated string; Telegram chat ID and email are placeholders to be replaced.  
  - *Connections:* Outputs to Edit Fields node.  
  - *Failures:* Misconfiguration could cause empty or invalid URLs or missing credentials.

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Processes raw websites string into an array by splitting on newlines and filtering out empty lines.  
  - *Config:* Uses JavaScript expression to split and clean website list.  
  - *Connections:* Outputs to Split Out node.  
  - *Failures:* Expression errors if input format is unexpected.

---

#### 2.2 Input Preparation & Splitting

- **Overview:** Splits the array of websites into individual items for separate HTTP tests.  
- **Nodes Involved:** Split Out

**Nodes:**

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Takes array of websites and outputs each as separate item for parallel processing.  
  - *Config:* Field to split out is "Websites" array.  
  - *Connections:* Outputs to Perform Site Test node.  
  - *Failures:* If input is not an array or empty, no HTTP requests will be sent.

---

#### 2.3 Website Availability Testing

- **Overview:** Executes HTTP GET requests to each website to check availability, capturing full response including headers and status codes.  
- **Nodes Involved:** Perform Site Test, Perform Site Test1

**Nodes:**

- **Perform Site Test**  
  - *Type:* HTTP Request  
  - *Role:* Sends HTTP request to each website URL obtained from Split Out.  
  - *Config:*  
    - URL dynamically set from input website item.  
    - Full HTTP response requested (`fullResponse: true`).  
    - Never error on HTTP errors (`neverError: true`) to capture response regardless of status code.  
    - On error: set to continue error output to avoid workflow failure on unreachable sites.  
  - *Connections:* Outputs to Merge node (both main and error outputs).  
  - *Failures:* Network errors, DNS resolution failures, or timeout can occur but handled gracefully.

- **Perform Site Test1**  
  - *Type:* HTTP Request  
  - *Role:* Duplicate of Perform Site Test, used after Wait block for second round of testing before email alert.  
  - *Config:* Same as Perform Site Test.  
  - *Connections:* Outputs to Merge1 node.  
  - *Failures:* Same as Perform Site Test.

---

#### 2.4 Status Calculation & Routing

- **Overview:** Evaluates HTTP response status codes to determine if websites are UP (status code ≤ 400) or DOWN, then routes flow appropriately for alerting or waiting.  
- **Nodes Involved:** Calculate Status, Status Router, Merge, Calculate Status1, Status Router1, Merge1

**Nodes:**

- **Calculate Status**  
  - *Type:* Set  
  - *Role:* Extracts and assigns:  
    - `date` from HTTP response headers.  
    - `website` URL from Split Out.  
    - Boolean `IS_UP` based on statusCode ≤ 400.  
  - *Connections:* Outputs to Status Router.  
  - *Failures:* Missing headers or malformed responses could cause null values.

- **Status Router**  
  - *Type:* Switch  
  - *Role:* Routes data based on `IS_UP`:  
    - `UP` branch if `IS_UP` is true.  
    - `DOWN` branch if false.  
  - *Connections:*  
    - DOWN branch leads to Wait node.  
    - UP branch has no output (ends flow).  
  - *Failures:* Expression errors if `IS_UP` missing or not boolean.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines outputs of Perform Site Test and error outputs for Calculate Status.  
  - *Connections:* Inputs from Perform Site Test success and error outputs; outputs to Calculate Status.  
  - *Failures:* Incorrect merging could cause data loss.

- **Calculate Status1**  
  - *Type:* Set  
  - *Role:* Similar to Calculate Status but for second HTTP test after waiting.  
  - *Connections:* Outputs to Status Router1.  
  - *Failures:* Same as Calculate Status.

- **Status Router1**  
  - *Type:* Switch  
  - *Role:* Routes based on second test `IS_UP`:  
    - `UP` branch ends flow.  
    - `DOWN` branch leads to Send email node.  
  - *Connections:* Outputs accordingly.  
  - *Failures:* Same as Status Router.

- **Merge1**  
  - *Type:* Merge  
  - *Role:* Merges Perform Site Test1 outputs before Calculate Status1.  
  - *Connections:* Inputs from Perform Site Test1 success and error outputs; outputs to Calculate Status1.  
  - *Failures:* Same as Merge.

---

#### 2.5 Wait Mechanism & Notification Handling

- **Overview:** Implements a delay before sending email alerts to avoid immediate notifications on transient failures; sends Telegram alerts immediately on first detection of downtime; sends email alerts after the wait period if the site is still down.  
- **Nodes Involved:** Wait, Send a text message (Telegram), Send email

**Nodes:**

- **Wait**  
  - *Type:* Wait  
  - *Role:* Delays workflow execution for configured seconds (`wait_secs` from Config node, default 5 sec).  
  - *Config:* Duration set by expression referencing Config node's `wait_secs` parameter.  
  - *Connections:* After wait, triggers Perform Site Test1 to verify site status again.  
  - *Failures:* Long wait times may cause workflow execution queueing issues.

- **Send a text message**  
  - *Type:* Telegram  
  - *Role:* Sends Telegram message alert for website downtime immediately after first detection.  
  - *Config:*  
    - Chat ID from Config.  
    - Message text includes website URL and note it's down.  
    - Telegram Bot API credentials required.  
  - *Connections:* After Send email node (to chain email after Telegram).  
  - *Failures:* Telegram API errors, invalid chat ID, or credential misconfiguration.

- **Send email**  
  - *Type:* Email Send  
  - *Role:* Sends a formatted HTML email alert if the website remains down after the wait period and second test.  
  - *Config:*  
    - To email address set to configured alert email (hardcoded here as example).  
    - From email configured.  
    - Email body is a rich HTML template with placeholders for website, date/time, and contact info.  
    - SMTP credentials required.  
  - *Connections:* Outputs to Send a text message node to send Telegram alert afterwards.  
  - *Failures:* SMTP authentication errors, invalid email addresses, or connectivity issues.

---

#### 2.6 Auxiliary Notes & Credentials Guidance

- **Overview:** Provides user guidance and instructions for configuring Telegram bot, email credentials, and contact info. Also contains author credits.  
- **Nodes Involved:** Sticky Note, Sticky Note2, Sticky Note3, Sticky Note10

**Nodes:**

- **Sticky Note (Instructions; Telegram bot)**  
  - Explains how to create a Telegram bot, retrieve the chat ID, and set credentials in n8n.  
  - Positioned near Telegram node for contextual help.

- **Sticky Note2**  
  - Notes about adding SMTP email credentials and Gmail server port details.

- **Sticky Note3**  
  - Instructions to add websites, Telegram chat ID, and email address in the Config node.

- **Sticky Note10**  
  - Author credits with contact info and social links.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                      | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                   |
|---------------------|--------------------|------------------------------------|----------------------|----------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger   | Initiates workflow every minute    |                      | Config               |                                                                                                               |
| Config              | Set                | Defines websites and alert settings| Schedule Trigger     | Edit Fields           | See Sticky Note3: Add websites, Telegram chat ID, email here                                                  |
| Edit Fields         | Set                | Converts websites string to array  | Config               | Split Out             |                                                                                                               |
| Split Out           | Split Out           | Splits website array into items    | Edit Fields           | Perform Site Test     |                                                                                                               |
| Perform Site Test    | HTTP Request        | Tests website availability         | Split Out             | Merge                 |                                                                                                               |
| Merge               | Merge               | Combines success and error outputs | Perform Site Test     | Calculate Status      |                                                                                                               |
| Calculate Status    | Set                 | Determines if site is UP or DOWN   | Merge                 | Status Router         |                                                                                                               |
| Status Router       | Switch              | Routes flow based on site status   | Calculate Status      | Wait (DOWN branch)    |                                                                                                               |
| Wait                | Wait                | Delays before rechecking site      | Status Router         | Perform Site Test1    |                                                                                                               |
| Perform Site Test1   | HTTP Request        | Re-tests site after delay           | Wait                  | Merge1                |                                                                                                               |
| Merge1              | Merge               | Combines success and error outputs | Perform Site Test1    | Calculate Status1     |                                                                                                               |
| Calculate Status1   | Set                 | Determines site status again        | Merge1                | Status Router1        |                                                                                                               |
| Status Router1      | Switch              | Routes flow for final alert decision| Calculate Status1     | Send email (DOWN)     |                                                                                                               |
| Send email          | Email Send          | Sends email alert for downtime     | Status Router1        | Send a text message   | See Sticky Note2: Add Email credentials (SMTP) details                                                        |
| Send a text message | Telegram             | Sends Telegram alert message       | Send email            |                      | See Sticky Note: Telegram bot setup instructions                                                             |
| Sticky Note         | Sticky Note         | Telegram bot instructions          |                      |                      | Detailed Telegram bot and chat ID setup instructions                                                         |
| Sticky Note2        | Sticky Note         | SMTP credentials notes             |                      |                      | Add Email credentials (SMTP). Port: 465, Gmail server: mail.google.com                                        |
| Sticky Note3        | Sticky Note         | Configuration instructions        |                      |                      | Add your websites here; Also add Telegram Chat ID and Mail Address                                           |
| Sticky Note10       | Sticky Note         | Author credits                    |                      |                      | Author: Muntasir Mubin; Founder of WebDextro Ltd; contact via email and socials                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add "Schedule Trigger" node:**  
   - Set to trigger every 1 minute (interval: minutes = 1).

3. **Add "Set" node named "Config":**  
   - Add string parameter `Websites` with newline-separated URLs (example: `https://google.com\nhttps://bad.url\nhttps://webdextro.org`).  
   - Add number parameter `wait_secs` with value (default 5).  
   - Add string parameter `telegram_chat_id` with your Telegram chat ID.  
   - Add string parameter `alert_email` with your alert email address.  
   - Connect Schedule Trigger → Config.

4. **Add "Set" node named "Edit Fields":**  
   - Create field `Websites` of type array with expression:  
     ```js
     {{$json.Websites.split('\n').filter(url => url)}}
     ```  
   - Connect Config → Edit Fields.

5. **Add "Split Out" node:**  
   - Set fieldToSplitOut to `Websites`.  
   - Connect Edit Fields → Split Out.

6. **Add "HTTP Request" node named "Perform Site Test":**  
   - URL: Expression `{{$json.Websites}}` (current item URL).  
   - Set Response to "Full Response".  
   - Enable "Never Fail" option to true.  
   - On error: Continue with error output.  
   - Connect Split Out → Perform Site Test.

7. **Add "Merge" node:**  
   - Connect both success and error outputs of Perform Site Test as inputs to Merge.  
   - Connect Perform Site Test → Merge.

8. **Add "Set" node named "Calculate Status":**  
   - Assign:  
     - `date` = `{{$json.headers.date}}` (from HTTP response headers)  
     - `website` = `{{$json.Websites}}` (from Split Out)  
     - `IS_UP` = `{{$json.statusCode <= 400}}` (boolean)  
   - Connect Merge → Calculate Status.

9. **Add "Switch" node named "Status Router":**  
   - Condition:  
     - Output "UP" if `IS_UP` is true.  
     - Output "DOWN" if `IS_UP` is false.  
   - Connect Calculate Status → Status Router.

10. **Add "Wait" node:**  
    - Set duration to expression `{{$('Config').item.json.wait_secs}}` seconds.  
    - Connect Status Router "DOWN" output → Wait.

11. **Add second "HTTP Request" node named "Perform Site Test1":**  
    - Same configuration as Perform Site Test.  
    - URL expression: `{{$json.Websites}}`.  
    - On error: continue error output.  
    - Connect Wait → Perform Site Test1.

12. **Add second "Merge" node named "Merge1":**  
    - Connect Perform Site Test1 success and error outputs → Merge1.

13. **Add "Set" node named "Calculate Status1":**  
    - Same assignments as Calculate Status.  
    - Connect Merge1 → Calculate Status1.

14. **Add "Switch" node named "Status Router1":**  
    - Same condition logic as Status Router.  
    - Connect Calculate Status1 → Status Router1.

15. **Add "Email Send" node named "Send email":**  
    - Set To: your alert email (or use expression from Config).  
    - Set From: your sender email.  
    - Use the provided HTML email template for body (can copy from original).  
    - Configure SMTP credentials accordingly.  
    - Connect Status Router1 "DOWN" output → Send email.

16. **Add "Telegram" node named "Send a text message":**  
    - Text: `{{$json.website}} is DOWN.\nFrom: n8n uptime`  
    - Chat ID: expression from Config `{{$('Config').item.json.telegram_chat_id}}`.  
    - Set credentials with your Telegram Bot API token.  
    - Connect Send email → Send a text message.

17. **(Optional) Add Sticky Notes:**  
    - Add notes near respective nodes with instructions for Telegram bot creation, chat ID retrieval, SMTP setup, and author credits.

18. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Instructions to create Telegram Bot, retrieve chat ID, and configure Telegram credentials for sending messages.                                                                                                                          | See Sticky Note near Telegram node.                                                                      |
| Add your websites as newline separated URLs in Config node. Also add your Telegram chat ID and alert email here.                                                                                                                        | See Sticky Note3 near Config node.                                                                       |
| Add SMTP email credentials for sending email alerts. Gmail server details: mail.google.com, port 465.                                                                                                                                     | See Sticky Note2 near Send email node.                                                                   |
| Author: Muntasir Mubin, Founder & CEO of WebDextro Ltd, UK-based Web & App development company. Contact via email, Facebook, and LinkedIn provided.                                                                                     | See Sticky Note10.                                                                                        |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---