Send interview reminders (10 minutes) from Google Calendar to Slack DMs

https://n8nworkflows.xyz/workflows/send-interview-reminders--10-minutes--from-google-calendar-to-slack-dms-6683


# Send interview reminders (10 minutes) from Google Calendar to Slack DMs

### 1. Workflow Overview

This workflow automates sending interview reminders via Slack direct messages (DMs) to interviewers shortly before an interview starts, using Google Calendar events as the source of interview schedules. It is designed to:

- Query upcoming interview events from a specified Google Calendar.
- Filter events starting within a configurable lead time (default 10 minutes).
- Extract relevant links (meeting, CV, notes) from event descriptions.
- Identify interview attendees (excluding organizers, declined, or external emails).
- Send personalized Slack DMs to each valid attendee with interview details.
- Fall back to posting the reminder in a Slack channel if DM fails.
- Track sent reminders to avoid duplicate notifications.

**Logical blocks:**

- **1.1 Scheduling & Configuration:** Triggering the workflow periodically and setting config variables.
- **1.2 Google Calendar Event Retrieval:** Fetching all events from the configured calendar.
- **1.3 Event Filtering & Preparation:** Filtering events starting soon, parsing details, and preparing ping payloads.
- **1.4 Deduplication Check:** Verifying if a reminder for the attendee-event pair was already sent.
- **1.5 Slack Messaging:** Sending the DM reminder or posting to fallback channel if DM unavailable.
- **1.6 Logging:** Recording sent pings to avoid repeats.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Configuration

**Overview:**  
This block periodically triggers the workflow and sets configuration parameters such as calendar name, company domain, timezone, lead time before interviews, and fallback Slack channel.

**Nodes Involved:**  
- Cron Trigger  
- Set: Config

**Node Details:**  

- **Cron Trigger**  
  - Type: Trigger  
  - Role: Periodically starts the workflow on a schedule (default schedule is empty; user must configure).  
  - Inputs: None  
  - Outputs: Connects to Set: Config  
  - Failure Modes: None, but misconfigured schedule means no triggers.

- **Set: Config**  
  - Type: Set  
  - Role: Defines workflow variables:
    - `CALENDAR_NAME`: "Interviews" (target Google Calendar)  
    - `COMPANY_DOMAIN`: Company email domain to filter attendees (e.g., "domain.com")  
    - `TIMEZONE`: Timezone for date formatting ("Asia/Kolkata")  
    - `LEAD_MINUTES`: Lead time before interview to send reminders (default "10")  
    - `FALLBACK_CHANNEL`: Slack channel for fallback messages ("#recruiting-alerts")  
  - Inputs: From Cron Trigger  
  - Outputs: Connects to Get many events  
  - Edge Cases: Misconfigured values may cause filtering or timezone issues.

---

#### 1.2 Google Calendar Event Retrieval

**Overview:**  
Fetch all events from the specified Google Calendar for processing.

**Nodes Involved:**  
- Get many events

**Node Details:**

- **Get many events**  
  - Type: Google Calendar  
  - Role: Retrieves all events from the calendar named by `CALENDAR_NAME` (though the calendar name parameter is empty in config, this should be set to the intended calendar).  
  - Operation: `getAll` (fetch all events)  
  - Inputs: From Set: Config  
  - Outputs: Connects to Prepare Pings  
  - Credentials: Google Calendar API with appropriate scopes  
  - Failure Modes: Authentication errors, API rate limits, empty calendar.

---

#### 1.3 Event Filtering & Preparation

**Overview:**  
Process each event to identify upcoming interviews within the lead time, parse relevant links from descriptions, and prepare individual reminder items per attendee.

**Nodes Involved:**  
- Prepare Pings

**Node Details:**  

- **Prepare Pings**  
  - Type: Function  
  - Role:  
    - Filters events to those starting within `LEAD_MINUTES` from current time.  
    - Ignores events with no attendees, no start time, or already started.  
    - Extracts CV, notes, and meeting links from event description or hangoutLink field.  
    - For each attendee (excluding organizers, declined, or missing email), creates a reminder item with:
      - Event ID, title, start time, attendee email and name  
      - Links: CV, notes, meeting  
      - Localized start time string formatted in configured timezone  
      - Unique key combining event ID and attendee email for deduplication  
  - Inputs: From Get many events  
  - Outputs: Multiple items, one per valid attendee-event pair  
  - Key Expressions: Uses config variables for lead time and timezone, regex for link extraction  
  - Edge Cases: Missing or malformed description links, attendees without emails, timezone conversion errors.

---

#### 1.4 Deduplication Check

**Overview:**  
Checks whether a reminder was already sent for a specific event-attendee pair to avoid duplicate messages.

**Nodes Involved:**  
- Check Ledger  
- If Not Already Sent

**Node Details:**  

- **Check Ledger**  
  - Type: Function  
  - Role: Reads global static data `sentPings` to check if the unique key exists, marking if the reminder was already sent.  
  - Inputs: From Prepare Pings  
  - Outputs: JSON including boolean `alreadySent` flag  
  - Edge Cases: Static data not initialized; read/write concurrency issues.

- **If Not Already Sent**  
  - Type: If  
  - Role: Routes flow only if `alreadySent` is false (i.e., no prior reminder sent)  
  - Inputs: From Check Ledger  
  - Outputs: Success branch to User Found?, else no further processing  
  - Edge Cases: Expression evaluation failure.

---

#### 1.5 Slack Messaging

**Overview:**  
Attempts to send a DM reminder to the interviewer via Slack. If user not found or DM fails, posts reminder message to a fallback Slack channel.

**Nodes Involved:**  
- User Found?  
- Send DM  
- Post to Fallback Channel

**Node Details:**  

- **User Found?**  
  - Type: If  
  - Role: Checks if `ok` property is true in Slack user lookup response (indicating user found and DM possible)  
  - Inputs: From If Not Already Sent  
  - Outputs:
    - True branch: Send DM  
    - False branch: Post to Fallback Channel  
  - Edge Cases: Slack API rate limits, user not found, missing user ID.

- **Send DM**  
  - Type: Slack  
  - Role: Sends a direct message to Slack user ID with interview reminder details:
    - Time (lead minutes until interview)  
    - Candidate name extracted from event title  
    - Localized interview time  
    - Role derived from event title  
    - Meeting link, CV link, notes link as clickable attachments  
  - Channel: Set dynamically to user ID from JSON  
  - Authentication: OAuth2 Slack credential  
  - Inputs: From User Found? (true branch)  
  - Outputs: Connects to Record Ping Sent  
  - Edge Cases: Slack API errors, invalid user ID, message formatting issues.

- **Post to Fallback Channel**  
  - Type: Slack  
  - Role: Posts reminder to a predefined Slack channel if DM fails:
    - Includes interviewer email and name  
    - Candidate, time, role, and links as above  
  - Channel: Configured fallback channel (e.g., "#recruiting-alerts")  
  - Authentication: OAuth2 Slack credential  
  - Inputs: From User Found? (false branch)  
  - Outputs: Connects to Record Ping Sent  
  - Edge Cases: Channel not found, permissions errors.

---

#### 1.6 Logging

**Overview:**  
Records sent reminders in global static data to prevent duplicate notifications.

**Nodes Involved:**  
- Record Ping Sent

**Node Details:**  

- **Record Ping Sent**  
  - Type: Function  
  - Role: Updates global static data `sentPings` with the unique key, timestamp, event ID, and attendee email, marking the reminder as sent.  
  - Inputs: From Send DM or Post to Fallback Channel  
  - Outputs: JSON with confirmation flag  
  - Edge Cases: Static data storage limits, concurrent writes, loss of static data on workflow reset.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)                | Sticky Note                           |
|-----------------------|---------------------|----------------------------------------|------------------------|------------------------------|-------------------------------------|
| Cron Trigger          | Cron Trigger        | Periodic workflow start                 | —                      | Set: Config                  |                                     |
| Set: Config           | Set                 | Defines constants and config variables  | Cron Trigger           | Get many events              |                                     |
| Get many events       | Google Calendar     | Retrieves all events from calendar      | Set: Config            | Prepare Pings               |                                     |
| Prepare Pings         | Function            | Filters and prepares reminder items     | Get many events        | Check Ledger                |                                     |
| Check Ledger          | Function            | Checks if reminder was already sent     | Prepare Pings          | If Not Already Sent          |                                     |
| If Not Already Sent   | If                  | Filters items not already reminded      | Check Ledger           | User Found?                 |                                     |
| User Found?           | If                  | Routes based on Slack user existence    | If Not Already Sent    | Send DM, Post to Fallback   |                                     |
| Send DM               | Slack               | Sends direct message reminder           | User Found? (true)     | Record Ping Sent            |                                     |
| Post to Fallback Channel | Slack            | Posts reminder in fallback Slack channel| User Found? (false)    | Record Ping Sent            |                                     |
| Record Ping Sent      | Function            | Logs sent reminders                      | Send DM, Post to Fallback Channel | —                    |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger node**  
   - Set schedule to desired interval (e.g., every 5 minutes).  
   - No parameters needed beyond schedule.

2. **Create Set node named "Set: Config"**  
   - Add string parameters:  
     - `CALENDAR_NAME` = "Interviews"  
     - `COMPANY_DOMAIN` = "domain.com"  
     - `TIMEZONE` = "Asia/Kolkata"  
     - `LEAD_MINUTES` = "10"  
     - `FALLBACK_CHANNEL` = "#recruiting-alerts"  
   - Connect Cron Trigger output to this node.

3. **Create Google Calendar node named "Get many events"**  
   - Operation: `getAll`  
   - Calendar: Select calendar named "Interviews" or appropriate calendar.  
   - Connect Set: Config output to this node.  
   - Configure Google Calendar credentials with read access.

4. **Create Function node named "Prepare Pings"**  
   - Paste code that:  
     - Filters events starting within `LEAD_MINUTES`.  
     - Extracts CV, notes, meeting links from description or hangoutLink.  
     - Creates one item per attendee who is not organizer, declined, or external.  
     - Formats local start time in configured timezone.  
     - Creates `uniqueKey` by combining event ID and attendee email.  
   - Connect Get many events output to this node.

5. **Create Function node named "Check Ledger"**  
   - Paste code that:  
     - Checks global static data `sentPings` for `uniqueKey`.  
     - Adds `alreadySent` boolean to output JSON.  
   - Connect Prepare Pings output to this node.

6. **Create If node named "If Not Already Sent"**  
   - Set condition: Boolean `alreadySent` equals false (negate to filter not sent).  
   - Connect Check Ledger output to this node.

7. **Create If node named "User Found?"**  
   - Condition: Boolean `ok` equals true (this requires Slack user lookup beforehand; see note below).  
   - Connect If Not Already Sent (true output) to this node.

8. **Create Slack node named "Send DM"**  
   - Operation: Send Message  
   - Channel: Expression set to `{{$json.user.id}}` (user ID from Slack lookup)  
   - Text includes interview info with markdown formatting:  
     - Interview time, candidate name, role, meeting link, CV link (if any), notes link (if any).  
   - Authentication: Slack OAuth2 credentials.  
   - Connect User Found? (true) output to this node.

9. **Create Slack node named "Post to Fallback Channel"**  
   - Operation: Send Message  
   - Channel: Expression set to `{{$node["Set: Config"].json["FALLBACK_CHANNEL"]}}`  
   - Text similar to DM but addressed to fallback channel, includes attendee email.  
   - Authentication: Slack OAuth2 credentials.  
   - Connect User Found? (false) output to this node.

10. **Create Function node named "Record Ping Sent"**  
    - Paste code that:  
      - Stores sent reminder info in global static data `sentPings` keyed by uniqueKey with timestamp, event ID, attendee email.  
      - Adds confirmation flag to output JSON.  
    - Connect outputs of Send DM and Post to Fallback Channel to this node.

11. **Slack User Lookup (Missing in JSON but required for User Found?)**  
    - Insert Slack API user lookup node before User Found? node to populate `$json.user.id` and `$json.ok`.  
    - Connect If Not Already Sent true output to Slack user lookup node, then to User Found?.  
    - This step is essential to check if DM can be sent.

12. **Verify credential setup**  
    - Google Calendar node: OAuth2 or API key with calendar read scope.  
    - Slack nodes: OAuth2 with chat:write and users:read scopes.

13. **Test workflow with sample events**  
    - Ensure event descriptions include links using expected formats (CV: URL, Notes: URL).  
    - Confirm timezones and lead time work as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Slack user lookup node is required before "User Found?" node to provide user ID and status for DM capability check.   | This node is not present in JSON but necessary for correct routing.                                          |
| Regex patterns in "Prepare Pings" node extract links with labels "CV:", "Notes:" and meeting URLs from description.  | Ensure event descriptions follow expected format for link extraction.                                        |
| Global static data is used to store sent reminders; workflow restart or reset may clear this data and cause duplicates. | Consider persisting data externally if needed for durability.                                                |
| Timezone is used for formatting local start time string; ensure configured timezone matches your interview location. |                                                                                                             |
| Slack OAuth2 credentials require appropriate scopes: chat:write, users:read, im:write.                                |                                                                                                             |

---

**Disclaimer:**  
This documentation was generated from an n8n workflow automation respecting all applicable content policies. The workflow processes legal and public data only.