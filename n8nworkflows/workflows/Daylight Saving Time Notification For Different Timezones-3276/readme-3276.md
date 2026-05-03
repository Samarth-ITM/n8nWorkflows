Daylight Saving Time Notification For Different Timezones

https://n8nworkflows.xyz/workflows/daylight-saving-time-notification-for-different-timezones-3276


# Daylight Saving Time Notification For Different Timezones

### 1. Workflow Overview

This workflow is designed to notify users about upcoming Daylight Saving Time (DST) changes in multiple specified timezones. It is particularly useful for professionals managing meetings across different regions where DST change dates may vary, such as project managers, CFOs, CTOs, or CEOs.

The workflow runs daily, checks if any of the monitored timezones will experience a DST change the next day (or a configurable number of days ahead), and sends notifications via Slack and optionally email to alert the user to adjust meeting times accordingly.

**Logical Blocks:**

- **1.1 Input Reception and Scheduling:** Triggering the workflow daily and providing the list of timezones to monitor.
- **1.2 Date and Timezone Calculations:** Calculating current and next-day datetime for each timezone.
- **1.3 DST Change Detection:** Determining if a DST change occurs between today and tomorrow in each timezone.
- **1.4 Notification Dispatch:** Sending Slack and email notifications if a DST change is detected.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

**Overview:**  
This block initiates the workflow either manually or on a schedule and provides the list of timezones to monitor for DST changes.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Timezones List  
- Sticky Note1 (Documentation)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow at a defined interval (daily).  
  - *Configuration:* Default interval (daily) with no specific time set, meaning it triggers once every day.  
  - *Input/Output:* No input; outputs to "Timezones List".  
  - *Potential Failures:* None typical; possible scheduling misconfiguration.  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution for testing purposes.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Input/Output:* No input; outputs to "Timezones List".  
  - *Potential Failures:* None typical.  

- **Timezones List**  
  - *Type:* Code (JavaScript)  
  - *Role:* Provides the list of timezones to monitor.  
  - *Configuration:* Returns an array of objects, each containing a `timezone` property with IANA timezone strings (e.g., "America/New_York", "Europe/Warsaw").  
  - *Key Expressions:* Hardcoded timezone list in JavaScript.  
  - *Input/Output:* Receives trigger from Schedule or Manual Trigger; outputs to "Calculate Zone Date and Time".  
  - *Potential Failures:* Incorrect timezone strings could cause errors downstream.  

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Provides user instructions and overview.  
  - *Content:* Explains workflow logic and setup reminders (add timezones, set Slack channel).  
  - *Input/Output:* None; purely informational.  

---

#### 1.2 Date and Timezone Calculations

**Overview:**  
Calculates the current datetime and the datetime for tomorrow (or configurable offset) in each timezone from the list.

**Nodes Involved:**  
- Calculate Zone Date and Time  
- Calculate Tomorrow's Date

**Node Details:**

- **Calculate Zone Date and Time**  
  - *Type:* Set  
  - *Role:* Assigns the current datetime in the specified timezone to a new field `datetime_zone`.  
  - *Configuration:* Uses expression `{{$now.setZone($json.timezone)}}` to get current datetime in each timezone.  
  - *Input/Output:* Input from "Timezones List"; outputs to "Calculate Tomorrow's Date".  
  - *Potential Failures:* If timezone is invalid, expression may fail or return incorrect datetime.  

- **Calculate Tomorrow's Date**  
  - *Type:* DateTime  
  - *Role:* Calculates the datetime for tomorrow (or offset days) based on `datetime_zone`.  
  - *Configuration:* Adds 1 day to `datetime_zone` and stores result in `datetime_zone_tomorrow`.  
  - *Input/Output:* Input from "Calculate Zone Date and Time"; outputs to "Check If Daylight Saving Time".  
  - *Potential Failures:* Misconfiguration of duration or magnitude could cause incorrect date calculation.  

---

#### 1.3 DST Change Detection

**Overview:**  
Determines if there is a DST status change between today and tomorrow for each timezone.

**Nodes Involved:**  
- Check If Daylight Saving Time  
- Check If Change Tomorrow

**Node Details:**

- **Check If Daylight Saving Time**  
  - *Type:* Set  
  - *Role:* Adds two boolean fields:  
    - `datetime_zone_dst` indicating if current datetime is in DST  
    - `datetime_zone_tomorrow_dst` indicating if tomorrow's datetime is in DST  
  - *Configuration:* Uses expressions:  
    - `{{$json.datetime_zone.toDateTime().setZone($json.timezone).isInDST}}`  
    - `{{$json.datetime_zone_tomorrow.toDateTime().setZone($json.timezone).isInDST}}`  
  - *Input/Output:* Input from "Calculate Tomorrow's Date"; outputs to "Check If Change Tomorrow".  
  - *Potential Failures:* Expression errors if datetime or timezone is invalid.  

- **Check If Change Tomorrow**  
  - *Type:* If  
  - *Role:* Compares `datetime_zone_dst` and `datetime_zone_tomorrow_dst` to detect DST change.  
  - *Configuration:* Condition: `datetime_zone_dst != datetime_zone_tomorrow_dst` (boolean not equals).  
  - *Input/Output:* Input from "Check If Daylight Saving Time"; outputs true branch to notification nodes, false branch ends workflow for that timezone.  
  - *Potential Failures:* Logical errors if fields are missing or incorrectly typed.  

---

#### 1.4 Notification Dispatch

**Overview:**  
Sends notifications via Slack and email if a DST change is detected for a timezone.

**Nodes Involved:**  
- Send Notification On Upcoming Change (Slack)  
- Send Email On Upcoming Change

**Node Details:**

- **Send Notification On Upcoming Change**  
  - *Type:* Slack  
  - *Role:* Sends a Slack message notifying about the DST change.  
  - *Configuration:*  
    - Message text: `"Tomorrow is Daylight Saving Time change in zone {{ $json.timezone }} - remember to adjust meeting times!"`  
    - Channel: Configurable via node parameter (dropdown or list)  
    - Authentication: OAuth2 with Slack credentials  
  - *Input/Output:* Input from "Check If Change Tomorrow" (true branch).  
  - *Potential Failures:* Slack API auth errors, invalid channel ID, rate limits.  

- **Send Email On Upcoming Change**  
  - *Type:* Email Send  
  - *Role:* Sends an email notification about the DST change.  
  - *Configuration:*  
    - Subject: `"DST change tomorrow in {{ $json.timezone }}"`  
    - Body text: `"Tomorrow is Daylight Saving Time change in zone {{ $json.timezone }} - remember to adjust meeting times!"`  
    - Format: Plain text  
    - SMTP credentials required  
  - *Input/Output:* Input from "Check If Change Tomorrow" (true branch).  
  - *Potential Failures:* SMTP auth errors, invalid recipient addresses, network issues.  

---

### 3. Summary Table

| Node Name                       | Node Type           | Functional Role                         | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                   |
|--------------------------------|---------------------|---------------------------------------|-------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger    | Initiates workflow daily               | None                          | Timezones List                       |                                                                                              |
| When clicking ‘Test workflow’  | Manual Trigger      | Manual execution for testing           | None                          | Timezones List                       |                                                                                              |
| Timezones List                | Code                | Provides list of timezones             | Schedule Trigger, Manual Trigger | Calculate Zone Date and Time         |                                                                                              |
| Calculate Zone Date and Time  | Set                 | Calculates current datetime per zone  | Timezones List                | Calculate Tomorrow's Date            |                                                                                              |
| Calculate Tomorrow's Date     | DateTime            | Calculates tomorrow's datetime per zone | Calculate Zone Date and Time  | Check If Daylight Saving Time        |                                                                                              |
| Check If Daylight Saving Time | Set                 | Determines DST status today and tomorrow | Calculate Tomorrow's Date     | Check If Change Tomorrow             |                                                                                              |
| Check If Change Tomorrow      | If                  | Checks if DST status changes tomorrow | Check If Daylight Saving Time | Send Notification On Upcoming Change, Send Email On Upcoming Change |                                                                                              |
| Send Notification On Upcoming Change | Slack           | Sends Slack notification on DST change | Check If Change Tomorrow (true) | None                               |                                                                                              |
| Send Email On Upcoming Change | Email Send           | Sends email notification on DST change | Check If Change Tomorrow (true) | None                               |                                                                                              |
| Sticky Note1                 | Sticky Note          | Workflow overview and setup reminders | None                          | None                               | ## How it works\n- check list of timezones\n- check if any timezone switches from/to Daylight Saving Time\n- notify on Slack\n\n## Remember to set up\n- Add timezones to "Timezones List"\n- Slack notification channel |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node: set it to run daily (default interval).  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" for manual runs.

2. **Create Timezones List Node:**  
   - Add a **Code** node named "Timezones List".  
   - Paste the following JavaScript code to return monitored timezones:  
     ```javascript
     return [
       { timezone: "America/New_York" },
       { timezone: "Europe/Warsaw" },
     ];
     ```  
   - Connect both triggers ("Schedule Trigger" and "When clicking ‘Test workflow’") to this node.

3. **Calculate Current Datetime per Timezone:**  
   - Add a **Set** node named "Calculate Zone Date and Time".  
   - Configure to add a new field `datetime_zone` (string) with expression:  
     `{{$now.setZone($json.timezone)}}`  
   - Connect "Timezones List" output to this node.

4. **Calculate Tomorrow's Datetime per Timezone:**  
   - Add a **DateTime** node named "Calculate Tomorrow's Date".  
   - Set operation to "Add to Date".  
   - Add 1 day to the field `datetime_zone`.  
   - Output field name: `datetime_zone_tomorrow`.  
   - Connect "Calculate Zone Date and Time" output to this node.

5. **Determine DST Status Today and Tomorrow:**  
   - Add a **Set** node named "Check If Daylight Saving Time".  
   - Add two fields:  
     - `datetime_zone_dst` (string) with expression:  
       `{{$json.datetime_zone.toDateTime().setZone($json.timezone).isInDST}}`  
     - `datetime_zone_tomorrow_dst` (string) with expression:  
       `{{$json.datetime_zone_tomorrow.toDateTime().setZone($json.timezone).isInDST}}`  
   - Connect "Calculate Tomorrow's Date" output to this node.

6. **Check for DST Change:**  
   - Add an **If** node named "Check If Change Tomorrow".  
   - Set condition:  
     - Left value: `{{$json.datetime_zone_dst}}`  
     - Operator: Not Equals  
     - Right value: `{{$json.datetime_zone_tomorrow_dst}}`  
   - Connect "Check If Daylight Saving Time" output to this node.

7. **Send Slack Notification:**  
   - Add a **Slack** node named "Send Notification On Upcoming Change".  
   - Configure message text:  
     `Tomorrow is Daylight Saving Time change in zone {{ $json.timezone }} - remember to adjust meeting times!`  
   - Select Slack channel or provide channel ID where notifications should be sent.  
   - Set authentication to OAuth2 with Slack credentials.  
   - Connect the **true** output of "Check If Change Tomorrow" to this node.

8. **Send Email Notification (Optional):**  
   - Add an **Email Send** node named "Send Email On Upcoming Change".  
   - Configure subject:  
     `DST change tomorrow in {{ $json.timezone }}`  
   - Configure body text:  
     `Tomorrow is Daylight Saving Time change in zone {{ $json.timezone }} - remember to adjust meeting times!`  
   - Set email format to plain text.  
   - Configure SMTP credentials.  
   - Connect the **true** output of "Check If Change Tomorrow" to this node.

9. **Add Documentation:**  
   - Add a **Sticky Note** node named "Sticky Note1" with content describing workflow logic and setup instructions.

10. **Final Connections:**  
    - Connect "Schedule Trigger" and "When clicking ‘Test workflow’" to "Timezones List".  
    - Connect nodes as described above in the logical order.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow requires valid IANA timezone names to function correctly.                                      | Timezone list in "Timezones List" node must be accurate.                                                  |
| Slack OAuth2 credentials must be configured with appropriate scopes to send messages to the desired channels. | Slack API documentation: https://api.slack.com/authentication/oauth-v2                                    |
| SMTP credentials must be valid and tested to ensure email notifications are delivered.                        | Use your SMTP provider’s recommended settings.                                                           |
| Adjust the "Calculate Tomorrow's Date" node's duration to notify earlier than 1 day if needed.               | For example, set duration to 2 days to get notifications 2 days before DST change.                        |
| The workflow can be extended or modified to support other notification channels by replacing or adding nodes. |                                                                                                          |
| Reminder: DST change dates vary by timezone and can differ by country and region.                            | Always verify timezone data if unexpected results occur.                                                  |