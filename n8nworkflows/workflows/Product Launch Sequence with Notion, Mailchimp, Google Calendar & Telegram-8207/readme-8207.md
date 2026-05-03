Product Launch Sequence with Notion, Mailchimp, Google Calendar & Telegram

https://n8nworkflows.xyz/workflows/product-launch-sequence-with-notion--mailchimp--google-calendar---telegram-8207


# Product Launch Sequence with Notion, Mailchimp, Google Calendar & Telegram

### 1. Workflow Overview

This workflow automates client nurture communications and testimonial collection by integrating Notion, Mailchimp (via SMTP email node), Google Calendar (implicitly via scheduling), and Telegram notifications. It is designed primarily for businesses and marketing teams aiming to maintain engagement with clients through personalized milestone emails sent at 7, 30, and 60 days after a triggering event, tracked via a Notion database. Additionally, it collects testimonials via a webhook and notifies via Telegram.

Logical blocks:

- **1.1 Input Triggers:** Manual trigger, scheduled trigger, and webhook for receiving testimonials.
- **1.2 Initialization and Settings:** Setting up workflow parameters and calculating current date/time context.
- **1.3 Notion Data Query:** Fetching client milestone data from Notion.
- **1.4 Milestone Determination:** Calculating elapsed days and branching logic for sending emails.
- **1.5 Email Dispatch:** Sending personalized emails at 7, 30, and 60-day milestones.
- **1.6 Notifications:** Sending Telegram notifications post-email and on testimonial receipt.
- **1.7 Testimonial Handling:** Parsing webhook data for testimonials and responding accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Triggers

- **Overview:** This block initiates the workflow either manually, on a schedule, or via HTTP webhook input for testimonials.
- **Nodes Involved:** Manual Trigger, Schedule Trigger, Webhook

##### Manual Trigger
- **Type:** Manual trigger node, initiates workflow on user request.
- **Configuration:** Default empty; used for testing or manual intervention.
- **Inputs:** None.
- **Outputs:** Connects to Settings node.
- **Failure Points:** None significant; manual start.
- **Version:** 1

##### Schedule Trigger
- **Type:** Scheduled trigger node.
- **Configuration:** Default schedule (not explicitly set here) triggers workflow automatically.
- **Inputs:** None.
- **Outputs:** Connects to Settings node.
- **Failure Points:** Scheduling misconfiguration or service downtime.
- **Version:** 1.2

##### Webhook
- **Type:** Webhook node to receive external HTTP POST data.
- **Configuration:** Default, listens for incoming testimonial data.
- **Inputs:** External HTTP request.
- **Outputs:** Connects to Parse node.
- **Failure Points:** Incorrect webhook URL usage, missing data in payload.
- **Version:** 1

---

#### 1.2 Initialization and Settings

- **Overview:** Sets up workflow variables and computes the current date/time for later calculations.
- **Nodes Involved:** Settings, Get Date

##### Settings
- **Type:** Set node.
- **Configuration:** No explicit parameters; serves as a placeholder to trigger downstream nodes.
- **Inputs:** From Manual or Schedule Trigger.
- **Outputs:** To Get Date node.
- **Failure Points:** None; simple pass-through.
- **Version:** 2

##### Get Date
- **Type:** Code node.
- **Configuration:** JavaScript code to obtain current date/time, likely in ISO format.
- **Inputs:** From Settings node.
- **Outputs:** To Query Notion node.
- **Failure Points:** Code errors, time zone miscalculations.
- **Version:** 2

---

#### 1.3 Notion Data Query

- **Overview:** Queries the Notion API to retrieve client milestone data to determine which clients to email.
- **Nodes Involved:** Query Notion

##### Query Notion
- **Type:** HTTP Request node.
- **Configuration:** Configured to query Notion database using API credentials.
- **Inputs:** From Get Date node.
- **Outputs:** To Calculate node.
- **Key Variables:** Uses Notion API credentials.
- **Failure Points:** API auth errors, rate limits, malformed query.
- **Version:** 4.2
- **Credentials:** Notion API

---

#### 1.4 Milestone Determination

- **Overview:** Calculates elapsed days since client milestone and uses conditional logic to decide which milestone emails to send.
- **Nodes Involved:** Calculate, IF Day 7, IF Day 30, IF Day 60

##### Calculate
- **Type:** Code node.
- **Configuration:** JavaScript code calculates days elapsed since milestone date from Notion data.
- **Inputs:** From Query Notion.
- **Outputs:** To IF Day 7, IF Day 30, IF Day 60 nodes.
- **Failure Points:** Data parsing errors, date format issues.
- **Version:** 2

##### IF Day 7
- **Type:** If node.
- **Configuration:** Checks if days elapsed equals 7.
- **Inputs:** From Calculate node.
- **Outputs:** If true, to Email Day 7 node; else no action.
- **Failure Points:** Expression errors.
- **Version:** 2

##### IF Day 30
- **Type:** If node.
- **Configuration:** Checks if days elapsed equals 30.
- **Inputs:** From Calculate node.
- **Outputs:** If true, to Email Day 30 node.
- **Failure Points:** Expression errors.
- **Version:** 2

##### IF Day 60
- **Type:** If node.
- **Configuration:** Checks if days elapsed equals 60.
- **Inputs:** From Calculate node.
- **Outputs:** If true, to Email Day 60 node.
- **Failure Points:** Expression errors.
- **Version:** 2

---

#### 1.5 Email Dispatch

- **Overview:** Sends milestone emails corresponding to days 7, 30, and 60. Each email triggers a Telegram notification.
- **Nodes Involved:** Email Day 7, Email Day 30, Email Day 60

##### Email Day 7
- **Type:** Email Send node.
- **Configuration:** SMTP configured to send email for day 7 milestone.
- **Inputs:** From IF Day 7 node.
- **Outputs:** To Telegram Notify node.
- **Failure Points:** SMTP auth failure, email formatting errors.
- **Credentials:** SMTP
- **Version:** 2.1

##### Email Day 30
- **Type:** Email Send node.
- **Configuration:** SMTP configured to send email for day 30 milestone.
- **Inputs:** From IF Day 30 node.
- **Outputs:** To Telegram Notify node.
- **Failure Points:** SMTP auth failure, email formatting errors.
- **Credentials:** SMTP
- **Version:** 2.1

##### Email Day 60
- **Type:** Email Send node.
- **Configuration:** SMTP configured to send email for day 60 milestone.
- **Inputs:** From IF Day 60 node.
- **Outputs:** To Telegram Notify node.
- **Failure Points:** SMTP auth failure, email formatting errors.
- **Credentials:** SMTP
- **Version:** 2.1

---

#### 1.6 Notifications

- **Overview:** Sends Telegram notifications to inform about sent milestone emails or new testimonial received.
- **Nodes Involved:** Telegram Notify, Notify Testimonial

##### Telegram Notify
- **Type:** Telegram node.
- **Configuration:** Sends message after milestone email sent.
- **Inputs:** From Email Day 7, Email Day 30, Email Day 60 nodes.
- **Outputs:** None.
- **Failure Points:** Telegram API errors, invalid chat ID.
- **Credentials:** Telegram API
- **Version:** 1

##### Notify Testimonial
- **Type:** Telegram node.
- **Configuration:** Sends notification when a testimonial is received via webhook.
- **Inputs:** From Parse node.
- **Outputs:** To Respond node.
- **Failure Points:** Telegram API errors.
- **Credentials:** Telegram API
- **Version:** 1

---

#### 1.7 Testimonial Handling

- **Overview:** Parses incoming webhook data for testimonials and sends acknowledgment via Telegram and webhook response.
- **Nodes Involved:** Parse, Respond

##### Parse
- **Type:** Code node.
- **Configuration:** JavaScript code parses webhook payload to extract testimonial info.
- **Inputs:** From Webhook node.
- **Outputs:** To Notify Testimonial node.
- **Failure Points:** Malformed input data, parsing errors.
- **Version:** 2

##### Respond
- **Type:** Respond to Webhook node.
- **Configuration:** Sends HTTP response back to webhook caller confirming receipt.
- **Inputs:** From Notify Testimonial node.
- **Outputs:** None.
- **Failure Points:** Response timeout or errors.
- **Version:** 1

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                          | Input Node(s)         | Output Node(s)              | Sticky Note                           |
|--------------------|--------------------|----------------------------------------|-----------------------|-----------------------------|-------------------------------------|
| Manual Trigger     | Manual Trigger      | Manual workflow initiation              | None                  | Settings                    |                                     |
| Schedule Trigger   | Schedule Trigger    | Automated scheduled initiation          | None                  | Settings                    |                                     |
| Settings          | Set                 | Initialize variables                     | Manual Trigger, Schedule Trigger | Get Date               |                                     |
| Get Date          | Code                | Compute current date/time                | Settings               | Query Notion                |                                     |
| Query Notion      | HTTP Request        | Fetch client milestone data from Notion | Get Date               | Calculate                  |                                     |
| Calculate         | Code                | Calculate days elapsed since milestone  | Query Notion           | IF Day 7, IF Day 30, IF Day 60 |                                  |
| IF Day 7          | If                  | Check if day 7 milestone                 | Calculate              | Email Day 7                 |                                     |
| IF Day 30         | If                  | Check if day 30 milestone                | Calculate              | Email Day 30                |                                     |
| IF Day 60         | If                  | Check if day 60 milestone                | Calculate              | Email Day 60                |                                     |
| Email Day 7       | Email Send          | Send milestone email for day 7           | IF Day 7               | Telegram Notify             |                                     |
| Email Day 30      | Email Send          | Send milestone email for day 30          | IF Day 30              | Telegram Notify             |                                     |
| Email Day 60      | Email Send          | Send milestone email for day 60          | IF Day 60              | Telegram Notify             |                                     |
| Telegram Notify   | Telegram            | Notify about sent milestone emails       | Email Day 7, Email Day 30, Email Day 60 | None            |                                     |
| Webhook           | Webhook             | Receive testimonial submissions          | None                   | Parse                      |                                     |
| Parse             | Code                | Parse testimonial payload                 | Webhook                | Notify Testimonial          |                                     |
| Notify Testimonial | Telegram            | Notify testimonial received               | Parse                  | Respond                    |                                     |
| Respond           | Respond to Webhook  | Send acknowledgment response              | Notify Testimonial      | None                       |                                     |
| Sticky Note       | Sticky Note         | (Empty content)                          | None                   | None                       |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node** named "Manual Trigger" with default settings.
2. **Create Schedule Trigger node** named "Schedule Trigger" with desired schedule (e.g., daily at a fixed time).
3. **Create Set node** named "Settings" with no parameters; connect outputs of Manual Trigger and Schedule Trigger to this node.
4. **Create Code node** named "Get Date" with JavaScript code to get current date/time, e.g.:

   ```javascript
   return [{ json: { currentDate: new Date().toISOString() } }];
   ```

   Connect Settings output to this node.
5. **Create HTTP Request node** named "Query Notion" configured to:

   - Use Notion API credentials.
   - Method: POST or GET depending on API.
   - Query the Notion database for clients with milestone dates.
   - Use current date from previous node if applicable.

   Connect Get Date output to this node.
6. **Create Code node** named "Calculate" with JavaScript code that calculates days elapsed since the milestone date fetched from Notion. Example logic:

   - Parse Notion response.
   - Calculate difference between current date and client milestone date.
   - Output days elapsed.

   Connect Query Notion output to this node.
7. **Create three If nodes** named "IF Day 7", "IF Day 30", and "IF Day 60":

   - Each node checks if calculated days elapsed equals 7, 30, or 60 respectively.
   - Expressions example for IF Day 7:

     ``` 
     {{$json["daysElapsed"]}} === 7
     ```

   Connect Calculate output to all three If nodes.
8. **Create three Email Send nodes** named "Email Day 7", "Email Day 30", and "Email Day 60":

   - Configure each with SMTP credentials.
   - Customize email content for each milestone.
   - Connect IF Day 7 true output to Email Day 7, IF Day 30 true to Email Day 30, IF Day 60 true to Email Day 60.
9. **Create Telegram node** named "Telegram Notify":

   - Configure Telegram API credentials.
   - Set message content to notify about email sent.
   - Connect outputs of all three Email nodes to this Telegram node.
10. **Create Webhook node** named "Webhook":

    - Configure to receive testimonial submissions.
11. **Create Code node** named "Parse":

    - Write JavaScript to parse incoming webhook JSON payload to extract testimonial details.
    - Connect Webhook output to this node.
12. **Create Telegram node** named "Notify Testimonial":

    - Configure to notify about new testimonial received.
    - Connect Parse output to this node.
13. **Create Respond to Webhook node** named "Respond":

    - Configure to send HTTP 200 acknowledgment.
    - Connect Notify Testimonial output to Respond node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow integrates Notion API, SMTP email, Telegram notifications, and webhook handling.  | Useful for CRM automation and client engagement pipelines.                                        |
| Telegram Bot setup needed with valid API token and chat ID configured in Telegram nodes.        | https://core.telegram.org/bots/api                                                                     |
| Notion API credentials require integration token with access to the relevant database.          | https://developers.notion.com/reference/authorization                                            |
| SMTP credentials must be configured for sending emails (e.g., Mailchimp SMTP, Gmail SMTP).      |                                                                                                   |
| Testing webhook requires external URL exposure or tunneling (e.g., with ngrok) if local.        |                                                                                                   |
| The workflow uses conditional logic to avoid sending duplicate emails outside milestone days.   |                                                                                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.