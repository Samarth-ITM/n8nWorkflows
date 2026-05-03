Recover No-Show Appointments with Calendly, Telegram & Email Alerts

https://n8nworkflows.xyz/workflows/recover-no-show-appointments-with-calendly--telegram---email-alerts-10146


# Recover No-Show Appointments with Calendly, Telegram & Email Alerts

### 1. Workflow Overview

This workflow, titled **"No-Show Appointment Recovery Bot"**, is designed to automatically detect no-show appointments from Calendly and trigger customer re-engagement and internal notifications. It targets high-intent leads who missed their scheduled calls and helps sales teams recover those opportunities by sending personalized Telegram messages with reschedule links and alerting the responsible sales representatives by email.

The workflow logically divides into the following blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the workflow every hour to check for missed appointments.
- **1.2 Data Retrieval:** Fetches all active scheduled events from Calendly using their API.
- **1.3 No-Show Filtering:** Identifies appointments that ended more than 30 minutes ago without attendance.
- **1.4 Lead Qualification:** Filters the no-shows to only those tagged as "High Intent" to prioritize recovery efforts.
- **1.5 Customer Outreach:** Sends a personalized reschedule message via Telegram to the no-show lead.
- **1.6 Internal Alert:** Notifies the assigned sales representative by email about the missed appointment and outreach action.

Supporting the workflow are comprehensive sticky notes clarifying each step, configuration requirements, and benefits.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Initiates the workflow every hour to start the recovery process.
- **Nodes Involved:**  
  - Hourly Check Trigger
- **Node Details:**

| Node Name            | Type              | Configuration & Role                                                                                   | Expressions / Variables                | I/O Connections                         | Version Notes | Edge Cases / Failures                     | Sub-workflow Reference |
|----------------------|-------------------|------------------------------------------------------------------------------------------------------|--------------------------------------|----------------------------------------|---------------|-------------------------------------------|------------------------|
| Hourly Check Trigger  | Cron Trigger      | Triggers the workflow exactly once every hour (`everyHour` mode).                                    | None                                 | Output to "Get Active Calendly Appointments" | v1            | Workflow not triggered if n8n service down or cron misconfigured; no input. | None                   |

---

#### 1.2 Data Retrieval

- **Overview:** Queries Calendly's API to fetch all active scheduled events for the user.
- **Nodes Involved:**  
  - Get Active Calendly Appointments
- **Node Details:**

| Node Name                  | Type            | Configuration & Role                                                                                      | Expressions / Variables                                   | I/O Connections                           | Version Notes | Edge Cases / Failures                              | Sub-workflow Reference |
|----------------------------|-----------------|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------|------------------------------------------|---------------|----------------------------------------------------|------------------------|
| Get Active Calendly Appointments | HTTP Request    | Performs a GET request to `https://api.calendly.com/scheduled_events?user={{$env.CALENDLY_USER_URI}}&status=active`. Uses environment variable `CALENDLY_USER_URI` to specify the user. | URL uses environment variable `{{$env.CALENDLY_USER_URI}}` to dynamically target user. | Input from "Hourly Check Trigger"; output to "Filter No-Show Appointments (30+ min past)" | v3            | API failures (network error, auth failure), empty or malformed response, rate limiting by Calendly API | None                   |

---

#### 1.3 No-Show Filtering

- **Overview:** Processes the list of active appointments to find those that ended more than 30 minutes ago and where the invitee did not attend.
- **Nodes Involved:**  
  - Filter No-Show Appointments (30+ min past)
- **Node Details:**

| Node Name                            | Type       | Configuration & Role                                                                                                  | Expressions / Variables                      | I/O Connections                             | Version Notes | Edge Cases / Failures                        | Sub-workflow Reference |
|------------------------------------|------------|---------------------------------------------------------------------------------------------------------------------|----------------------------------------------|--------------------------------------------|---------------|----------------------------------------------|------------------------|
| Filter No-Show Appointments (30+ min past) | Function   | Custom JavaScript function iterates over `items[0].json.collection`. For each appointment, calculates difference between current time and appointment end time. Filters only those ended >30 minutes ago and `attended` is false. Returns list of missed appointments. | Uses `new Date()` for current time; accesses `item.end_time`, `item.attended` | Input from "Get Active Calendly Appointments"; output to "Check If High Intent Lead" | v1            | Potential failure if Calendly data structure changes, missing fields, or time parsing errors | None                   |

---

#### 1.4 Lead Qualification

- **Overview:** Checks if the missed appointment’s contact tags include "High Intent" to prioritize follow-ups.
- **Nodes Involved:**  
  - Check If High Intent Lead
- **Node Details:**

| Node Name               | Type | Configuration & Role                                                                                              | Expressions / Variables                                | I/O Connections                          | Version Notes | Edge Cases / Failures                              | Sub-workflow Reference |
|-------------------------|------|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|-----------------------------------------|---------------|----------------------------------------------------|------------------------|
| Check If High Intent Lead | If   | Boolean condition node tests if the string `High Intent` is contained within `metadata.contact_tags` field of the appointment JSON. Proceeds only on true branch. | Expression: `={{ $json.metadata ? $json.metadata.contact_tags : '' }}` contains `"High Intent"` | Input from "Filter No-Show Appointments"; output true to "Send Reschedule Link via Telegram" | v1            | Fails if `metadata` or `contact_tags` is missing or malformed; empty tags result in skip | None                   |

---

#### 1.5 Customer Outreach

- **Overview:** Sends a personalized Telegram message to the no-show lead with a reschedule link and friendly text.
- **Nodes Involved:**  
  - Send Reschedule Link via Telegram
- **Node Details:**

| Node Name                     | Type     | Configuration & Role                                                                                                    | Expressions / Variables                                                  | I/O Connections                  | Version Notes | Edge Cases / Failures                                      | Sub-workflow Reference |
|-------------------------------|----------|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|---------------------------------|---------------|------------------------------------------------------------|------------------------|
| Send Reschedule Link via Telegram | Telegram | Sends a message to the lead’s Telegram ID from metadata. Message text includes greeting with invitee’s name and direct reschedule link (`uri` field). Uses Telegram API credentials configured in n8n. | Chat ID: `={{ $json.metadata ? $json.metadata.telegram_id : '' }}`<br>Text includes: `{{$json.invitee_name}}`, `{{$json.uri}}` | Input from "Check If High Intent Lead"; output to "Alert Sales Rep via Email" | v1            | Telegram API errors (auth, invalid chat ID), missing Telegram ID in metadata, network timeouts | None                   |

---

#### 1.6 Internal Alert

- **Overview:** Notifies the assigned sales representative by email about the missed appointment and that a Telegram message was sent.
- **Nodes Involved:**  
  - Alert Sales Rep via Email
- **Node Details:**

| Node Name             | Type      | Configuration & Role                                                                                                  | Expressions / Variables                                           | I/O Connections              | Version Notes | Edge Cases / Failures                                | Sub-workflow Reference |
|-----------------------|-----------|---------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|------------------------------|---------------|----------------------------------------------------|------------------------|
| Alert Sales Rep via Email | Email Send | Sends an email to the sales rep’s email address from metadata or defaults to `sales@yourcompany.com`. Email text confirms missed meeting, Telegram message sent, and includes reschedule link. Uses SMTP credentials configured in n8n. | To: `={{ $json.metadata ? $json.metadata.rep_email : 'sales@yourcompany.com' }}`<br>Subject and body include `{{$json.invitee_name}}`, `{{$json.uri}}` | Input from "Send Reschedule Link via Telegram" | v1            | SMTP errors, missing or invalid sales rep email, email delivery failures | None                   |

---

### 3. Summary Table

| Node Name                         | Node Type    | Functional Role                             | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                                 |
|----------------------------------|--------------|---------------------------------------------|-----------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Hourly Check Trigger              | Cron         | Scheduled hourly start of workflow          |                                   | Get Active Calendly Appointments  | ## 🚀 WORKFLOW START<br>**Purpose:** Automatically detect and recover no-show appointments<br>**Trigger:** Runs every hour to check for missed meetings<br>**Target:** High-intent leads who didn't attend their scheduled calls |
| Get Active Calendly Appointments  | HTTP Request | Retrieves active appointments from Calendly | Hourly Check Trigger              | Filter No-Show Appointments (30+ min past) | ## 📅 STEP 1: Fetch Appointments<br>**Action:** Retrieves all active scheduled events from Calendly<br>**API Endpoint:** Calendly scheduled_events API<br>**Data Retrieved:** Meeting details, invitee info, attendance status, metadata |
| Filter No-Show Appointments (30+ min past) | Function     | Filters appointments to identify no-shows   | Get Active Calendly Appointments  | Check If High Intent Lead         | ## 🔍 STEP 2: Identify No-Shows<br>**Logic:** Filters appointments where:<br>- Meeting ended 30+ minutes ago<br>- Attendee did NOT show up (attended = false)<br>**Output:** List of missed appointments only |
| Check If High Intent Lead         | If           | Checks if lead is marked "High Intent"      | Filter No-Show Appointments (30+ min past) | Send Reschedule Link via Telegram | ## 🎯 STEP 3: Priority Check<br>**Condition:** Only proceed if contact has "High Intent" tag<br>**Why:** Focus recovery efforts on qualified leads<br>**Source:** Checks metadata.contact_tags field |
| Send Reschedule Link via Telegram | Telegram     | Sends reschedule message to lead via Telegram | Check If High Intent Lead         | Alert Sales Rep via Email          | ## 💬 STEP 4: Customer Outreach<br>**Action:** Sends friendly reschedule message via Telegram<br>**Message Includes:**<br>- Personalized greeting<br>- Direct reschedule link<br>- Company signature<br>**Target:** Lead's Telegram ID from metadata |
| Alert Sales Rep via Email         | Email Send   | Sends internal email alert to sales rep     | Send Reschedule Link via Telegram |                                   | ## 📧 STEP 5: Internal Notification<br>**Action:** Alerts assigned sales rep about the no-show<br>**Email Contains:**<br>- Customer name<br>- Confirmation of Telegram message sent<br>- Reschedule link for manual follow-up<br>**Recipient:** Sales rep email from metadata |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node:**
   - Type: Cron
   - Name: "Hourly Check Trigger"
   - Set trigger to run every hour (`everyHour` mode).
   - No credentials needed.
   - Connect output to next node.

2. **Add HTTP Request node:**
   - Name: "Get Active Calendly Appointments"
   - HTTP Method: GET
   - URL: `https://api.calendly.com/scheduled_events?user={{$env.CALENDLY_USER_URI}}&status=active`
   - Use environment variable `CALENDLY_USER_URI` to specify Calendly user URI.
   - No authentication configured here (assumes public or API key handled via environment).
   - Connect input from "Hourly Check Trigger".
   - Connect output to next node.

3. **Add Function node:**
   - Name: "Filter No-Show Appointments (30+ min past)"
   - Paste the following JavaScript code:

     ```javascript
     const now = new Date();
     const thresholdMinutes = 30;

     const missed = [];
     for (const item of items[0].json.collection) {
       const endTime = new Date(item.end_time);
       const diffMinutes = (now - endTime) / 60000;
       if (diffMinutes > thresholdMinutes && !item.attended) {
         missed.push({ json: item });
       }
     }
     return missed;
     ```
   - Connect input from "Get Active Calendly Appointments".
   - Connect output to next node.

4. **Add If node:**
   - Name: "Check If High Intent Lead"
   - Set condition type: String
   - Condition: Check if `metadata.contact_tags` contains string "High Intent"
   - Expression for value1: `={{ $json.metadata ? $json.metadata.contact_tags : '' }}`
   - Connect input from "Filter No-Show Appointments (30+ min past)".
   - Connect the **true** output branch to next node.

5. **Add Telegram node:**
   - Name: "Send Reschedule Link via Telegram"
   - Credentials: Set up Telegram API credentials (bot token).
   - Chat ID: `={{ $json.metadata ? $json.metadata.telegram_id : '' }}`
   - Text message:

     ```
     Hi {{$json.invitee_name}}, we missed you on our call earlier!
     You can reschedule here: {{$json.uri}}
     — Team Shoonya
     ```

   - Connect input from true branch of "Check If High Intent Lead".
   - Connect output to next node.

6. **Add Email Send node:**
   - Name: "Alert Sales Rep via Email"
   - Credentials: Configure SMTP credentials (e.g., Gmail or SMTP server).
   - To Email: `={{ $json.metadata ? $json.metadata.rep_email : 'sales@yourcompany.com' }}`
   - From Email: `sales@yourcompany.com`
   - Subject: `Missed Appointment: {{$json.invitee_name}}`
   - Text:

     ```
     The meeting with {{$json.invitee_name}} appears missed.
     Telegram message sent successfully.
     Please follow up or reschedule here: {{$json.uri}}
     ```

   - Connect input from "Send Reschedule Link via Telegram".

7. **Set environment variables and credentials:**
   - Define `CALENDLY_USER_URI` environment variable with your Calendly user URI.
   - Configure Telegram API bot credentials in n8n credentials manager.
   - Configure SMTP email credentials in n8n credentials manager.

8. **Optional: Add sticky notes for documentation:**
   - Add notes describing each step, environment variables, and benefits to assist future maintenance.

9. **Activate the workflow:**
   - Test the workflow manually or wait for the hourly trigger.
   - Monitor logs for errors and verify Telegram and email notifications are sent correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow automatically prioritizes leads tagged as "High Intent" to optimize sales recovery efforts. | Internal business logic to focus on qualified leads.                                                  |
| Requires environment variable: `CALENDLY_USER_URI` to specify Calendly user for API calls.           | Set in n8n environment variables or workflow settings.                                                |
| Telegram API credentials must be configured with a valid bot token to send messages.                 | See Telegram Bot API docs: https://core.telegram.org/bots/api                                         |
| SMTP credentials must be configured for email alerts to sales reps.                                  | Ensure email provider allows SMTP relay and less secure app access if needed.                          |
| Time threshold for no-show detection is 30 minutes after scheduled end time.                         | Can be adjusted in the function node for different grace periods.                                     |
| The workflow runs hourly but can be adjusted to different schedules as needed.                       | Modify the Cron Trigger node accordingly.                                                             |
| For scalability, consider error handling and retries for API calls and message sends.                | n8n supports error workflows and retry policies that can be added if desired.                          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.