Dynamically Send Out-of-Office / Vacation Message using Gmail & Google Calendar

https://n8nworkflows.xyz/workflows/dynamically-send-out-of-office---vacation-message-using-gmail---google-calendar-6336


# Dynamically Send Out-of-Office / Vacation Message using Gmail & Google Calendar

### 1. Workflow Overview

This workflow automates the process of sending personalized out-of-office (vacation) email replies based on real-time Google Calendar events. It is designed for users who want dynamic, calendar-driven out-of-office messaging through Gmail, reflecting their actual availability rather than a static status.

**Use Cases:**
- Freelancers, consultants, and remote workers without fixed schedules.
- Professionals who want automated email responses tied to their real calendar events.
- Businesses aiming to maintain professional communication even when staff are away.

**Logical Blocks:**

- **1.1 Input Reception:** Monitor incoming emails in Gmail to trigger the workflow.
- **1.2 Calendar Check #1:** Check if there are any remaining events today, indicating the user is still working.
- **1.3 Conditional Logic:** Decide if the user is off work today or still busy.
- **1.4 Calendar Check #2:** If no events remain today, find the next calendar event within the next two weeks to determine the return date.
- **1.5 Date Formatting:** Format the return date into a readable string.
- **1.6 Email Sending:** Send the out-of-office reply with the personalized return date.
- **1.7 Documentation and Setup Notes:** Sticky notes providing setup instructions, explanations, and customization tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block continuously monitors incoming Gmail messages and triggers the workflow for each new email.

**Nodes Involved:**  
- Receive an Email in Gmail

**Node Details:**

- **Receive an Email in Gmail**  
  - *Type:* Gmail Trigger  
  - *Role:* Initiates the workflow on receipt of new emails.  
  - *Configuration:* Polls Gmail every minute without filters by default (can be customized to filter by labels or senders).  
  - *Expressions/Variables:* Uses the sender email address from incoming email for message reply.  
  - *Input:* None (trigger node).  
  - *Output:* Passes email data downstream for calendar checking.  
  - *Edge Cases:* Gmail API rate limits, authentication errors, or delayed email polling could affect trigger reliability.  
  - *Credentials:* Requires Gmail OAuth2 credentials.

---

#### 1.2 Calendar Check #1: Check Calendar for Upcoming Work Events

**Overview:**  
Checks if there are any upcoming events remaining today on the user's selected work calendar.

**Nodes Involved:**  
- Check Calendar for Upcoming Work Events

**Node Details:**

- **Check Calendar for Upcoming Work Events**  
  - *Type:* Google Calendar  
  - *Role:* Queries the user's selected work calendar for any events later today.  
  - *Configuration:*  
    - Limits results to 1 event.  
    - Orders events by start time ascending.  
    - Uses `timeMax` set to the end of the current day (`$now.plus(1, 'days').format('yyyy-MM-dd')`).  
    - Selects a specific calendar identified by ID (work calendar).  
  - *Expressions:* Uses n8n's date/time formatting functions to set the time range.  
  - *Input:* Triggered by the Gmail trigger node.  
  - *Output:* Passes calendar event data to the conditional node.  
  - *Edge Cases:* No events today, calendar access permission errors, API quota limits.  
  - *Credentials:* Google Calendar OAuth2 required.

---

#### 1.3 Conditional Logic: Any Upcoming Events Today?

**Overview:**  
Determines if there is at least one event remaining today to decide whether the user is still working or off.

**Nodes Involved:**  
- Any Upcoming Events Today?

**Node Details:**

- **Any Upcoming Events Today?**  
  - *Type:* If node (conditional logic)  
  - *Role:* Checks if the calendar event summary exists (meaning an event is scheduled today).  
  - *Configuration:* Uses the condition “exists” on `$json.summary` from calendar data to detect events.  
  - *Input:* Receives event data from calendar node.  
  - *Output:*  
    - **True:** Ends workflow (no further action because user is working).  
    - **False:** Proceeds to next calendar check for return date.  
  - *Edge Cases:* Missing or malformed calendar data could cause false negatives.  
  - *Version:* Uses version 2.2 of the If node for enhanced conditions.

---

#### 1.4 Calendar Check #2: Find Return-to-Office Date

**Overview:**  
If no events remain today, this block finds the next event within the next 14 days to estimate the return-to-office date.

**Nodes Involved:**  
- Find Return-to-Office Date

**Node Details:**

- **Find Return-to-Office Date**  
  - *Type:* Google Calendar  
  - *Role:* Queries the same work calendar for the next event within the next two weeks.  
  - *Configuration:*  
    - Limits results to 1 event.  
    - Orders by start time ascending.  
    - Sets `timeMax` to 2 weeks from now.  
    - Uses the same work calendar ID as before.  
  - *Input:* Activated only if no events remain today.  
  - *Output:* Passes the next event's start datetime downstream for formatting.  
  - *Edge Cases:* No events in the next 14 days (may cause missing return date), API errors.  
  - *Credentials:* Shares Google Calendar OAuth2 credentials with other calendar nodes.

---

#### 1.5 Date Formatting: Nicely Format Return Date

**Overview:**  
Transforms the raw ISO date-time returned from the calendar into a human-readable format suitable for email.

**Nodes Involved:**  
- Nicely Format Return Date

**Node Details:**

- **Nicely Format Return Date**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the calendar event start date and formats it as "Weekday, Month day, Year" (e.g., "Thursday, July 24, 2025").  
  - *Configuration:*  
    - Extracts `dateTime` and `timeZone` from calendar event JSON.  
    - Uses JavaScript's `Intl.DateTimeFormat` with locale default and event timezone.  
  - *Input:* Receives next event data from previous calendar node.  
  - *Output:* Returns JSON with `returnDate` string for email template.  
  - *Edge Cases:* Null or missing date/time fields could cause code errors; timezone inconsistencies.  
  - *Version:* Uses Code node v2 syntax.

---

#### 1.6 Email Sending: Send Out of Office Message

**Overview:**  
Sends a customized out-of-office reply email to the original sender, including the formatted return date.

**Nodes Involved:**  
- Send Out of Office Message

**Node Details:**

- **Send Out of Office Message**  
  - *Type:* Gmail node (Send Email)  
  - *Role:* Sends the personalized vacation responder email.  
  - *Configuration:*  
    - Sends to the original email's `From` address.  
    - Message body includes a greeting, acknowledgment, the return date, and a signature placeholder `[User Name]`.  
    - Subject line fixed as "Out of Office / Vacation Responder".  
    - Email type is plain text.  
    - Optionally appends n8n attribution.  
  - *Input:* Receives formatted return date and original email data.  
  - *Output:* Email dispatched; no further nodes.  
  - *Edge Cases:* Gmail API limits, authentication failures, invalid recipient addresses.  
  - *Credentials:* Gmail OAuth2 credentials required.

---

#### 1.7 Documentation and Setup Notes

**Overview:**  
Sticky notes provide explanations, instructions, and customization tips to assist users in understanding and adapting the workflow.

**Nodes Involved:**  
- Sticky Note (General overview)  
- Sticky Note1 (User setup instructions)  
- Sticky Note2 (Gmail trigger description)  
- Sticky Note3 (First calendar check explanation)  
- Sticky Note4 (Second calendar check explanation)  
- Sticky Note5 (Function node description)  
- Sticky Note6 (Gmail send node details)  
- Sticky Note7 (Customization options)

**Node Details:**  
- *Type:* Sticky Note  
- *Role:* Documentation only, no execution impact.  
- *Content:* Covers workflow purpose, prerequisites, user instructions, and customization.  
- *Position:* Spread around the workflow for contextual help.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                            |
|--------------------------------|---------------------|-----------------------------------------|---------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Receive an Email in Gmail       | Gmail Trigger       | Triggers workflow on incoming email     | None                            | Check Calendar for Upcoming Work Events | ## Gmail Trigger - Monitors incoming emails every minute - Can be filtered (e.g., labels or VIP senders)               |
| Check Calendar for Upcoming Work Events | Google Calendar    | Checks for remaining events today       | Receive an Email in Gmail        | Any Upcoming Events Today?       | ## Calendar Check #1 - Inspects if any events remain today                                                             |
| Any Upcoming Events Today?      | If                  | Checks if any events exist today         | Check Calendar for Upcoming Work Events | (True) None / (False) Find Return-to-Office Date |                                                                                                                        |
| Find Return-to-Office Date      | Google Calendar     | Finds next event within 14 days          | Any Upcoming Events Today? (False) | Nicely Format Return Date        | ## Calendar Check #2 - If no remaining events, scans next 14 days for the next event                                   |
| Nicely Format Return Date       | Code                | Formats return date for email             | Find Return-to-Office Date       | Send Out of Office Message       | ## Function Node - Formats the return date as Weekday, Month D, YYYY (e.g., “Thursday, July 24, 2025”)                   |
| Send Out of Office Message      | Gmail (Send Email)  | Sends personalized out-of-office reply   | Nicely Format Return Date        | None                           | ## Gmail Send - Sends a customized out‑of‑office email, using the formatted date - Optionally includes n8n attribution |
| Sticky Note                    | Sticky Note         | General workflow explanation              | None                            | None                           | ## What it Does ... Ideal for freelancers, consultants, or remote workers...                                            |
| Sticky Note1                   | Sticky Note         | User setup instructions                   | None                            | None                           | ## User Setup Instructions ...                                                                                            |
| Sticky Note2                   | Sticky Note         | Gmail trigger explanation                  | None                            | None                           | ## Gmail Trigger ...                                                                                                     |
| Sticky Note3                   | Sticky Note         | Calendar Check #1 explanation              | None                            | None                           | ## Calendar Check #1 ...                                                                                                |
| Sticky Note4                   | Sticky Note         | Calendar Check #2 explanation              | None                            | None                           | ## Calendar Check #2 ...                                                                                                |
| Sticky Note5                   | Sticky Note         | Function node explanation                   | None                            | None                           | ## Function Node ...                                                                                                    |
| Sticky Note6                   | Sticky Note         | Gmail send node explanation                 | None                            | None                           | ## Gmail Send ...                                                                                                       |
| Sticky Note7                   | Sticky Note         | Customization tips                          | None                            | None                           | ## Customization Options ...                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Add a *Gmail Trigger* node named "Receive an Email in Gmail".  
   - Configure credentials with your Gmail OAuth2 account.  
   - Set polling to every minute. Optionally add filters (labels, senders).  

2. **Create First Google Calendar Node**  
   - Add a *Google Calendar* node named "Check Calendar for Upcoming Work Events".  
   - Authenticate with Google Calendar OAuth2.  
   - Select your work calendar by calendar ID or from the list.  
   - Set operation to "Get All".  
   - Limit results to 1, order by start time ascending.  
   - Set `timeMax` parameter to the end of the current day using expression:  
     `={{ $now.plus(1, 'days').format('yyyy-MM-dd') }}`  
   - Connect output of Gmail trigger to this node.  

3. **Add Conditional If Node**  
   - Add an *If* node named "Any Upcoming Events Today?".  
   - Configure condition: Check if field `$json.summary` exists (string exists operator).  
   - Connect output of calendar node to the If node.  

4. **Create Second Google Calendar Node**  
   - Add a *Google Calendar* node named "Find Return-to-Office Date".  
   - Use same Google Calendar credentials and calendar ID as before.  
   - Operation: "Get All" events.  
   - Limit results to 1, order ascending by start time.  
   - Set `timeMax` to two weeks from now with expression:  
     `={{ $now.plus({ week: 2 }) }}`  
   - Connect the **False** output of the If node to this node.  

5. **Create Code Node for Date Formatting**  
   - Add a *Code* node named "Nicely Format Return Date".  
   - Use JavaScript to parse `start.dateTime` and `start.timeZone` from input JSON.  
   - Format the date as:  
     ```js
     const dateTime = $input.first().json.start.dateTime;
     const timeZone = $input.first().json.start.timeZone;

     const date = new Date(dateTime);

     const returnDate = new Intl.DateTimeFormat(undefined, {
       timeZone,
       weekday: 'long',
       year: 'numeric',
       month: 'long',
       day: 'numeric'
     }).format(date);

     return [{ json: { returnDate } }];
     ```  
   - Connect output of "Find Return-to-Office Date" node to this node.  

6. **Create Gmail Send Node**  
   - Add a *Gmail* node named "Send Out of Office Message".  
   - Authenticate with Gmail OAuth2 credentials.  
   - Set the recipient as the sender of the incoming email:  
     `={{ $('Receive an Email in Gmail').item.json.From }}`  
   - Compose the message:  
     ```
     Hello, 

     Thanks for your message!

     I am currently out of the office, and I will be returning on {{ $json.returnDate }}.

     Best, 
     [User Name]
     ```  
   - Set subject to "Out of Office / Vacation Responder".  
   - Set email type to plain text.  
   - Enable or disable attribution as desired.  
   - Connect output of "Nicely Format Return Date" to this node.  

7. **Connect Nodes Sequentially:**  
   - Gmail Trigger → Check Calendar for Upcoming Work Events → Any Upcoming Events Today?  
   - If True branch: no output (workflow ends).  
   - If False branch: → Find Return-to-Office Date → Nicely Format Return Date → Send Out of Office Message  

8. **Add Sticky Notes (Optional but Recommended):**  
   - Add sticky notes near respective nodes to document purpose, usage, and setup instructions.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Automatically checks your Google Calendar to determine if you're officially off work for the rest of today. If so, it auto-sends a personalized out‑of‑office reply via Gmail, telling senders when you’ll be back.                     | General workflow description             |
| To use this template, you'll need Gmail and Google Calendar credentials and a dedicated work calendar selected in the Calendar nodes.                                                                                             | Prerequisites                           |
| Ideal for freelancers, consultants, or remote workers who don’t follow a strict 9–5, yet want automated responses aligned with their actual availability, not a static setting.                                                     | Use case description                     |
| User Setup Instructions: Connect Gmail and Google Calendar accounts, set up calendar selection, adjust date formatting if needed, and customize email message templates.                                                           | Setup guidance                          |
| Gmail Trigger polls every minute and can be filtered by labels or senders for targeted auto-replies.                                                                                                                              | Gmail trigger explanation               |
| Calendar Check #1 inspects if any events remain today to decide if the user is still working.                                                                                                                                      | Calendar check explanation              |
| Calendar Check #2 finds the next event within 14 days to determine return date if no events remain today.                                                                                                                         | Calendar check explanation              |
| Function Node formats the return date as “Weekday, Month D, YYYY” for a clear, professional message.                                                                                                                               | Function node explanation               |
| Gmail Send node sends the customized email with the formatted return date; attribution to n8n can be toggled.                                                                                                                     | Gmail send node explanation             |
| Customization options include editing the email message, adjusting calendar lookahead period, and adding Gmail filters to restrict auto-replies to certain senders or labels.                                                        | Customization tips                      |
| Workflow respects Google APIs rate limits and OAuth2 authentication requirements for secure and reliable operation.                                                                                                               | Security and API notes                  |

---

This documentation fully describes the workflow for dynamically sending out-of-office messages based on Google Calendar events via Gmail, including how to build, understand, and customize it.