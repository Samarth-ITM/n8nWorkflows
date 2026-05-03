Daily Calendar Summary Notifications via Telegram from Google Calendar

https://n8nworkflows.xyz/workflows/daily-calendar-summary-notifications-via-telegram-from-google-calendar-6952


# Daily Calendar Summary Notifications via Telegram from Google Calendar

### 1. Workflow Overview

This workflow automates the process of sending a daily summary of Google Calendar events via Telegram each morning. Its primary use case is to notify a user about their scheduled events for the day, providing a concise but informative digest directly in a Telegram chat.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**  
  Automatically triggers the workflow every day at 7 AM to start the process.

- **1.2 Google Calendar Query**  
  Retrieves all calendar events scheduled for the current day from a specified Google Calendar.

- **1.3 Event Counting and Branching**  
  Counts the number of events fetched and determines if any events exist for the day, branching logic accordingly.

- **1.4 Message Construction**  
  If there are events, constructs a detailed summary message that includes event names, times, creators, organizers, types, links, descriptions, and locations.

- **1.5 Telegram Notification**  
  Sends the constructed message to a Telegram chat if events exist; otherwise, sends a default message indicating no meetings are scheduled.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow execution automatically every day at 7 AM to ensure timely daily notifications.

- **Nodes Involved:**  
  - "7am trigger"

- **Node Details:**  
  - **"7am trigger"**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger daily at 8:00 AM (likely local time, though the workflow name suggests 7 AM—timezone configuration should be verified)  
    - Key Parameters: Hour trigger set to 8  
    - Input: None (trigger node)  
    - Output: Triggers "Check google Calendar" node  
    - Failure Modes: Misconfigured timezone could cause unexpected trigger times; n8n instance downtime would delay trigger  
    - Version: 1.2

#### 2.2 Google Calendar Query

- **Overview:**  
  Fetches all events from a specific Google Calendar for the day, ordered by start time.

- **Nodes Involved:**  
  - "Check google Calendar"

- **Node Details:**  
  - **"Check google Calendar"**  
    - Type: Google Calendar node  
    - Configuration:  
      - Operation: getAll (fetch all events)  
      - Calendar: Set to "fr.french#holiday@group.v.calendar.google.com" (French holidays calendar)  
      - Options: orderBy startTime, returnAll true  
      - Credentials: Google Calendar OAuth2 API connected  
    - Input: Trigger from "7am trigger"  
    - Output: Passes fetched events to "Count event" node  
    - Failure Modes: OAuth token expiration, API quota limits, incorrect calendar ID, network issues  
    - Version: 1.3

#### 2.3 Event Counting and Branching

- **Overview:**  
  Processes the fetched events to count how many are scheduled, then routes workflow execution based on whether events exist.

- **Nodes Involved:**  
  - "Count event"  
  - "If condition"

- **Node Details:**  
  - **"Count event"**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Parses the output from "Check google Calendar", normalizes the events into an array, counts the total number  
      - Returns JSON with `eventCount` and the array `events`  
    - Input: Output from "Check google Calendar"  
    - Output: Passes count and events to "If condition"  
    - Failure Modes: Unexpected data structure from Google Calendar could cause errors; empty/null data handling is implemented  
    - Version: 2

  - **"If condition"**  
    - Type: If node  
    - Configuration:  
      - Condition: Checks if `eventCount` > 0  
      - Branches: True if events exist, False otherwise  
    - Input: From "Count event"  
    - Output:  
      - True branch leads to "Message code"  
      - False branch leads to "Message no meeting today"  
    - Failure Modes: Expression evaluation errors if input data is malformed  
    - Version: 2.2

#### 2.4 Message Construction

- **Overview:**  
  Constructs a comprehensive message summarizing all calendar events for the day.

- **Nodes Involved:**  
  - "Message code"

- **Node Details:**  
  - **"Message code"**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Aggregates all events from the input data  
      - Builds a formatted message string including:  
        - Greeting and count of events  
        - For each event: name, start/end times, creator, organizer, event type, Google Calendar link, description, and location  
      - Returns the message in JSON as `text`  
    - Input: True branch output from "If condition"  
    - Output: Passes message to "Send sum up message" node  
    - Failure Modes: Missing event fields handled with defaults ("N/A", "No description"); expression errors if input data is not as expected  
    - Version: 2

#### 2.5 Telegram Notification

- **Overview:**  
  Sends the constructed or default message to a Telegram chat via a bot.

- **Nodes Involved:**  
  - "Send sum up message"  
  - "Message no meeting today"

- **Node Details:**  
  - **"Send sum up message"**  
    - Type: Telegram node  
    - Configuration:  
      - Sends text from previous node’s `text` field  
      - Chat ID: Set to "1234" (placeholder, to be replaced with actual chat ID)  
      - Credentials: Connected to Telegram API via bot token  
    - Input: From "Message code"  
    - Output: None (end node)  
    - Failure Modes: Invalid Telegram bot token, wrong chat ID, Telegram API downtime  
    - Version: 1.2

  - **"Message no meeting today"**  
    - Type: Telegram node  
    - Configuration:  
      - Sends static message: "👋 Hi! Your calendar is clear for today 🗓️✨"  
      - Chat ID: "1234" (same as above)  
      - Credentials: Telegram API credentials connected  
    - Input: False branch from "If condition"  
    - Output: None (end node)  
    - Failure Modes: Same as above  
    - Version: 1.2

---

### 3. Summary Table

| Node Name             | Node Type              | Functional Role                  | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                                                                            |
|-----------------------|------------------------|--------------------------------|------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| 7am trigger           | Schedule Trigger       | Daily workflow trigger          | -                      | Check google Calendar       |                                                                                                                                                        |
| Check google Calendar | Google Calendar        | Fetch today's calendar events   | 7am trigger            | Count event                 | ## 1.Workflow trigger and Google Calendar<br><br>The workflow is **triggered automatically** each morning at 7am.<br><br>Then the second node will **analyse your Google Calendar** of the day looking for any event or meeting scheduled.<br><br>How to setup:<br>Set up your Google Agenda **API credentials** |
| Count event           | Code                   | Count events and prepare data   | Check google Calendar  | If condition                | ## 2.Count event code and Branch<br><br>The first node will **count the number of event items** scheduled and **return a number**. Even if there is no event, a number (0 in this case) would be returned.<br><br>The the second node will **analyse this number**, if the number is not 0, the True branch ✅ will be activated. If it's O, it will be the False branch ❌ that will be activated. |
| If condition          | If                     | Branch workflow based on events | Count event            | Message code (True branch)<br>Message no meeting today (False branch) | ## 2.Count event code and Branch<br><br>The first node will **count the number of event items** scheduled and **return a number**. Even if there is no event, a number (0 in this case) would be returned.<br><br>The the second node will **analyse this number**, if the number is not 0, the True branch ✅ will be activated. If it's O, it will be the False branch ❌ that will be activated. |
| Message code          | Code                   | Construct detailed event message| If condition (True)    | Send sum up message         | ## 3. Send the telegram message<br><br>In the **True branch** ✅, the code node, will create the message and gather all the informations wanted about the events and meeting. So a full sum um will be sent in one message only.<br>In the Telegram send text node, the **message is sent**.<br><br>If it's the **False branch** ❌, the Telegram node will send the **message about not having meeting that day**.<br><br>How to setup:<br>- Set up a Telegram bot and get the key API.<br>- Connect the API credentials to the node.<br>- For the **True branch** ✅, set up your message in javascript and then copy and paste the json file in the next node.<br>- For the **False branch** ❌, just set up your message. |
| Send sum up message   | Telegram               | Send event summary message      | Message code           | -                          | ## 3. Send the telegram message<br><br>In the **True branch** ✅, the code node, will create the message and gather all the informations wanted about the events and meeting. So a full sum um will be sent in one message only.<br>In the Telegram send text node, the **message is sent**.<br><br>If it's the **False branch** ❌, the Telegram node will send the **message about not having meeting that day**.<br><br>How to setup:<br>- Set up a Telegram bot and get the key API.<br>- Connect the API credentials to the node.<br>- For the **True branch** ✅, set up your message in javascript and then copy and paste the json file in the next node.<br>- For the **False branch** ❌, just set up your message. |
| Message no meeting today | Telegram               | Send no events message          | If condition (False)   | -                          | ## 3. Send the telegram message<br><br>In the **True branch** ✅, the code node, will create the message and gather all the informations wanted about the events and meeting. So a full sum um will be sent in one message only.<br>In the Telegram send text node, the **message is sent**.<br><br>If it's the **False branch** ❌, the Telegram node will send the **message about not having meeting that day**.<br><br>How to setup:<br>- Set up a Telegram bot and get the key API.<br>- Connect the API credentials to the node.<br>- For the **True branch** ✅, set up your message in javascript and then copy and paste the json file in the next node.<br>- For the **False branch** ❌, just set up your message. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node ("7am trigger")**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger daily at 8:00 AM (verify timezone)  
   - No credentials required

2. **Create Google Calendar node ("Check google Calendar")**  
   - Type: Google Calendar  
   - Operation: getAll  
   - Calendar: Enter calendar ID (e.g., "fr.french#holiday@group.v.calendar.google.com") or your target calendar  
   - Options:  
     - orderBy: startTime  
     - returnAll: true  
   - Connect Google Calendar OAuth2 credentials  
   - Connect input from "7am trigger"

3. **Create Code node ("Count event")**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code:  
     ```javascript
     const allItems = $items("Check google Calendar");
     let events = [];
     allItems.forEach(item => {
       const data = item.json;
       if (Array.isArray(data)) {
         events.push(...data.filter(e => e && Object.keys(e).length > 0));
       } else if (data && Object.keys(data).length > 0) {
         events.push(data);
       }
     });
     const eventCount = events.length;
     return [{ json: { eventCount, events } }];
     ```
   - Connect input from "Check google Calendar"

4. **Create If node ("If condition")**  
   - Type: If  
   - Condition:  
     - Expression: `{{$json.eventCount}}`  
     - Operator: Greater than (>)  
     - Value: 0  
   - Connect input from "Count event"

5. **Create Code node ("Message code")**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code:  
     ```javascript
     const rawItems = $items();
     let events = [];
     rawItems.forEach(item => {
       const data = item.json;
       if (Array.isArray(data.events)) {
         events.push(...data.events);
       } else if (data.event) {
         events.push(data.event);
       } else {
         events.push(data);
       }
     });
     let message = `☀️ Good morning! Here's your calendar for today\n\n📅 You have ${events.length} event${events.length > 1 ? 's' : ''} today.\n\n`;
     events.forEach(e => {
       message += `──────────────────────\n`;
       message += `📌 Name: ${e.summary || "N/A"}\n`;
       message += `⏰ Starts: ${e.start?.dateTime || e.start?.date || "N/A"}\n`;
       message += `⏳ Ends: ${e.end?.dateTime || e.end?.date || "N/A"}\n\n`;
       message += `👤 Creator: ${e.creator?.email || "N/A"}\n`;
       message += `👥 Organizer: ${e.organizer?.email || "N/A"}\n\n`;
       message += `📄 Type: ${e.eventType || "N/A"}\n`;
       message += `🔗 Link: ${e.htmlLink || "N/A"}\n\n`;
       message += `📝 Description: ${e.description || "No description"}\n`;
       message += `📍 Location: ${e.location || "N/A"}\n\n`;
     });
     message += "✨ Have a great day!";
     return [{ json: { text: message } }];
     ```
   - Connect input from "If condition" True branch

6. **Create Telegram node ("Send sum up message")**  
   - Type: Telegram  
   - Parameters:  
     - Text: Set to expression `{{$json.text}}` (from previous code node)  
     - Chat ID: Replace `"1234"` with your actual Telegram chat ID  
   - Connect Telegram API credentials (Telegram bot token)  
   - Connect input from "Message code"

7. **Create Telegram node ("Message no meeting today")**  
   - Type: Telegram  
   - Parameters:  
     - Text: Static text "👋 Hi! Your calendar is clear for today 🗓️✨"  
     - Chat ID: Same as above  
   - Connect Telegram API credentials  
   - Connect input from "If condition" False branch

8. **Connect nodes properly**:  
   - "7am trigger" → "Check google Calendar"  
   - "Check google Calendar" → "Count event"  
   - "Count event" → "If condition"  
   - "If condition" True → "Message code" → "Send sum up message"  
   - "If condition" False → "Message no meeting today"

9. **Verify all credentials**:  
   - Google Calendar OAuth2 API credentials authorized for the appropriate calendar scopes  
   - Telegram Bot API credentials with bot token and correct chat ID

10. **Test workflow**:  
    - Run manually or wait for scheduled trigger  
    - Check Telegram for messages  
    - Adjust time zones or calendar IDs as needed

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Required Credentials: Telegram Bot API, Google Account with Calendar API enabled                      | Essential for workflow operation                                                                 |
| Workflow triggers at 7 AM but node is set to 8 AM — verify timezone settings to ensure correct timing | Workflow trigger and schedule trigger node configuration                                         |
| Telegram chat ID ("1234") is a placeholder — replace with actual chat ID                              | Telegram API setup and bot configuration                                                         |
| Message formatting uses Unicode emojis and line separators for readability                            | Enhances user experience in Telegram notifications                                               |
| Google Calendar ID used is for French holidays — replace with user's calendar ID                      | Google Calendar node configuration                                                               |
| Sticky notes in the workflow provide helpful setup instructions and explanations                      | Refer to sticky notes for contextual guidance                                                    |

---

This document fully describes the workflow "Daily Calendar Summary Notifications via Telegram from Google Calendar," enabling advanced users or automation agents to understand, reproduce, and maintain the workflow effectively.