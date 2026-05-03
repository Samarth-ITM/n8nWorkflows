Automated Water Consumption Tracker - Stored in Sheet and Notify in Slack

https://n8nworkflows.xyz/workflows/automated-water-consumption-tracker---stored-in-sheet-and-notify-in-slack-3042


# Automated Water Consumption Tracker - Stored in Sheet and Notify in Slack

### 1. Workflow Overview

This workflow automates water consumption reminders and tracking using n8n, Slack, Google Sheets, and OpenAI. It targets users who want timely, personalized hydration reminders with easy logging and progress tracking. The workflow is logically divided into four main blocks:

- **1.1 Scheduled Triggers and Data Collection**: Periodically triggers reminder logic, fetches daily water intake goals and today's logged water consumption from Google Sheets, and summarizes the data.
- **1.2 Intelligent Reminder Logic**: Combines fetched data, checks if the user has consumed water recently, and delays reminders intelligently to avoid redundancy.
- **1.3 AI Message Generation and Sending**: Uses OpenAI to generate personalized reminder messages and sends them to a Slack channel with interactive buttons for quick water intake logging.
- **1.4 User Interaction and Data Recording**: Handles Slack button interactions, parses user input, records water intake into Google Sheets, and sends confirmation messages with iOS shortcut links for Health app integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Triggers and Data Collection

- **Overview:**  
  This block triggers the workflow on a randomized schedule during waking hours, retrieves the daily water intake goal and today's water logs from Google Sheets, summarizes the total intake, and limits the data to the most recent record for further processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Google Sheets - Get Target  
  - Google Sheets - Get Today Water Log  
  - Limit  
  - Summarize  

- **Node Details:**

  1. **Schedule Trigger**  
     - *Type:* Schedule Trigger  
     - *Role:* Initiates the workflow at randomized minutes every hour between 8 AM and 11 PM.  
     - *Configuration:* Cron expression `0 {{ Math.floor(Math.random() * 11) }} 8-23 * * *` triggers at a random minute (0–10) each hour in the specified range.  
     - *Input/Output:* No input; outputs trigger event to Google Sheets nodes.  
     - *Edge Cases:* Cron misconfiguration could cause missed triggers; time zone differences may affect trigger timing.

  2. **Google Sheets - Get Target**  
     - *Type:* Google Sheets  
     - *Role:* Fetches daily water intake goal from the `setting` sheet.  
     - *Configuration:* Uses spreadsheet ID and sheet named `setting`. Returns all rows (no filter).  
     - *Input:* Triggered by Schedule Trigger.  
     - *Output:* Water intake goal data to merge node.  
     - *Edge Cases:* Authentication failure, spreadsheet access issues, or missing data could cause errors.

  3. **Google Sheets - Get Today Water Log**  
     - *Type:* Google Sheets  
     - *Role:* Retrieves all water intake records logged today from the `log` sheet.  
     - *Configuration:* Filters rows where `date` equals current date (`{{ $now.format('yyyy-MM-dd') }}`).  
     - *Input:* Triggered by Schedule Trigger.  
     - *Output:* Water intake logs to Limit and Summarize nodes.  
     - *Edge Cases:* Empty logs for the day, API rate limits, or auth errors.

  4. **Limit**  
     - *Type:* Limit  
     - *Role:* Selects the most recent water intake record from today's logs.  
     - *Configuration:* Keeps last item only.  
     - *Input:* Water logs from Google Sheets - Get Today Water Log.  
     - *Output:* Most recent water intake record to merge node.  
     - *Edge Cases:* No records today results in empty output.

  5. **Summarize**  
     - *Type:* Summarize  
     - *Role:* Calculates total water intake for today by summing the `value` field.  
     - *Configuration:* Aggregates sum of `value` and includes `date`.  
     - *Input:* Water logs from Google Sheets - Get Today Water Log.  
     - *Output:* Summarized total intake to merge node.  
     - *Edge Cases:* Empty input results in zero sum.

---

#### 2.2 Intelligent Reminder Logic

- **Overview:**  
  This block merges the goal, total intake, and last intake time, checks if the user drank water recently (within 30 minutes), and if so, delays the reminder by a randomized time to avoid spamming.

- **Nodes Involved:**  
  - combine data  
  - Edit Fields-Set progress  
  - If  
  - Wait  

- **Node Details:**

  1. **combine data**  
     - *Type:* Merge  
     - *Role:* Combines three inputs by position: daily goal, most recent intake record, and summarized total intake.  
     - *Configuration:* Combine mode, combining 3 inputs by position.  
     - *Input:* Receives data from Google Sheets - Get Target, Limit, and Summarize nodes.  
     - *Output:* Combined data object for progress calculation and conditional check.  
     - *Edge Cases:* Mismatched input lengths could cause missing data.

  2. **Edit Fields-Set progress**  
     - *Type:* Set  
     - *Role:* Calculates progress percentage and generates a visual progress bar with water drop emojis.  
     - *Configuration:*  
       - `progress_percent` = sum_value / target  
       - `progress_image` = string of 0-10 water drop emojis representing progress  
     - *Input:* Combined data from merge node.  
     - *Output:* Data enriched with progress info to If node.  
     - *Edge Cases:* Division by zero if target is zero or missing.

  3. **If**  
     - *Type:* If  
     - *Role:* Checks if the last water intake time is within the last 30 minutes.  
     - *Configuration:*  
       - Condition: last intake datetime > current time minus 30 minutes  
       - Uses ISO datetime parsing and comparison expressions.  
     - *Input:* Progress data from Set node.  
     - *Output:*  
       - True branch: User drank water recently → triggers Wait node.  
       - False branch: No recent intake → triggers OpenAI node.  
     - *Edge Cases:* Missing or malformed date/time fields could cause expression errors.

  4. **Wait**  
     - *Type:* Wait  
     - *Role:* Delays the reminder by a random number of minutes between 21 and 31 if water was consumed recently.  
     - *Configuration:* Wait time = `{{ Math.floor(Math.random() * 11) + 21 }}` minutes.  
     - *Input:* True branch from If node.  
     - *Output:* After wait, triggers OpenAI node to generate reminder.  
     - *Edge Cases:* Workflow pause could be interrupted; long waits may delay reminders excessively.

---

#### 2.3 AI Message Generation and Sending

- **Overview:**  
  Generates a personalized water drinking reminder message in English using OpenAI GPT-4o-mini, then sends it as a Slack message with interactive buttons for quick logging.

- **Nodes Involved:**  
  - OpenAI  
  - Slack send drink notification  

- **Node Details:**

  1. **OpenAI**  
     - *Type:* OpenAI (LangChain)  
     - *Role:* Generates a friendly, professional, and persuasive reminder message in JSON format.  
     - *Configuration:*  
       - Model: `gpt-4o-mini`  
       - Temperature: 1 (creative responses)  
       - Messages:  
         - System prompt instructs to respond in English with JSON format `{ "message": "..." }`.  
         - Assistant prompt defines tone as gentle, professional, and encouraging with health advice.  
         - User prompt includes last water intake time, current time, daily goal, and progress count.  
     - *Input:* Combined data from If or Wait node.  
     - *Output:* JSON message content to Slack node.  
     - *Edge Cases:* API rate limits, network errors, malformed JSON responses.

  2. **Slack send drink notification**  
     - *Type:* Slack  
     - *Role:* Sends the AI-generated reminder message to a specified Slack channel with buttons for water amounts (100ml to 300ml).  
     - *Configuration:*  
       - Channel ID configured for target Slack channel.  
       - Message type: Block Kit UI with:  
         - Section block showing AI message text.  
         - Section block showing progress bar emoji string.  
         - Actions block with buttons labeled 100, 150, 200, 250, 300 (ml).  
       - OAuth2 authentication with Slack app token.  
     - *Input:* AI message JSON from OpenAI node.  
     - *Output:* Slack message posted, waiting for user interaction.  
     - *Edge Cases:* Slack API errors, permission issues, button interaction failures.

---

#### 2.4 User Interaction and Data Recording

- **Overview:**  
  Handles incoming Slack button interactions, parses the payload, extracts water intake value, logs it to Google Sheets, and sends a confirmation message with an iOS shortcut link for Health app integration.

- **Nodes Involved:**  
  - slack drink webhook  
  - slack_action_payload  
  - slack_action_drink_data  
  - Google Sheets - log water value to sheet  
  - Send to Slack with confirm  

- **Node Details:**

  1. **slack drink webhook**  
     - *Type:* Webhook  
     - *Role:* Receives POST requests from Slack when a water amount button is clicked.  
     - *Configuration:*  
       - HTTP Method: POST  
       - Path: unique webhook path (auto-generated)  
     - *Input:* Slack interaction payload.  
     - *Output:* Raw payload to next node.  
     - *Edge Cases:* Webhook URL exposure, invalid payloads, Slack retry logic.

  2. **slack_action_payload**  
     - *Type:* Set  
     - *Role:* Extracts and parses the JSON payload from the raw webhook body.  
     - *Configuration:*  
       - Mode: Raw  
       - JSON Output: `{{ $json.body.payload }}` (parses nested JSON string)  
     - *Input:* Raw webhook data.  
     - *Output:* Parsed JSON object with action details.  
     - *Edge Cases:* Malformed JSON, missing payload field.

  3. **slack_action_drink_data**  
     - *Type:* Set  
     - *Role:* Extracts relevant fields from Slack action payload and prepares iOS shortcut URL data.  
     - *Configuration:*  
       - Assigns:  
         - `value`: water amount from button (`actions[0].value`)  
         - `message_text`: original Slack message text  
         - `shortcut_url`: fixed prefix `shortcuts://run-shortcut?name=darrell_water&input=`  
         - `shortcut_url_data`: JSON string with water amount and current timestamp  
         - `message_ts`: timestamp of original Slack message thread  
     - *Input:* Parsed Slack action payload.  
     - *Output:* Structured data for logging and confirmation message.  
     - *Edge Cases:* Missing fields, time formatting errors.

  4. **Google Sheets - log water value to sheet**  
     - *Type:* Google Sheets  
     - *Role:* Appends a new row to the `log` sheet recording date, time, and water amount.  
     - *Configuration:*  
       - Operation: Append  
       - Columns:  
         - `date`: current date (`yyyy-MM-dd`)  
         - `time`: current time (`HH:mm:ss`)  
         - `value`: water amount from Slack button  
       - Uses configured Google Sheets credentials and document ID.  
     - *Input:* Water intake data from Set node.  
     - *Output:* Confirmation to Slack node.  
     - *Edge Cases:* API errors, permission issues, data format mismatches.

  5. **Send to Slack with confirm**  
     - *Type:* Slack  
     - *Role:* Sends a confirmation message in the same Slack thread with a button linking to the iOS Health shortcut.  
     - *Configuration:*  
       - Channel ID: same as notification channel  
       - Message type: Block Kit UI with:  
         - Divider and confirmation text  
         - Section with button labeled "iOS Health" linking to the encoded shortcut URL  
       - Replies in thread using original message timestamp (`message_ts`)  
       - OAuth2 authentication.  
     - *Input:* After logging data to Google Sheets.  
     - *Output:* Confirmation message in Slack thread.  
     - *Edge Cases:* Slack API errors, URL encoding issues, thread reply failures.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                              | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                           |
|----------------------------------|---------------------------|----------------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger          | Triggers workflow at randomized times         | None                             | Google Sheets - Get Target, Google Sheets - Get Today Water Log |                                                                                                     |
| Google Sheets - Get Target       | Google Sheets             | Retrieves daily water intake goal              | Schedule Trigger                 | combine data                      |                                                                                                     |
| Google Sheets - Get Today Water Log | Google Sheets             | Retrieves today's water intake logs            | Schedule Trigger                 | Limit, Summarize                  |                                                                                                     |
| Limit                           | Limit                     | Selects most recent water intake record        | Google Sheets - Get Today Water Log | combine data                      |                                                                                                     |
| Summarize                      | Summarize                 | Calculates total water intake for today        | Google Sheets - Get Today Water Log | combine data                      |                                                                                                     |
| combine data                   | Merge                     | Combines goal, last intake, and total intake   | Google Sheets - Get Target, Limit, Summarize | Edit Fields-Set progress           |                                                                                                     |
| Edit Fields-Set progress        | Set                       | Calculates progress percentage and visual bar  | combine data                    | If                               |                                                                                                     |
| If                             | If                        | Checks if water was consumed in last 30 minutes | Edit Fields-Set progress          | Wait (true), OpenAI (false)       | If already drink recently. Delay the notification in 3x minutes randomly                            |
| Wait                           | Wait                      | Delays reminder if water was consumed recently | If (true)                      | OpenAI                           | If the user log water recently. Wait for another 3x minutes                                         |
| OpenAI                         | OpenAI                    | Generates personalized reminder message        | If (false), Wait                | Slack send drink notification     | Send the slack notification with AI wording. Also have the drink water action buttons              |
| Slack send drink notification   | Slack                     | Sends reminder message with water buttons      | OpenAI                         | None                            |                                                                                                     |
| slack drink webhook             | Webhook                   | Receives Slack button click interactions       | None                           | slack_action_payload              |                                                                                                     |
| slack_action_payload            | Set                       | Parses Slack interaction payload                | slack drink webhook             | slack_action_drink_data           |                                                                                                     |
| slack_action_drink_data         | Set                       | Extracts water amount and prepares shortcut URL | slack_action_payload            | Google Sheets - log water value to sheet, Send to Slack with confirm | When User interact the drink button. Record the drink value to sheet and send back the iOS health log water url to start the shortcut |
| Google Sheets - log water value to sheet | Google Sheets             | Logs water intake data to Google Sheets         | slack_action_drink_data         | Send to Slack with confirm        |                                                                                                     |
| Send to Slack with confirm      | Slack                     | Sends confirmation message with iOS shortcut   | Google Sheets - log water value to sheet | None                            |                                                                                                     |
| Sticky Note                    | Sticky Note               | Notes for recent drink data                      | None                           | None                            | ## Grab recent drink data                                                                           |
| Sticky Note1                   | Sticky Note               | Notes about delaying notification                | None                           | None                            | If already drink recently. Delay the notification in 3x minutes randomly                            |
| Sticky Note2                   | Sticky Note               | Notes about Slack notification with AI message  | None                           | None                            | ## Send the slack notification with AI wording. Also have the drink water action buttons           |
| Sticky Note3                   | Sticky Note               | Notes about Slack button interaction and shortcut URL | None                           | None                            | ## When User interact the drink button. Record the drink value to sheet and send back the iOS health log water url to start the shortcut |
| Sticky Note4                   | Sticky Note               | Author contact information                       | None                           | None                            | ## Created by darrell_tw_ \n\nAn engineer now focus on AI and Automation\n\n### contact me with following:\n[X](https://x.com/darrell_tw_)\n[Threads](https://www.threads.net/@darrell_tw_)\n[Instagram](https://www.instagram.com/darrell_tw_/)\n[Website](https://www.darrelltw.com/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set Cron Expression: `0 {{ Math.floor(Math.random() * 11) }} 8-23 * * *`  
   - Purpose: Trigger every hour between 8 AM and 11 PM at a random minute 0-10.

2. **Create Google Sheets - Get Target node**  
   - Type: Google Sheets  
   - Operation: Read rows from sheet `setting`  
   - Document ID: Your Google Sheets ID  
   - Credentials: Google Sheets OAuth2  
   - Connect input from Schedule Trigger.

3. **Create Google Sheets - Get Today Water Log node**  
   - Type: Google Sheets  
   - Operation: Read rows from sheet `log`  
   - Filter: `date` equals current date (`{{ $now.format('yyyy-MM-dd') }}`)  
   - Document ID: Same as above  
   - Credentials: Google Sheets OAuth2  
   - Connect input from Schedule Trigger.

4. **Create Limit node**  
   - Type: Limit  
   - Keep: Last item only  
   - Connect input from Google Sheets - Get Today Water Log.

5. **Create Summarize node**  
   - Type: Summarize  
   - Fields to summarize: sum of `value` and include `date`  
   - Connect input from Google Sheets - Get Today Water Log.

6. **Create combine data node**  
   - Type: Merge  
   - Mode: Combine by position  
   - Number of inputs: 3  
   - Connect inputs from:  
     - Google Sheets - Get Target (input 1)  
     - Limit (input 2)  
     - Summarize (input 3)

7. **Create Edit Fields-Set progress node**  
   - Type: Set  
   - Assignments:  
     - `progress_percent` = `sum_value / target`  
     - `progress_image` = emoji string representing progress (💧 and ⬜)  
   - Connect input from combine data.

8. **Create If node**  
   - Type: If  
   - Condition: Check if last drink time (`date + time`) is after current time minus 30 minutes  
   - Expression example:  
     `DateTime.fromISO($('combine data').item.json.date + "T" + $('combine data').item.json.time).format('yyyy-MM-dd HH:mm:ss') > $now.minus(30, "minutes")`  
   - Connect input from Edit Fields-Set progress.

9. **Create Wait node**  
   - Type: Wait  
   - Wait time: Random minutes between 21 and 31 (`{{ Math.floor(Math.random() * 11) + 21 }}`)  
   - Connect input from If node’s true branch.

10. **Create OpenAI node**  
    - Type: OpenAI (LangChain)  
    - Model: `gpt-4o-mini`  
    - Temperature: 1  
    - Messages:  
      - System prompt: Instruct response in English JSON format `{ "message": "..." }`  
      - Assistant prompt: Friendly Chinese medicine practitioner tone, brief, persuasive message encouraging water intake, mention benefits and risks, end with action prompt  
      - User prompt: Include last drink time, current time, goal, and progress count from combined data  
    - Connect input from If node’s false branch and Wait node’s output.

11. **Create Slack send drink notification node**  
    - Type: Slack  
    - Channel: Your Slack channel ID  
    - Message type: Block Kit UI  
    - Blocks:  
      - Section with AI-generated message text  
      - Section with progress emoji bar  
      - Actions block with buttons labeled 100, 150, 200, 250, 300 (ml) each with corresponding values  
    - Authentication: Slack OAuth2  
    - Connect input from OpenAI node.

12. **Create slack drink webhook node**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Path: Unique webhook path (auto-generated or custom)  
    - Purpose: Receive Slack button click events.

13. **Create slack_action_payload node**  
    - Type: Set  
    - Mode: Raw  
    - JSON Output: `{{ $json.body.payload }}` (parse Slack’s nested JSON payload)  
    - Connect input from slack drink webhook.

14. **Create slack_action_drink_data node**  
    - Type: Set  
    - Assignments:  
      - `value`: `{{ $json.actions[0].value }}` (water amount)  
      - `message_text`: `{{ $json.message.text }}`  
      - `shortcut_url`: fixed string `shortcuts://run-shortcut?name=darrell_water&input=`  
      - `shortcut_url_data`: JSON string with water amount and current timestamp, e.g. `{"value":100,"time":"2025-03-04T16:10:15"}`  
      - `message_ts`: `{{ $json.container.message_ts }}` (Slack thread timestamp)  
    - Connect input from slack_action_payload.

15. **Create Google Sheets - log water value to sheet node**  
    - Type: Google Sheets  
    - Operation: Append row to sheet `log`  
    - Columns:  
      - `date`: current date (`{{ $now.format('yyyy-MM-dd') }}`)  
      - `time`: current time (`{{ $now.format('HH:mm:ss') }}`)  
      - `value`: water amount from slack_action_drink_data  
    - Credentials: Google Sheets OAuth2  
    - Connect input from slack_action_drink_data.

16. **Create Send to Slack with confirm node**  
    - Type: Slack  
    - Channel: Same Slack channel ID  
    - Message type: Block Kit UI  
    - Blocks:  
      - Divider  
      - Section with confirmation text "Already log the water"  
      - Section with button labeled "iOS Health" linking to encoded shortcut URL (`shortcut_url` + URL-encoded `shortcut_url_data`)  
    - Reply in thread using `message_ts` from slack_action_drink_data  
    - Authentication: Slack OAuth2  
    - Connect input from Google Sheets - log water value to sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Google Sheet Template for setup: includes `log` and `setting` sheets with required columns.                                                  | https://docs.google.com/spreadsheets/d/1cMf03l3Ae3vo-4w8yeaUvSh9dB7CLM7CFUb2y4BmqwY/edit?usp=sharing     |
| iOS Shortcut named `darrell_water` to log water intake into Health app, triggered via URL scheme with JSON input.                          | Shortcut URL example: `shortcuts://run-shortcut?name=darrell_water&input={"value":100,"time":"2025-03-04T16:10:15"}` |
| Video demo of the workflow and integration on YouTube.                                                                                      | https://www.youtube.com/watch?v=h2_QO5gxdts                                                           |
| Author contact and social media: darrell_tw_, AI and automation engineer.                                                                    | [X](https://x.com/darrell_tw_), [Threads](https://www.threads.net/@darrell_tw_), [Instagram](https://www.instagram.com/darrell_tw_/), [Website](https://www.darrelltw.com/) |

---

This documentation provides a detailed, structured understanding of the Automated Water Consumption Tracker workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.